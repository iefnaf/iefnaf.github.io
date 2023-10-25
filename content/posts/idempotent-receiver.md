+++
title = 'Idempotent Receiver'
date = 2023-10-25T14:35:28+08:00
+++

辨别来自客户端的请求之前是否处理过，从而当客户端重试时能够丢弃重复的请求。

> Identify requests from clients uniquely so they can ignore duplicate requests when client retries.

## 问题

客户端给服务器发送了一个请求，但是没有得到响应。造成客户端没有收到响应的原因可能有两个：

1. 网络问题导致丢包。可能是客户端发送的请求在网络中丢失，或者是服务器返回的响应在网络中丢失。
2. 服务器发生crash。可能服务器在处理客户端请求之前发生crash，也可能在处理请求之后，返回响应之前发生crash。

在客户端的视角中，它无法分辨到底是1还是2导致它没有得到响应。为了保证请求最终能够被处理，客户端选择给服务器重新发送请求。

如果在处理第一个请求的时候发生了第二种情况，服务器在处理请求之后，返回响应之前发生了crash，那么服务器在收到客户端发送的第二个请求就是一个已经处理过的重复的请求。

## 解决方法

给每个客户端一个唯一的编号，来在系统中唯一地标识每个客户端。在发送第一个请求之前，客户端向服务器发起一次`注册流程`：

```java
class ConsistentCoreClient {
  private void registerWithLeader() {
    RequestOrResponse request = new RequestOrResponse(RequestId.RegisterClientRequest.getId()),correlationId.incrementAndGet());
    // blockingSend will attempt to create a new connection if there is a network error.
    RequestOrResponse response = blockingSend(request);
    RegisterClientResponse registerClientResponse = JsonSerDes.deserialize(response.getMessageBodyJson(), RegisterClientResponse.class);
    this.clientId = registerClientResponse.getClientId();
  }
}
```

如果服务器收到客户端发送来的注册请求，那么它就给这个客户端返回一个唯一的ID。如果这个服务器是一致性的（Consistent Core），它就可以把WAL日志的Index作为ID返回给客户端。

```java
class ReplicatedKVStore {
  private Map<Long, Session> clientSessions = new ConcurrentHashMap<>();
  
  private RegisterClientResponse registerClient(WALEntry walEntry) {
    Long clientId = walEntry.getEntryIndex();
    // clientId to store client response.
    clientSessions.put(clientId, new Session(clock.nanoTime()));
    
    return new RegisterClientResponse(client);
  }
}
```

服务器为注册的客户端创建一个session来保存请求的响应。同时，session中还保存了创建时间，创建时间可以用来丢弃不活跃的session。

```java
public class Session {
  long lastAccessTimestamp;
  Queue<Response> clientResponses = new ArrayDeque<>();
  
  public Session(long lastAccessTimestamp) {
    this.lastAccessTimestamp = lastAccessTimestamp;
  }
  
  public long getLastAccessTimestamp() {
    return lastAccessTimestamp;
  }
  
  public Optional<Response> getResponse(int requestNumber) {
    return clientResponses.stream().filter(r -> requestNumber == r.getRequestNumber()).findFirst();
  }
  
  private static final int MAX_SAVED_RESPONSES = 5;
  
  public void addResponse(Response response) {
    if (clientResponses.size() == MAX_SAVED_RESPONSES) {
      clientResponses.remove(); // remove the oldest request
    }
    clientResponses.add(response);
  }
  
  public void refresh(long nanoTime) {
    this.lastAccessTimestamp = nanoTime;
  }
}
```

对于一致性的系统（Consistent Core）来说，客户端注册请求作为共识协议的一部分来进行复制。所以即使Leader挂掉，客户端住客服务依然可用。同时，除Leader外的其他服务器也存储了返回给客户端的响应。



**幂等和非幂等请求**：需要注意到的是，有一些请求天然就是幂等的。比如，在一个KV存储中设置一个key和value，它是天然幂等的。即使相同的key和value被设置多次，也不会造成什么问题。

在另一方面，创建一个`租约`（lease）不是幂等的。如果一个租约已经被创建了，那么重试创建租约将会失败。这是一个问题。考虑如下的场景：一个客户端给服务器发送一个创建租约的请求，服务器创建成功，但是之后发生了crash，或者在发送响应之前与客户端的连接断开了。客户端之后发起重连，重新尝试创建租约。但是因为服务器关于客户端已经有了一个租约，所以它返回一个error。那么客户端就会认为它没有租约。这显然不是我们想要看到的。

当有了幂等接收者之后，客户端使用相同的请求ID给服务器发送租约请求。因为之前的那个响应已经在服务器中保存了，所以服务器将返回这个保存的响应。通过这种方式，假如客户端能够在连接断开之前成功地完成租约的创建，那么它之后重试这个请求，就会得到这个响应。



对于收到的所有非幂等请求，在成功执行完毕之后，服务端在客户端的session中保存响应。

