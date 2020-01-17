# Dubbo  

## Consumer调用服务  

``` txt
proxy0#sayHello(String)  
  |—> InvokerInvocationHandler#invoke(Object, Method, Object[])  
    |—> MockClusterInvoker#invoke(Invocation)  
      |—> AbstractClusterInvoker#invoke(Invocation)  
        |—> FailoverClusterInvoker#doInvoke(Invocation, List<Invoker<T>>, LoadBalance)  
          |—> Filter#invoke(Invoker, Invocation)  // 包含多个 Filter 调用  
            |—> ListenerInvokerWrapper#invoke(Invocation)  
              |—> AbstractInvoker#invoke(Invocation)  
                |—> DubboInvoker#doInvoke(Invocation)  
                  |—> ReferenceCountExchangeClient#request(Object, int)  
                    |—> HeaderExchangeClient#request(Object, int)  
                      |—> HeaderExchangeChannel#request(Object, int)  
                        |—> AbstractPeer#send(Object)  
                          |—> AbstractClient#send(Object, boolean)  
                            |—> NettyChannel#send(Object, boolean)  
                              |—> NioClientSocketChannel#write(Object)  
```

proxy -> handler.invoke(this, Method, object...);

handler -> InvokerInvocationHandler implements InvocationHandler 内部有Invoker字段，handler.invoke()方法实际调用invoker.invoke(new RpcInvocation(method, args)).recreate()。注意下边所有invoker的invoke的入参就是这个new出来的RpcInvocation，就是一个封装。  

InvokerInvocationHandler 中的 invoker 成员变量类型为 MockClusterInvoker，MockClusterInvoker 内部封装了服务降级逻辑。内部持有一个invoker，实际主要是

``` java
try {
    // 调用其他 Invoker 对象的 invoke 方法
    result = this.invoker.invoke(invocation);
} catch (RpcException e) {
    if (e.isBiz()) {
        throw e;
    } else {
        // 调用失败，执行 mock 逻辑
        result = doMockInvoke(invocation, e);
    }
}
```

invoker除了上面的MockClusterInvoker主要有FailoverClusterInvoker和DubboInvoker。在DubboInvoker的doInvoke()方法中，先Round方式拿到一个client(暂不明这个client是什么，不应该是从注册中心拿到的)，接着根据配置异步无返回值(void)->oneway；异步有返回值生成一个ResponseFuture，ResponseFuture future = currentClient.request(inv, timeout)，放到ThreadLocal中，返回一个空结果；同步就直接阻塞在ResponseFuture的get()上。  
到这里应该invoke就好了。注意Dubbo如果使用Future，是这样获取的RpcContext.getContext().getCompletableFuture()。  

```java
/* 2.6.x DubboInvoker */
    // 异步无返回值
    if (isOneway) {
        boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
        // 发送请求
        currentClient.send(inv, isSent);
        // 设置上下文中的 future 字段为 null
        RpcContext.getContext().setFuture(null);
        // 返回一个空的 RpcResult
        return new RpcResult();
    }

    // 异步有返回值
    else if (isAsync) {
        // 发送请求，并得到一个 ResponseFuture 实例
        ResponseFuture future = currentClient.request(inv, timeout);
        // 设置 future 到上下文中
        RpcContext.getContext().setFuture(new FutureAdapter<Object>(future));
        // 暂时返回一个空结果
        return new RpcResult();
    }

    // 同步调用
    else {
        RpcContext.getContext().setFuture(null);
        // 发送请求，得到一个 ResponseFuture 实例，并调用该实例的 get 方法进行等待
        return (Result) currentClient.request(inv, timeout).get();
    }

/* 2.7.x DubboInvoker*/
    if (isOneway) {
        boolean isSent = getUrl().getMethodParameter(methodName, Constants.SENT_KEY, false);
        currentClient.send(inv, isSent);
        RpcContext.getContext().setFuture(null);
        return AsyncRpcResult.newDefaultAsyncResult(invocation);
    } else {
        AsyncRpcResult asyncRpcResult = new AsyncRpcResult(inv);
        CompletableFuture<Object> responseFuture = currentClient.request(inv, timeout);
        asyncRpcResult.subscribeTo(responseFuture);
        RpcContext.getContext().setFuture(new FutureAdapter(asyncRpcResult));
        return asyncRpcResult;
    }
/* AsyncToSyncInvoker */
    @Override
    public Result invoke(Invocation invocation) throws RpcException {
        Result asyncResult = invoker.invoke(invocation);

        try {
            if (InvokeMode.SYNC == ((RpcInvocation)invocation).getInvokeMode()) {
                asyncResult.get();
            }
        } catch (/* ... */) {
            // ...
        }
    }
```

