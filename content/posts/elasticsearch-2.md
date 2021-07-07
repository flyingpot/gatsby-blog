+++
categories = ["Elasticsearch源码解析"]
date = 2021-04-12T16:00:00Z
tags = ["Elasticsearch", "Java"]
title = "Elasticsearch源码解析——通信模块（二）"
url = "/post/elasticsearch-network2"

+++
书接上文，本文继续讲解ES的通信模块。第一部分介绍了ES的总体通信模块组成，并详细介绍了ES的Rest通信模块部分。本文就来介绍ES的transport通信模块部分。

### 一、底层Netty部分

还是看ES的底层默认Netty实现部分，Netty4Transport中实现了transport的初始化。这里与Rest部分差别如下：

| Rest | Transport |
| --- | --- |
| 要实现TCP层和HTTP层的处理 | 只要实现TCP层处理 |
| 只需要实现服务端 | 需要实现客户端和服务端 |

Rest请求的处理，作为ES服务的入口，需要实现HTTP协议的服务端，而集群的内部请求既需要发送也要接受，所以需要实现服务端和客户端两部分。而在协议上为了省略HTTP协议解析的消耗，可以直接在TCP层上做，以提高通信效率。

与Netty4HttpServerTransport类似，协议层面的解析都是在这两个类中做的，而实际解析出来的对象是在父类中处理的。Netty4Transport父类TcpTransport的构造函数：

```java
    public TcpTransport(Settings settings, Version version, ThreadPool threadPool, PageCacheRecycler pageCacheRecycler,
                        CircuitBreakerService circuitBreakerService, NamedWriteableRegistry namedWriteableRegistry,
                        NetworkService networkService) {
        this.settings = settings;
        this.profileSettings = getProfileSettings(settings);
        this.version = version;
        this.threadPool = threadPool;
        this.pageCacheRecycler = pageCacheRecycler;
        this.circuitBreakerService = circuitBreakerService;
        this.networkService = networkService;
        String nodeName = Node.NODE_NAME_SETTING.get(settings);
        BigArrays bigArrays = new BigArrays(pageCacheRecycler, circuitBreakerService, CircuitBreaker.IN_FLIGHT_REQUESTS);

        this.outboundHandler = new OutboundHandler(nodeName, version, statsTracker, threadPool, bigArrays);
        this.handshaker = new TransportHandshaker(version, threadPool,
            (node, channel, requestId, v) -> outboundHandler.sendRequest(node, channel, requestId,
                TransportHandshaker.HANDSHAKE_ACTION_NAME, new TransportHandshaker.HandshakeRequest(version),
                TransportRequestOptions.EMPTY, v, false, true));
        this.keepAlive = new TransportKeepAlive(threadPool, this.outboundHandler::sendBytes);
        this.inboundHandler = new InboundHandler(threadPool, outboundHandler, namedWriteableRegistry, handshaker, keepAlive,
            requestHandlers, responseHandlers);
    }
```

主要看最后的几行：入方向的handler是inboundHandler，出方向的handler是outboundHandler。还能看到handshaker和keeplive，这两个分别做两节点之间的握手和TCP保活的，后续有机会再讲它们。

### 二、入方向

首先看下inboundHandler，处理请求的方法是inboundMessage：

```java
    void inboundMessage(TcpChannel channel, InboundMessage message) throws Exception {
        final long startTime = threadPool.relativeTimeInMillis();
        channel.getChannelStats().markAccessed(startTime);
        TransportLogger.logInboundMessage(channel, message);

        if (message.isPing()) {
            keepAlive.receiveKeepAlive(channel);
        } else {
            messageReceived(channel, message, startTime);
        }
    }
```

如图是核心方法inboundMessage的调用关系图：

![](/images/2021-04-18-10-17-22.png)

client和server两部分都注册了这个handler，因为客户端和服务端都需要处理入方向的请求。

继续看messageReceived方法：

```java
    private void messageReceived(TcpChannel channel, InboundMessage message, long startTime) throws IOException {
		...
        final Header header = message.getHeader();
		if (header.isRequest()) {
			handleRequest(channel, header, message);
		} else {
			handler = responseHandlers.onResponseReceived(requestId, messageListener);
			if (handler != null) {
				handleResponse(remoteAddress, EMPTY_STREAM_INPUT, handler);
			}
		}
	}
```

这块代码我简化了很多，只留下了核心逻辑。根据传入的header判断消息是一个request还是response，是response的话就调用发送请求时注册在responseHandlers中的hander处理，这里会跟一个requestId对应，是request的话就继续处理：

```java
    private <T extends TransportRequest> void handleRequest(TcpChannel channel, Header header, InboundMessage message) throws IOException {
        final String action = header.getActionName();
        final StreamInput stream = namedWriteableStream(message.openOrGetStreamInput());
        final RequestHandlerRegistry<T> reg = requestHandlers.getHandler(action);
        final T request = reg.newRequest(stream);
        request.remoteAddress(new TransportAddress(channel.getRemoteAddress()));
        final String executor = reg.getExecutor();
        handler.messageReceived(request, taskTransportChannel, task);
    }
```

这里同样简化，实际上的处理工作是从requestHandlers里面拿出来的。如果要实现一个新的Transport请求，需要预先注册相应的requestHandler:

```java
    public <Request extends TransportRequest> void registerRequestHandler(String action, String executor,
                                                                          Writeable.Reader<Request> requestReader,
                                                                          TransportRequestHandler<Request> handler) {
        validateActionName(action);
        handler = interceptor.interceptHandler(action, executor, false, handler);
        RequestHandlerRegistry<Request> reg = new RequestHandlerRegistry<>(
            action, requestReader, taskManager, handler, executor, false, true);
        transport.registerRequestHandler(reg);
    }
```

* 参数action对应handler的名字
* 参数executor对应了线程的类型，执行时会从线程池中拿相应类型的线程来处理request
* 参数requestReader对应请求的reader，服务端根据reader来从网络IO流中反序列化出相应对象
* 参数handler对应实际的handler，做实际的处理工作

列举起来不是很清晰，举个例子：

```java
        registerRequestHandler(
            HANDSHAKE_ACTION_NAME,
            ThreadPool.Names.SAME,
            HandshakeRequest::new,
            (request, channel, task) -> channel.sendResponse(
                new HandshakeResponse(localNode.getVersion(), Build.CURRENT.hash(), localNode, clusterName)));
```

这是一个握手action的注册方法，处理方法是返回HandshakeResponse，包含了服务端节点的一些信息。根据这些内容，开发者就可以很简单地定义出自己的action。实际上，ES源码做了更细致地封装，根据具体需求来实现TransportAction的一些子类（定义在org.elasticsearch.action.support里面的包中）即可。

### 三、出方向

如方向这里就简单很多了，我们这次反方向来看。调用的入口是TransportService的sendRequest，然后是sendRequestInternal。比较重要的是这一行代码：

```java
        final long requestId = responseHandlers.add(new Transport.ResponseContext<>(responseHandler, connection, action));
```

这里就可以和上文对应上了，在发送请求时注册了一个对应requestId的responseHandler，然后在接收请求时拿出来requestId对应handler。

然后就到了OutboundHander类，这里其实除了sendRequest还有sendResponse方法。因为作为服务端的节点要发送response给客户端节点。其余的就是一个序列化操作，这里就不赘述了。

### 四、总结

本文简单梳理了transport模块的定义，从Netty底层到出入两个方向的逻辑。Transport部分还没有结束，下一篇打算介绍一下连接管理的内容。