```java
class ReplicatedKVStore {
  private Response applyRegisterLeaseCommand(WALEntry walEntry, RegisterLeaseCommand command) {
    logger.info("Create lease with id " + command.getName() + "with timeout " + command.getTimeout() + " on server " + getReplicatedLog().getServerId());
    try {
      leaseTracker.addLease(command.getName(), command.getTimeout());
      Response success = Response.success(walEntry.getEntryIndex());
      if(command.hasClientId()) {
        Session session = clientSessions.get(command.getClientId());
        session.addResponse(success.withRequestNumber(command.getRequestNumber()));
      }
      return success;
    } catch(DuplicateLeaseException e) {
      return Response.error(DUPLICATE_LEASE_ERROR, e.getMessage(), walEntry.getEntryIndex());
    }
  }
}
```

客户端在发送给服务器的所有请求中附上自己的ID，同时，客户端也保存一个计数器，来给每个发送给服务器的请求赋一个编号。

```java
class ConsistentCoreClient {
  int nextRequestNumber = 1;
  
  public void registerLease(String name, Duration ttl) throws DuplicateLeaseException {
    RegisterLeaseRequest registerLeaseRequest = new RegisterLeaseRequest(clientId, nextRequestNumber, name, ttl.toNanos());
    nextRequestNumber++; // increment request number for next request.
    var serializedRequest = serialize(registerLeaseRequest);
    logger.info("Sending RegisterLeaseRequest for " + name);
    RequestOrResponse requestOrResponse = blockSendWithRetries(serializedRequest);
    Response response = JsonSerDes.deserialize(requestOrResponse.getMessageBodyJson(), Response.class);
    if (response.error == Errors.DUPLICATE_LEASE_ERROR) {
      throw new DuplicateLeaseException(name);
    }
  }
  
  private static final int MAX_RETRIES = 3;
  
  private RequestOrResponse blockingSendWithRetries(RequestOrResponse request) {
    for (int i = 0; i <= MAX_RETRIES; i++) {
      try {
        // blockingSend will attemp to create a new connection if there is no connection
        return blockingSend(request);
      } catch (NetWorkException e) {
        resetConnectionToLeader();
        logger.error("Failed sending request " + request + ". Try " + i, e);
      }
      
      throw new NetworkException("Timed out after " + MAX_RETRIES + " retries");
    }
  }
}
```

当服务器收到一个请求时，它首先根据这个请求的ID和客户端ID检查这个请求是否已经处理过，如果处理过，则返回缓存的响应，避免再次处理重复的请求。

```java
class ReplicatedKVStore {
  private Response applyWalEntry(WALEntry walEntry) {
    Command command = deserialize(walEntry);
    if (command.hasClientId) {
      Session session = clientSessions.get(command.getClientId());
      Optional<Response> savedResponse = session.getResponse(command.getRequestNumber());
      if (savedResponse.isPresent()) {
        return savedResponse.get();
      } // else continue and execute this command
    }
  }
}
```

### 移除response

服务器不能永远的缓存客户端的每个请求的响应。有多种方式可以来让缓存失效。在Raft中，每个客户端维护一个数字，记录最近一个成功收到响应的请求。客户端在每个请求中附上这个数字，服务器收到请求之后，就可以将小于这个数字的所有请求的响应删除。

如果客户端只有在上一个请求成功之后才会发送下一个请求，那么服务器可以成功地使用上述方式来将缓存的响应 删除。但是如果客户端使用了请求流水线技术（request pipeline），那么使用上面的这种方式就会导致问题。如果服务器知道客户端的请求流水线的并发度（窗口大小），那么它就可以只保存与窗口大小相等的请求个数。比如，Kafka对于生产者设置5作为它的最大请求个数，所以它可以保存最多5个响应。

```java
class Session…

  private static final int MAX_SAVED_RESPONSES = 5;

  public void addResponse(Response response) {
      if (clientResponses.size() == MAX_SAVED_RESPONSES) {
          clientResponses.remove(); //remove the oldest request
      }
      clientResponses.add(response);
  }
```

### 移除session

client的session也不能永远在服务器中保存。服务器可以给session设置一个最大值。每个客户端周期性地发送心跳。如果心跳超时，服务器就可以删除这个客户端的session。

服务器周期性地检查并删除过期的session。

```java
class ReplicatedKVStore {
  private long heartBeatIntervalMs = TimeUnit.SECONDS.toMillis(10);
  private long sessionTimeoutNanos = TimeUnit.MINUTES.toNanos(5);
  
  private void startSessionCheckerTask() {
    scheduledTask = executor.scheduleWithFixedDelay( () -> {
      removeExpiredSession();
    }, heartBeatIntervalMs, heartBeatIntervalMs, TimeUnit.MILLISECONDS);
  }
  
  private void removeExpiredSession() {
    long now = System.nanoTime();
    for (Long clientId : clientSessions.keySet()) {
      Session session = clientSessions.get(clientId);
      long elapsedNanosSinceLastAccess = now - session.getLastAccessTimestamp();
      if (elapsedNanosSinceLastAccess > sessionTimeoutNanos) {
        clientSessions.remove(clientId);
      }
    }
  }
}
```

## 例子

* Raft：利用幂等提供串行化
* Kafka：幂等生产者，允许客户端重新发送请求，忽略重复的请求。
* Zookeeper：有session的概念，允许客户端恢复。
* Hbase：有hbase-recoverable-zookeeper