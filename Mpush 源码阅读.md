#  Mpush 源码阅读

## 系统架构

![image-20191224115751867](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\image-20191224115751867.png)

推送系统的server 是采用集群的方式，启动应用时会注册到zookeeper上去，然后业务系统使用SDK里pushClient来推送消息，这个pushClient会通过Zookeeper的服务发现会得到集群里所有节点的信息，然后都使用Netty连接上；

手机上SDK会通过Alloc这个接口调用Zookeeper里集群节点信息来获取合适的某个pushServer，获取pushServer的地址信息来进行连接，pushServer收到连接后，会把这个连接信息存储到Redis里去；

当业务系统需要对某用户发消息时，可通过用户的ID来查询路由信息，即实体连接在哪台pushServer上，然后把发送的信息推送给某个pushServer，pushServer收到消息就会根据用户ID查找连接发诙消息；

当业务系统需要群发消息即通知所有在线用户时，遍历所有的pushServer发送消息。

##  推送流程![img](https://static.oschina.net/uploads/img/201611/01193442_ABeq.png)



* 初始化客户端

com.mpush.client.push.PushClient#start

```
 protected void doStart(Listener listener) throws Throwable {
        if (mPushClient == null) {
            mPushClient = new MPushClient();
        }

        pushRequestBus = mPushClient.getPushRequestBus();
        cachedRemoteRouterManager = mPushClient.getCachedRemoteRouterManager();
        gatewayConnectionFactory = mPushClient.getGatewayConnectionFactory();

        ServiceDiscoveryFactory.create().syncStart();
        CacheManagerFactory.create().init();
        pushRequestBus.syncStart();
        gatewayConnectionFactory.start(listener);
    }
```

* 初始化路由缓存信息

* 初始网关连接，连上所有pushServer

* 初始化服务发现

* 初始化缓存工厂

  构建推送消息

com.mpush.api.push.PushMsg#build

//构建推送消息上下文

com.mpush.api.push.PushContext#build(com.mpush.api.push.PushMsg)

//调用sender发送

com.mpush.api.push.PushSender#send(com.mpush.api.push.PushContext)

//查找用户的路由信息

com.mpush.common.router.CachedRemoteRouterManager#lookupAll

//构建推送请求实体

com.mpush.client.push.PushRequest#build

//调用发送

com.mpush.client.push.PushRequest#send(router)

//获取网关工厂

com.mpush.client.MPushClient#getGatewayConnectionFactory

//调用根据路由信息找到网关连接

com.mpush.client.gateway.connection.GatewayConnectionFactory#send

//再次把消息包装到网关信息

com.mpush.common.message.gateway.GatewayPushMessage#build

//编码

com.mpush.common.message.BaseMessage#sendRaw(io.netty.channel.ChannelFutureListener)

com.mpush.common.message.BaseMessage#encodeBodyRaw

com.mpush.api.connection.Connection#send(com.mpush.api.protocol.Packet, io.netty.channel.ChannelFutureListener)



* 服务端接收到推送请求消息

  //服务端netty接收数据

  com.mpush.core.server.ServerChannelHandler#channelRead

  //调用包解析器对消息进行分发

  com.mpush.api.message.PacketReceiver#onReceive

  com.mpush.common.MessageDispatcher#onReceive

  //调用父类`handle`方法 将依次调用子类的 `decode`进行解码再调用 `handle` 处理具体消息

  com.mpush.common.handler.BaseMessageHandler#handle(com.mpush.api.protocol.Packet, com.mpush.api.connection.Connection)

     ->com.mpush.core.handler.GatewayPushHandler#decode

     ->com.mpush.core.handler.GatewayPushHandler#handle

  //解析完后向推送中心发消息

  com.mpush.core.push.PushCenter#push

  //推送中心构建一个推送任务放进任务池里

  com.mpush.core.push.SingleUserPushTask#new

  com.mpush.core.push.PushCenter#addTask

  //执行推送任务

  com.mpush.core.push.SingleUserPushTask#checkLocal

  ->com.mpush.common.message.PushMessage#build

  com.mpush.common.message.BaseMessage#send(io.netty.channel.ChannelFutureListener)

  ->com.mpush.common.message.BaseMessage#encodeBody

  ->com.mpush.api.connection.Connection#send(com.mpush.api.protocol.Packet, io.netty.channel.ChannelFutureListener)

  

   ![架构图](https://static.oschina.net/uploads/img/201610/29215003_BWQU.png) 

  

  

  





