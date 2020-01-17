#JAVA NIO

JAVA NIO的定义

IO大家都知道是Input 与Output的缩写，而N是New的意思，java 1.4在不替换原有的IO实现的情况下，新增加一套IO机制，所以叫NIO，然后旧的叫OIO。

为什么学习Java NIO？

因为高性能的通讯框架底层很多都用到了它，如果不学习它，导致在看那些框架的源码看不懂，比如Netty，tomcat 7+等

它出现的目的

因为java为了实现write once run anywhere做出个JVM，通过Jvm屏蔽操作系统之间差异，带来的好处很明显，劣势就是屏蔽太多系统之间的特点，比如一些操作系统所对IO的优势，所以javaNIO就是解决OIO所存在的IO问题的，比如OIO会存在的就数据读取是网卡读取内核的空间，再拷贝到用户空间，在NIO里有直接支持零拷贝技术，所以Netty就是利用这一点实现压榨服务器资源。



如下根据java NIO 实现聊天室的Server端代码

~~~

public class ServerSocketWithNIO {

    private List<SocketChannel> socketChannels = new ArrayList<>();

    public static void main(String[] args) throws IOException {
        new ServerSocketWithNIO().go(args);
    }
    public void go(String[] args) throws IOException {
    	//创建serverSocketChannel
        ServerSocketChannel serverSocketChannel =   = SelectorProvider.provider().open();
		
		//绑定端口
        ServerSocket serverSocket = serverSocketChannel.socket();
        serverSocketChannel.configureBlocking(false);
        serverSocket.bind(new InetSocketAddress(Integer.parseInt(args[0])));

		//创建Selector
        Selector selector = Selector.open();
        //注册感兴趣事件
        serverSocketChannel.register(selector, SelectionKey.OP_ACCEPT);
        while (true) {
        	//获取就绪事件个数
            int select = selector.select();
            if (select > 0) {
            	//获取IO已就绪的selectedKeys
                Iterator<SelectionKey> iterator = selector.selectedKeys().iterator();
                while (iterator.hasNext()) {
                    SelectionKey selectionKey = iterator.next();
                    
                    //新连接进来，注册读事件
                    if (selectionKey.isAcceptable()) {
                        ServerSocketChannel channel = (ServerSocketChannel) selectionKey.channel();
                        SocketChannel socketChannel = channel.accept();
                        socketChannel.configureBlocking(false);
                        socketChannels.add(socketChannel);

                        socketChannel.register(selector, SelectionKey.OP_READ);
                        sayHello(socketChannel);
                    }

  					 //读取数据
                    if (selectionKey.isReadable()) {
                        readDataFromChannel((SocketChannel) selectionKey.channel());
                    }
                    //处理完之前要从集合里删除
                    iterator.remove();
                }
            }
        }
    }

    private void readDataFromChannel(SocketChannel channel) throws IOException {
        ByteBuffer allocate = ByteBuffer.allocate(1024);
        while (channel.read(allocate) > 0) {
            allocate.flip();
            while (allocate.hasRemaining()) {
                //向所有的发送消息
                socketChannels.forEach(socketChannel -> {
                    if (socketChannel != channel) {
                        try {
                            socketChannel.write(allocate);
                        } catch (IOException e) {
                            e.printStackTrace();
                        }
                    }
                });
            }
            allocate.clear();
        }
    }
    private void sayHello(SocketChannel socketChannel) throws IOException {
        socketChannel.write(UTF_8.encode("欢迎来到嘿嘿聊天室!!!"));
    }
}
~~~

其中表涉及到四个概念buffer,channel,seletor，selectKey

其中 channel理解成一个连接通道，可能基于这个通道进行一系列的IO操作，如socket上的操作accept, read, write等

然后buffer可以理解成一个缓冲区，在channel读取到的字节可以放到buffer里存放然后读取等；

selector可以根据字面的意思理解成一个选择器，就是根据注册到上面的channel列表选择出可进行IO操作的Channel;

selectKey可以理解channel与selector之间的注册关系

所以它的主要流程

1. 创建channel
2.  绑定端口
3. 创建selector
4. 把channel根据感兴趣的事件注册到selector
5. 循环获取selector返回IO就绪的channel列表，然后做相应的业务处理



其中这四个概念详细学习可以下载java NIO,看完它会对java NIO有个整体的了解

