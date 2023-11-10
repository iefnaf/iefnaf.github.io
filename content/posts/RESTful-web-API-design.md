+++
title = '🥤 RESTful Web API Design'
date = 2023-11-10T15:15:04+08:00

+++

原文地址：https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design

绝大多数的web应用都会对外暴露API，以便客户端与之进行交互。一个设计良好的web API应该致力于实现：
- 平台无关。任何一个客户端都应该能够调用API，不管这个API内部是如何实现的。这需要使用标准的协议，并且有一个机制，让web服务端和客户端在交互数据的格式上达成一致。
- 服务升级。web API应该能够支持独立于客户端进行升级，以增加更多的功能。随着API的升级，现有的客户端应该不加修改就能够继续工作。所有的功能都应该能够被发现，以让客户端能够充分使用它们。

本教程将会介绍当你设计一个web API的时候应该考虑哪些问题。

## 什么是REST？
在2000年，Roy Fielding提出了Representational State Transfer(REST)作为设计web服务的一个框架结构。REST是一个基于hypermedia构造分布式系统的框架式的风格。REST是和任何的底层协议无关的，它不必依附于HTTP。然而，绝大多数REST API都是使用HTTP作为应用层协议，所以本教程聚焦于在HTTP之上设计REST API。

基于HTTP协议的REST的一个主要的优势在于，它使用了开放标准，让API独立于它的具体实现以及客户端。比如，一个REST的web服务可以使用ASP.NET来实现，客户端可以使用任何能够产生HTTP请求，解析HTTP响应的语言或者工具。