注意上边都是2.6.x版本，在2.7.x版本中不再使用ResponseFuture(dubbo定义的interface)，而是使用JDK的CompletableFuture。在DubboInvoker中只返回AsyncRpcResult(继承自CompletableFuture)，由AsyncToSyncInvoker包装一下，如果是Sync在DubboInvoker返回之后.get()阻塞。如果使用了CompletableFuture那么CallBack模式就不是那么重要了。  

```java
// 这里ThreadLocal存的是RpcContext，那么应当RpcContext.getContext()每次只能拿到最新的，也就是Map中只会有一个slot给RpcContext用。RpcContext只是存了一些调用信息，真正的Future在FutureContext的中，这也是一个ThreadLocal
public class RpcContext {
    private static final InternalThreadLocal<RpcContext> LOCAL = new InternalThreadLocal<RpcContext>() {
        protected RpcContext initialValue() {
            return new RpcContext();
        }
    };

    public <T> CompletableFuture<T> getCompletableFuture() {
        return FutureContext.getContext().getCompletableFuture();
    }

    public <T> Future<T> getFuture() {
        return FutureContext.getContext().getCompletableFuture();
    }

    public void setFuture(CompletableFuture<?> future) {
        FutureContext.getContext().setFuture(future);
    }

}
// 注意Dubbo的InternalThreadLocal的实现和Netty类似。  
```

~~ReferenceCountExchangeClient(计数close) --持有并调用--> HeaderExchangeClient(处理心跳连接) --持有并调用--> HeaderExchangeChannel(在这里创建了Request) --持有并调用--> NettyClient的request方法继承自AbstractClient，主要是getChannel()然后调用channel.request方法，getChannel由子类实现。NettyClient内部持有Netty定义的Channel，但是要通过这个Channel拿到dubbo的NettyChannel --调用NettyChannel的request--> NettyChannel持有Channel调用其方法。~~  

在DubboInvoker之后分析client.send(或者request)方法。  
首先是ReferenceCountExchangeClient，主要起计数的作用，每调用一次close()给计数器减一，到0的时候关闭其持有的ExchangeClient。这个client所有的send/request都是调用其持有的ExchangeClient的对应方法。  
HeaderExchangeClient主要处理心跳连接，内部持有一个Client(Netty Client)和一个Channel(HeaderExchangeChannel)，这个Client主要用来reset之类的。send/request功能主要由HeaderExchangeChannel实现。  
HeaderExchangeChannel持有一个NettyClient，其send/request方法，首先创建一个Request然后调用client.send方法。  
NettyClient在调用connect等方法时会创建并连接Netty的NioSocketChannel，NettyClient的send/request方法继承自AbstractClient，先通过getChannel()从Map中通过Channel(Netty)拿到Dubbo的NettyChannel，然后调用NettyChannel的send方法。  
然后NettyChannel调用NioSocketChannel发送，同时做一些处理异常的增强功能。  
(既然NettyClient已经有了NioSocketChannel，为什么还要额外的getChannel()先拿到NettyChannel，再在里边调用NioSocketChannel的方法？为了下边通过Codec中通过ctx拿到对应的Dubbo的Channel)  

