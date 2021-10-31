# info

当应用程序发起io调用时会经历两个步骤

1. 内核等待io设备准备好数据
2. 内核将数据传送到用户态

## unix io模型

阻塞：用户态等待内核



1. 同步阻塞
2. 同步非阻塞
3. 多路复用
4. 信号量驱动
5. 异步

## java socket编程的演变史

bio-->block io模型有4种

1. 一个服务器一个时间里只能处理一个客户端，且服务器没有响应。
```java
/**
 * 这个服务器没有向客户端输出
 */
public class SocketService {
    //搭建服务器端
    public static void main(String[] args) throws IOException {
        //创建服务器
        ServerSocket server=new ServerSocket(5209);
        System.out.println("服务器启动成功");
        //等待客户端连接后，接收客户端socket -->阻塞
        Socket socket=server.accept();
        //获取客户端socket的输入流
        BufferedReader in=new BufferedReader(new InputStreamReader(socket.getInputStream()));
        while(true){
            //等待客户端socket的不为空输入流-->也会阻塞
            String str = in.readLine();
            if (str == null) {
                break;
            }
            System.out.println("客户端说：" + str);
        }
        in.close(); //关闭Socket输入流
        socket.close(); //关闭Socket
        server.close(); //关闭ServerSocket
    }

}
```

2.一个服务器一个时间里只能处理一个客户端，服务器有响应，交互了
```java
/**
 * 这个服务器端向客户端输出
 */
public class SocketService {
    //搭建服务器端
    public static void main(String[] args) throws IOException {
        ServerSocket server=new ServerSocket(80);
        System.out.println("服务器启动成功");
        // accept是阻塞的
        Socket socket=server.accept();
        BufferedReader in=new BufferedReader(new InputStreamReader(socket.getInputStream()));
        BufferedReader out=new BufferedReader(new InputStreamReader(System.in));
        PrintWriter pw = new PrintWriter(socket.getOutputStream());
        while(true){
            System.out.println("客户端说："+in.readLine());
            String str = out.readLine();
            pw.println(str);
            pw.flush();
            System.out.println("服务器说："+str);
        }
    }

}
```

3. 一个服务器可以回应多个客户端，但是没有响应回去
```java

/**
 * 多线程的状态 Socket实现多个客户端向服务器端通信 服务器没有响应回去
 */
public class SocketService {
    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(5208);
        System.out.println("server has bootstraped");
        while (true) {
            Socket socket = serverSocket.accept();
            System.out.println("有人上线了" + socket.getInetAddress() + ":" + socket.getPort());
            new Thread(new ServerThread(socket)).start();
        }
    }
}

```

4.一个服务器可以回应多个客户端,且有响应
```java

/**
 * Socket实现客户端与客户端通信
 */
public class SocketService {
    public static List<Socket> socketList = new ArrayList<>();

    public static void main(String[] args) throws  Exception{
        ServerSocket serverSocket = new ServerSocket(5208);
        System.out.println("启动服务器");
        while (true){
            Socket socket = serverSocket.accept();
            System.out.println("有人上线:"+socket.getPort());
            socketList.add(socket);
            new Thread(new ServerThread(socket)).start();
        }
    }
}
```

nio-noblock io 模型

其实处理能力也有限，但是运用了响应式编程的思想，事件驱动。
打开通道，监听通道。
每收到消息，根据消息的事件来做处理

```java
public class NioServer {
    //通道管理器
    private Selector selector;

    //获取一个ServerSocket通道，并初始化通道
    public NioServer init(int port) throws IOException {
        //获取一个ServerSocket通道
        ServerSocketChannel serverChannel = ServerSocketChannel.open();
        serverChannel.configureBlocking(false);
        serverChannel.socket().bind(new InetSocketAddress(port));
        //获取通道管理器
        selector = Selector.open();
        //将通道管理器与通道绑定，并为该通道注册SelectionKey.OP_ACCEPT事件，
        //只有当该事件到达时，Selector.select()会返回，否则一直阻塞。
        serverChannel.register(selector, SelectionKey.OP_ACCEPT);
        return this;
    }

    public void listen() throws IOException {
        System.out.println("服务器端启动成功");

        //使用轮询访问selector
        while (true) {
            //当有注册的事件到达时，方法返回，否则阻塞。
            selector.select();

            //获取selector中的迭代器，选中项为注册的事件
            Iterator<SelectionKey> ite = selector.selectedKeys().iterator();
            System.out.println("!!!!!!!!!!");

            while (ite.hasNext()) {
                SelectionKey key = ite.next();
                //删除已选key，防止重复处理
                ite.remove();
                //客户端请求连接事件
                if (key.isAcceptable()) {
                    ServerSocketChannel server = (ServerSocketChannel) key.channel();
                    //获得客户端连接通道
                    SocketChannel channel = server.accept();
                    channel.configureBlocking(false);//?
                    //向客户端发消息
                    channel.write(ByteBuffer.wrap(new String("send message to client").getBytes()));
                    //在与客户端连接成功后，为客户端通道注册SelectionKey.OP_READ事件。
                    channel.register(selector, SelectionKey.OP_READ);

                    System.out.println("客户端请求连接事件");
                } else if (key.isReadable()) {//有可读数据事件
                    //获取客户端传输数据可读取消息通道。
                    SocketChannel channel = (SocketChannel) key.channel();
                    //创建读取数据缓冲器
                    ByteBuffer buffer = ByteBuffer.allocate(10);
                    int read = channel.read(buffer);
                    byte[] data = buffer.array();
                    String message = new String(data);

                    System.out.println("receive message from client, size:" + buffer.position() + " msg: " + message);
                    ByteBuffer outbuffer = ByteBuffer.wrap(("server.".concat(message)).getBytes());
                    channel.write(outbuffer);
                }
            }
        }
    }


}
```