以下是一些基于HTTP协议的RESTful API的设计原则：
- REST API围绕着**资源**进行设计，所谓的资源，就是任何类型的对象，数据，或者客户端能够访问的服务。
- 一个资源拥有一个标识符，所谓的标识符就是能够唯一标识这个资源的URI。比如，下面这个URI标识了一个顾客订单：
```HTTP
https://adventure-works.com/orders/1
```
- 客户端和服务之间通过交换资源的标识来进行交互。很多web API使用JSON作为交互的格式。比如，一个对上面那个URI的GET请求返回如下响应：
```JSON
{"orderId":1,"orderValue":99.90,"productId":1,"quantity":1}
```
- REST API使用一个统一的接口，这样有助于将客户端和服务的实现解耦。对于构建在HTTP之上的REST API，统一接口包括使用标准的HTTP动词来在资源上执行动作。最常用的动作包括GET，POST，PUT，PATCH以及DELETE。
- REST API使用状态无关模型。HTTP请求必须彼此独立，并且可能以任意的次序出现。，所以在请求之间维护状态信息并不容易。信息存储的唯一的一个地方是资源本身，每个请求都必须是一个原子的操作。这个限制让web服务能够很好的扩展。任意一个服务器都需要能够处理客户端发送的任意一个请求。这意味着其他因素可能限制系统的扩展性。比如，很多web服务需要访问数据库，数据库难以进行扩展。关于数据库的更多扩展手段，可以看一下[Horizontal, vertical, and functional data partitioning](https://learn.microsoft.com/en-us/azure/architecture/best-practices/data-partitioning)。
- REST API由超媒体链接驱动。比如，下面展示了一个订单的JSON表示。它包含一些用来获得或者更新这个订单相关顾客的链接。
```JSON
{
    "orderID":3,
    "productID":2,
    "quantity":4,
    "orderValue":16.60,
    "links": [
        {"rel":"product","href":"https://adventure-works.com/customers/3", "action":"GET" },
        {"rel":"product","href":"https://adventure-works.com/customers/3", "action":"PUT" }
    ]
}
```

在2008年，Leonard Richardson对web API提出了如下几个评判标准：
- Level 0：定义一个URI，所有操作都是对这个URI的POST请求
- Level 1：为不同的资源创建不同的URI
- Level 2：使用HTTP方法定义在资源上的操作
- Level 3：使用超媒体（HATEOAS，下面会介绍）

Level 3涉及到Fielding关于一个真正的RESTful API的定义。在实际中，大多数web API都是Level 2。

## 围绕资源设计API
回到上面那个web API暴露的商业实体上。比如，在一个电商系统中，主要的实体可能是顾客和订单。可以通过发送一个包含订单信息的HTTP POST请求来创建一个订单。这个HTTP请求的响应反应了这个订单是否创建成功。资源URI应该尽量基于名词（这个资源），而不是动词（在这个资源上的操作）。
```HTTP
https://adventure-works.com/orders // Good

https://adventure-works.com/create-order // Avoid
```

一个资源不必基于一个单独的物理数据项。比如，一个订单资源可能在内部设计到数据库中的多个表，但是对于客户端作为一个实体来表示。避免创建简单映射内部数据库结构的API。REST的目的是对实体和应用可以在这些实体上所做的操作进行建模。内部的实现不应该对客户端暴露。

实体常常组织成集合（订单s，顾客s）。一个集合是一个独立的资源，所以它应该有自己的API。比如，下面这个URI可能表示了一组订单：
```HTTP
https://adventure-works.com/orders
```

给集合URI发送一个HTTP GET请求，可以取回集合中元素的一个列表。集合中的每个元素也有它们自己的一个唯一的URI。一个对元素URI的HTTP GET请求返回这个元素的详细信息。

对URI的命名习惯要一以贯之。通常，对于集合使用复数名词。把集合和元素组织称层次结构是一个良好的实践方式。比如，`/customers` 是customers集合的URI，`/customers/5` 是ID为5的顾客的URI。这个方法有利于保持API的直观性。很多web API框架可以基于参数化的URI路径路由请求，所以你可以定义一个`/customers/{id}` 这样的URI。

在设计URI的时候同样需要考虑不同类型资源之间的关系，以及你将会如何暴露这些关系。比如，`/customers/5/orders` 可能表示ID为5的顾客的所有订单。你也可以采用另外一种方式，将顾客放在订单下面，比如，`/orders/99/customer` 。然而，采用这种方式可能会导致实现起来比较困难。一个更好的方式是在HTTP响应的body部分提供一些链接。详见： [Use HATEOAS to enable navigation to related resources](https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design#use-hateoas-to-enable-navigation-to-related-resources).

在一些更加复杂的系统中，可能会尝试提供能够让客户端在关系的不同层次上进行切换，比如`/customers/1/orders/99/products`。然而，这种方式维护起来很复杂，将来扩展起来也很麻烦。应该尽量让URI保持简单。一旦一个客户端有一个资源的引用，它应该能够找到关于这个资源的实体。上面这个请求可以使用`/customers/1/orders` 这个URI来获得顾客1的所有订单，然后使用`/orders/99/products` 来获得这个订单里的所有商品。

💡Tip
**避免使用比collection/item/collection更加复杂的URI。**

另一个事实是所有的web请求对于web服务器来说都是一个负担。请求越多，负担越大。所以，避免使用暴露很多小资源的“聊天式”API。这种API通常需要一个客户端发送多个请求才能取到它所要的所有数据。因此，你可能需要对数据进行denormalize处理，将相关的信息合并成一个更大的资源，以便可以在一个请求里就能够得到。然而，你需要在这种方法和取数据的代价之间进行权衡，防止增加额外的带宽消耗。详见： [Chatty I/O](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/chatty-io/) 以及 [Extraneous Fetching](https://learn.microsoft.com/en-us/azure/architecture/antipatterns/extraneous-fetching/)

避免在API之间，以及底层的数据上引入依赖关系。比如，如果你的数据存储在关系型数据库上，API不必将每个表都暴露出来。实际上，这是一个很差的设计。相反，你应该把API作为数据库的一层抽象。如果可以的话，在数据库和API之间引入一个映射层。使用这种方式，可以将底层数据库的schema变更与客户端隔离开。

最后，将每个资源都映射成一个API不太现实。你可以用HTTP请求调用一个方法，将这个方法的结果作为HTTP的响应，使用这种方式来处理非资源的场景。比如，一个实现了简单的加减的应用可以将操作作为资源，然后将操作数作为请求的参数，`URI/add?operand1=99&operand2=1` 。然而，谨慎地使用这种方式。

## 用HTTP方法来定义API操作
HTTP协议定义了一些方法，并赋予了这些方法一定的语义。RESTful web API最常使用的方法如下：
- GET：获得URI所定义的资源。响应的body部分包括这个资源的详细信息。
- POST：创建URI所定义的一个新资源。请求的body部分提供关于这个资源的详细信息。注意，POST也可以用来触发不创建新资源的那些操作。
- PUT：创建或者替换URI所定义的资源。请求的body部分指示要被创建或者更新的资源。
- PATCH：部分更新URI所定义的资源。请求的body部分指示应用在资源上的一些更新。
- DELETE：删除URI所定义的资源。

一个请求的效果应该取决于这个资源是一个集合还是一个单独的数据。下面这个表格总结了大多数RESTful实现所遵循的惯例，以电商作为例子。可能并不是所有的请求都会被实现，这取决于具体的场景。

| Resource            | POST                  | GET                 | PUT                               | DELETE              |
| ------------------- | --------------------- | ------------------- | --------------------------------- | ------------------- |
| /customers          | 创建一个新的顾客      | 获得所有的顾客      | 批量更新顾客                      | 删除所有的顾客      |
| /customers/1        | Error                 | 获得顾客1的详细信息 | 更新顾客1的详细信息，如果存在的话 | 删除顾客1           |
| /customers/1/orders | 为顾客1创建一个新订单 | 获得顾客1的所有订单 | 批量更新顾客1的订单               | 删除顾客1的所有订单 |

POST、PUT和PATCH之间的差别可能比较让人困惑。
- POST请求创建一个新的资源。服务端为新的资源赋予一个URI，然后将这个URI返回给客户端。在REST中，你会经常对一个集合应用POST请求。新的资源被加入到这个集合中。一个POST请求也可以用来提交一个对于已经存在的资源所处理的所需数据，这种情况不会创建新的资源。
- PUT请求创建一个新的资源，或者更新一个已经存在的资源。客户端指定资源的URI，请求的body包含了关于这个资源的完整描述信息。如果URI所指代的资源已经存在，那么这个资源将会被替换。否则，如果服务端支持的话，一个新的资源将会被创建。PUT请求常常用于独立数据类型的资源，比如某一个顾客，而不是顾客集合。服务端可能支持使用PUT进行更新，而不是创建。是否支持通过PUT进行创建取决于客户端是否能够有意义的给还没存在的资源赋予一个URI。如果不能，那么使用POST来创建资源，并使用PUT或者PATCH进行更新。
- PATCH请求对已经存在的资源进行局部更新。客户端指示资源所对应的URI，请求的body包含应用到这个资源的一批更新。这比PUT更加高效，因为客户端指发送更新（DIFF），而不是整个资源的所有表示。从技术场来讲，PATCH也可以用来创建一个新的资源，如果服务端支持的话。

PUT请求必须是幂等的。如果一个客户端提交多次PUT请求，那么结果应该是相同的。POST和PATCH请求并不保证是幂等的。