似乎所有都是先调用xxxClient的对应方法，内部在调用对应的xxxChannel实现。注意NettyClient(不是NettyChannel)在创建DubboInvoker的时候就创建好了，之后HeaderExchangeChannel都会引用这个Client。同一时间所有的NettyClient和NettyChannel和NioSocketChannel都是一一对应的。在发送msg时一定需要一个地方来指定序列化方式，Dubbo的这个参数是放在URL对象中的，在创建NettyClient时是以URL作为参数的，因此创建的一个Dubbo的Channel是绑定一种序列化方式的。所以猜测，对应的一个NioSocketChannel只能支持一种序列化方式。  
在NioSocketChannel中设置的Codec是NettyCodecAdapter里的InternalEncoder和InternalDecoder主要是做一个适配。InternalEncoder继承自Netty的MessageToByteEncoder，里边主要是调用AbstractCodec的encode(Channel channel, ChannelBuffer buffer, Request req)方法，里边的第一个参数是Dubbo的Channel，而MessageToByteEncoder只有Netty的ctx，因此通过上边的getChannel()拿到Dubbo的Channel。需要Dubbo的Channel对象主要是为了拿到其对应的URL的里边的哪种序列化方式，然后才可以进行序列化encode。原则上NIOSocketChannel和序列化无关，但是Dubbo将序列化作为URL的参数之一，然后URL又和Dubbo的Channel绑定，同一时间NIOSocketChannel又只能和一个Dubbo的Channel对应(在Map中)，因此NioSocketChannel只能支持一种序列化方式。  

对于一个Request对象其中保存有一些控制信息，比如version，是否twoWay，id等等，最重要的mdata这个字段，类型为RPCInvocation，其中保存了返回类型，参数Object[]等，还有额外的Map\<String, String\>attachments等。在写入NioChannel时进行序列化，注意序列化的是Invocation不是把Result整个序列化。  

## Provider提供服务  

``` txt
NettyHandler#channelRead(ChannelHandlerContext, MessageEvent)  
  —> AbstractPeer#received(Channel, Object)  [NettyServer这个Handler]
    —> MultiMessageHandler#received(Channel, Object)  
      —> HeartbeatHandler#received(Channel, Object)  
        —> AllChannelHandler#received(Channel, Object)  
          —> ExecutorService#execute(Runnable)    // 由线程池执行后续的调用逻辑  
线程池执行的是ChannelEventRunnable
ChannelEventRunnable#run()
  —> DecodeHandler#received(Channel, Object)
    —> HeaderExchangeHandler#received(Channel, Object)
      —> HeaderExchangeHandler#handleRequest(ExchangeChannel, Request)
        —> DubboProtocol.requestHandler#reply(ExchangeChannel, Object)
          —> Filter#invoke(Invoker, Invocation)
            —> AbstractProxyInvoker#invoke(Invocation)
              —> Wrapper0#invokeMethod(Object, String, Class[], Object[])
                —> DemoServiceImpl#sayHello(String)
```

注意所有的handler.received()方法都有一个Channel参数，在NettyServerHandler(ChannelDuplexHandler)中持有NettyServer这个Handler，然后也是通过NioChannel从Map中拿到NettyChannel，然后在channelRead中调用handler.received()方法。  
NettyHandler没有复写AbstractPeer的received方法，直接调用内部的handler的received方法。  
Dubbo支持多种线程派发策略，在ChannelHandlers中通过不同的Dispatcher创建不同的派发器默认AllChannelHandler将所有的处理都放进线程池中。然后AllChannelHandler外边再包装HeartbeatHandler和MultiMessageHandler。  
在AllChannelHandler中将必要信息包装成ChannelEventRunnable，包括NettyChannel(要写回去)，Handler(底下的DecodeHandler)等。  
DecodeHandler是为了确保已经解码过了(可以不解码直接放到线程池中)，然后交给HeaderExchangeHandler。对于双向通信，HeaderExchangeHandler首先向后进行方法调用，得到调用结果。然后将调用结果封装到 Response 对象中，最后再将该对象返回给服务消费方。如果请求不合法，或者调用失败，则将错误信息封装到 Response 对象中，并返回给服务消费方。  
Dubbo没有每次都使用反射，将一个服务动态生成为一个抽象类AbstractProxyInvoker的实现类对象。一个服务object动态生成一个abstractProxyInvoker对象，其doInvoke()方法综合考虑了object所有的方法。因此当收到调用请求时，直接找到这个service对应的Invoker调用invoke方法就好了，避免了反射。  

## Consumer接收回答  

Consumer拿到Response之后会从Map里边拿到对应的DefaultFuture然后complete它。  
