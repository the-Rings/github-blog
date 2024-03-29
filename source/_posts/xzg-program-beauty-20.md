---
title: 20. nio类库：BIO、NIO、AIO三种Java I/O模型的实现原理和区别
date: 2023-06-12 18:30:10
categories:
- 编程之美（王争）
---
这篇文章和{% post_link xzg-program-beauty-20-1 '图解I/O多路复用' %}一起读

Java中的I/O类库除了java.io之外，还包括java.nio。既然已经有了java.io了，为什么还要再开发一个新的java.nio呢？java.nio跟java.io有何区别？在平时的开发中，什么时候使用java.io？什么时候使用java.nio？面试中常被问到的BIO、NIO、AIO又是什么东西？带着这些问题，我们来学习本节的内容：java.nio。

## 一、java.nio类库
java.nio类库在JDK1.4中引入，nio的全称为New I/O，不过，因为其相对于java.io来说，对I/O提供了非阻塞的访问方式（这个待会再讲），所以，很多人也把nio解读为Non-blocking I/O。除此之外，尽管从功能上java.nio可以完全替代java.io，但在平时的开发中，对于普通的文件读写，我们更倾向于使用简单的java.io，**java.nio发挥作用的场合更多的是网络编程。**所以，还有人把nio解读为Network I/O。

上一节中讲到，在java.io中，Stream是一个核心的概念，所有的I/O都抽象为Stream，读写Stream就等同于读写I/O。**在java.nio中，已经没有了Stream，转而引入了Channel。Channel类似Stream，也是对I/O的抽象。除此之外，java.nio还引入了一个新的概念：Buffer，用来存储待写入或读取的数据。**


我们先拿一个比较简单的文件读写的例子，来看一下Channel和Buffer是如何使用的，让你对java.nio有个最初步的直观的认识。示例代码如下所示。
```Java
FileChannel channel = FileChannel.open(Paths.get("/Users/wangzheng/in.txt"));
ByteBuffer buffer = ByteBuffer.allocate(512);
while (channel.read(buffer) != -1) {
  // 处理buffer中的数据data
}
```


除了上面提到的Buffer、Channel之外，java.nio中还有两个重要的概念：Selector和AsynchronousChannel，接下来，我们就详细介绍一下java.nio中的这4个核心概念。

#### 1）Buffer
Buffer本质上就是一块内存，就相当于在使用java.io编程时申请的byte数组。常用到Buffer有：ByteBuffer、CharBuffer、ShortBuffer、IntBuffer、LongBuffer、FloatBuffer、DoubleBuffer、MappedByteBuffer。这些Buffer的不同之处在于：解析数据的方式不同，比如CharBuffer按照字符来解析数据，有点类似java.io中的字符流。


上一节我们讲到，java.io的设计有诸多问题，而java.nio的设计要优于java.io。上一节讲到，java.io分别为字符流和字节流设计了不同的类库，在代码实现上有些重复，毕竟I/O读写操作都是相同的，唯一的区别只是数据的解析方式不同。字节流类按照字节解析数据。字符流类按照字符解析数据。java.nio将解析这部分功能抽取出来，独立到Buffer类中。不同的Channel跟不同的Buffer组合在一起，可以实现不同的IO读写需求。比如，将FileChannel跟ByteBuffer组合起来，就相当于java.io中的文件字节流类（FileInputStream、FileOutputStream），将FileChannel跟CharBuffer组合起来，就相当于java.io中的文件字符流类（FileReader、FileWriter）。


实际上，Channel和Buffer独立开发，组合起来使用，这种设计思路应用的就是面向对象中“组合优于继承”的设计思想，通过组合来替代继承，避免了继承带来的组合爆炸问题。正因如此，实现相同甚至更多功能的情况下，java.nio中的类的个数却比java.io中的类的个数少。关于“组合优于继承”这一设计思想的详细介绍，你可以阅读我的《设计模式之美》这本书。



#### 2）Channel
常用的Channel有：FileChannel、DatagramChannel、SocketChannel、ServerSocketChannel。FileChannel用于文件读写。DatagramChannel、SocketChannel、ServerSocketChannel用于网络编程。DatagramChannel用来读写UDP数据，SocketChannel和ServerSocketChannel用来读写TCP数据。**SocketChannel和ServerSocketChannel的区别在于，ServerSocketChannel用于服务器编程，可以使用accept()函数监听客户端（SocketChannel）的连接请求。**


java.nio中的Channel既可以读，也可以写，而java.io中的Stream要么只能读，要么只能写，这也是java.nio比java.io类少的另一个重要原因。除此之外，Channel的设计也利用了“组合优于继承”的设计思想。 java.nio中包含大量的Channel接口，每个接口定义了一种功能。每个Channel类通过实现不同的接口组合，来支持不同的功能组合。如下图所示。其中，FileChannel实现了3个接口，支持3种不同的功能。
![java.nio.png](1.png)

**Channel有两种运行模式：阻塞模式和非阻塞模式。**其中，FileChannel只支持阻塞模式。DatagramChannel、SocketChannel、ServerSocketChannel支持阻塞和非阻塞两种模式，默认为阻塞模式。我们可以调用configureBlocking(false)函数将其设置为非阻塞模式。**非阻塞Channel一般会配合Selector，用于实现多路复用I/O模型。**


那么，到底什么是阻塞模式？什么是非阻塞模式呢？


**线程在调用read()或write()函数对I/O进行读写时，如果I/O不可读或者不可写（待会解释这两个的意思），那么，在阻塞模式下，read()或write()函数会等待，直到读取到数据或者写入完成时才会返回，在非阻塞模式下，read()或write()函数会直接返回，并报告读取或写入未成功。**


上一节，我们提到，**在操作系统层面，主要的I/O有：文件、网络、标准输入输出、管道。文件是没有非阻塞模式的。毕竟文件不存在不可读和不可写的情况。网络、标准输入输出、管道都存在阻塞和非阻塞两种模式。**我们拿最常用的网络来举例。


**一般来讲，应用程序调用read()或write()函数读取或写入数据，数据会在应用程序缓冲区、内核缓冲区、I/O设备这三者之间拷贝传递。**如下图所示。关于这点，我们下一节详细介绍。
![copy.png](2.png)


**当调用read()函数时，如果内核读缓冲区中没有数据可读，比如网络连接的对方此时并未发送数据过来，那么，在阻塞模式下，read()函数会等待，直到对方发送数据过来，内核读缓冲区中有数据可读时，才会将内核读缓冲区中的数据拷贝到应用程序缓存中，然后read()函数才返回，在非阻塞模式下，read()函数会直接返回，并报告读取情况。**

**当调用write()函数时，如果内核写缓冲区中没有足够空间承载应用程序缓存中的数据，比如网络不好，原来的数据还没来得及发送出去，那么，在阻塞模式下，write()函数会等待，直到内核写缓冲区中有足够空间，应用程序缓冲区中的数据全部写入内核写缓冲区，write()函数才会返回。在非阻塞模式下，write()函数会能写多少写多少，即便还有一部分未能写入内核写缓冲区，也不会等待，直接返回，并报告写入情况。**

**实际上，除了read()和write()函数有阻塞和非阻塞这两种模式之外，ServerSocketChannel中用于服务器接收客户端的连接的accpet()函数，也有阻塞和非阻塞两种模式。在阻塞模式下，调用accept()函数会等待，直到有客户端连接到来才返回。在非阻塞模式下，调用accept()函数，如果没有客户端连接到来，会直接返回。**


#### 3）Selector
在网络编程中，使用非阻塞模式，线程需要通过while循环，不停轮询调用read()、write()、accept()函数，查看是否有数据可读、是否可写、是否有客户端连接到来。如下所示。
```Java
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.bind(new InetSocketAddress("127.0.0.1", 1192));
serverChannel.configureBlocking(false);
ByteBuffer buffer = ByteBuffer.allocate(1024);

SocketChannel clientChannel = null;
while (clientChannel == null) {
  clientChannel = serverChannel.accept();
}

while (clientChannel.read(buffer) == -1);

buffer.flip(); //将buffer从"用于读"变成"用于写"
while(buffer.hasRemaining()) {
  clientChannel.write(buffer); // echo,读了啥就写啥
}
```



上述代码充斥着while轮询，显然不够优雅。多路复用I/O模型便用来解决这个问题。


**多路复用I/O模型是网络编程中非常经典的一种I/O模型。为了实现多路复用I/O模型，Unix提供了epoll库，Windows提供了iocp库，BSD提供了kequeue库...Java作为一种跨平台语言，对不同操作系统的实现方式进行了封装，提供了统一的Selector。**


**我们可以将需要监听的Channel，调用register()函数，注册到Selector中。Selector底层会通过轮询的方式，查看哪些Channel可读、可写、可连接等，并将其返回处理。**关于Selector的使用示例代码，我们在接下来的「Java I/O模型」小节中给出。


#### 4）异步Channel
尽管使用Selector可以避免程序员自己手写轮询代码，但是Selector底层仍然依赖轮询来实现。在JDK7中，java.nio类库做了升级，引入了支持异步模式的Channel，主要包括：AsynchronousFileChannel、AsynchronousSocketChannel、AsynchronousServerSocketChannel。而前面讲到的Channel都是同步模式的。


>那么，什么是同步模式？什么是异步模式呢？同步和异步这两个概念，跟阻塞和非阻塞又是否有联系呢？我们通过一个生活中的例子来给你形象解释一下。
>假设你去一家餐厅就餐，因为就餐的人太多，需要取号等位。取号之后，如果你站在餐厅门口一直等待被叫号，啥都不干，那么，这就是阻塞模式。如果你先去商场里逛一逛，一会回来看一下有没有轮到你，没有就继续再去逛，那么，这就是非阻塞模式。
>如果你在取号时，登记了手机号码，那么你就可以放心去逛商场了，等叫到你的号时，服务员会打电话通知你，这就是异步模式。相反，如果需要自己去查看有没有轮到你，不管是阻塞模式还是非阻塞模式，都是同步模式。
>实际上，异步模式下也可以有阻塞和非阻塞之分。如果在没有收到通知时，尽管你可以去干其他事情，但你偏偏就啥都不干，就站在门口等着被叫号，那么这就是阻塞异步模式，如果你选择享受通知服务，去干其他事情，那么这就是非阻塞异步模式。

从上面的解释，我们可以发现，同步、异步跟阻塞、非阻塞没有直接关系。

**在异步模式下，Channel不再注册到Selector，而是注册到操作系统内核中，由内核来通知某个Channel可读、可写或可连接，java.nio收到通知之后，为了不阻塞主线程，会使用线程池去执行事先注册的回调函数。**关于异步模式的用法，我们也是在「Java I/O模型」小节中展示。



#### 二、Java IO模型
**在面试和工作中，我们经常听到“I/O模型”这个概念。I/O模型一般用于网络编程中，所以，“I/O模型”的全称是“网络I/O模型”。**除此之外，I/O模型多数都用来指导服务器开发。相比服务器开发，客户端开发不需要处理多个并发连接的情况，往往会简单一些，也就不需要这些复杂的模型。


Java中常被提及的I/O模型有三个：阻塞I/O模型（BIO）、非阻塞I/O模型（NIO）、异步I/O模型（AIO）。我们依次看下这3种常见的I/O模型。

#### 1）阻塞I/O模型（BIO）
前面讲过，I/O访问模式有两种：阻塞模式和非阻塞模式。阻塞I/O模型指的是利用阻塞模式来实现服务器。一般来说，这种模型需要配合多线程来实现。

一般来讲，服务器需要连接大量客户端，因为read()函数是阻塞函数，所以，为了实时接收客户端发来的数据，服务器需要创建大量线程，每个线程负责等待读取（调用read()函数）一个客户端的数据。因为java.io支持阻塞模式，java.nio既支持阻塞模式又支持非阻塞模式，所以，java.io和java.nio都可以实现阻塞I/O模型。我们使用java.io来编写示例代码，如下所示。注意，使用java.io进行网络编程，需要配合java.net类库。比如在下面代码中，Socket、ServerSocket都是java.net包中的类。java.net类库用于管理连接，java.io用于读写数据。
```Java
public class BioEchoServer {
  public static void main(String[] args) throws IOException {
    ServerSocket serverSocket = new ServerSocket();
    serverSocket.bind(new InetSocketAddress("127.0.0.1", 1192));
    while (true) {
      // accept()为阻塞函数，直到有连接到来才返回
      Socket clientSocket = serverSocket.accept();
      // 为了每个客户端单独创建一个线程处理
      new Thread(new ClientHandler(clientSocket)).start();
    }
  }

  private static class ClientHandler implements Runnable {
    private Socket socket;
    public ClientHandler(Socket socket) {
      this.socket = socket;
    }

    @Override
    public void run() {
      byte[] data = new byte[1024];
      while (true) { //持续接收客户端发来的数据
        try {
          // read()为阻塞函数，直到读取到数据再返回
          socket.getInputStream().read(data);
          // write()为阻塞函数，全部写完成才会返回
          socket.getOutputStream().write(data); //echo
        } catch (IOException e) {
          // log and exit
          break;
        }
      }
    }
  }
}
```

**如果有n个客户端连接服务器，那么服务器需要创建n+1个线程，其中n个线程用于调用read()函数。除此之外，因为accept()函数也是阻塞函数，所以也独占一个线程。当连接的客户端非常多时，服务器需要创建大量线程，而每个线程会分配一个线程栈，需要占用一定的内存空间。当线程比较多时，内存资源的消耗就会比较大。大量线程来回切换，也会导致服务器整体处理性能的下降。除此之外，大部分线程可能都阻塞在read()函数上，等待数据的到来，什么都不做但又要白白占用内存和线程资源，非常浪费。**


#### 2）非阻塞I/O模型（NIO）
**非阻塞I/O模型指的是利用非阻塞模式来开发服务器，一般需要配合Selector多路复用器，所以，这种模型也叫做多路复用I/O模型。**不过，这两种叫法都有点以偏概全，所以，你不必太纠结于名称，知道这种模型具体是如何实现的即可。

因为java.io只支持阻塞模式，所以，这种模型只能通过java.nio来实现。非阻塞I/O模型的示例代码如下所示。利用java.nio进行网络编程，也像java.io那样，需要java.net类库的配合，比如代码中的InetSocketAddress就是java.net中的类。
```Java
public class NioEchoServer {
  public static void main(String[] args) throws IOException {
    // Selector
    Selector selector = Selector.open();

    // create serverChannel and register to selector
    ServerSocketChannel serverChannel = ServerSocketChannel.open();
    serverChannel.bind(new InetSocketAddress("127.0.0.1", 1192));
    serverChannel.configureBlocking(false);
    serverChannel.register(selector, SelectionKey.OP_ACCEPT);

    ByteBuffer buffer = ByteBuffer.allocate(1024);
    while (true) {
      int channelCount = selector.select();
      if (channelCount > 0) {
        Set<SelectionKey> keys = selector.selectedKeys();
        Iterator<SelectionKey> iterator = keys.iterator();
        while (iterator.hasNext()) {
          SelectionKey key = iterator.next();
          if (key.isAcceptable()) {
            // create clientChannel and register to selector
            SocketChannel clientChannel = serverChannel.accept();
            clientChannel.configureBlocking(false);
            clientChannel.register(selector, SelectionKey.OP_READ);
          } else if (key.isReadable()) {
            SocketChannel clientChannel = (SocketChannel) key.channel();
            clientChannel.read(buffer);
            buffer.flip(); //从"用于读"变为"用于写"
            if (buffer.hasRemaining()) { //也可以注册到selector中
              clientChannel.write(buffer); //echo
            }
            buffer.clear(); //重复利用
          }
        }
      }
    }
  }
}
```


**在NioEchoServer类中，如果有n个客户端连接服务器，那么就会创建n+1个Channel，其中一个serverChannel用于接受客户端的连接，另外n个clientChannel用于与客户端进行通信。这n+1个Channel均注册到Selector中。Selector会间隔一定时间轮训这n+1个Channel，查找可连接、可读、可写的Channel，然后再进行连接、读取、写入操作。**



如上代码所示，大部分情况下，我们都不需要监听Channel是否可写，毕竟网络写入跟文件写入类似，大部分情况下都不需要等待。只有当写入出现问题时，比如write()函数返回0，表示网络拥塞，此时才需要如下代码所示，将Channel注册到Selector中，等待可写
```Java
clientChannel.register(selector, SelectionKey.OP_WRITE);
```

需要注意的是，并不是所有的Channel都可以注册到Selector中被监听，只有实现了SelectableChannel接口的Channel才可以，比如DatagramChannel、SocketChannel、ServerSocketChannel。FileChannel因为没有实现SelectableChannel，并且不支持非阻塞模式，所以，无法被Selector监听。


***多路复用I/O模型只需要一个线程即可，解决了阻塞I/O模型线程开销大的问题。不过，这种模型依然存在问题。如果某些clientChannel读写的数据量比较大，或者逻辑操作比较复杂，耗时比较久，因为所有的工作都在一个线程中完成，那么其他clientChannel便迟迟得不到处理，最终的效果就是，服务器响应客户端的延迟很大。***


**为了解决这个问题，我们可以引入线程池，对于Selector检测到有数据可读的clientChannel，我们从线程池中取线程来处理，而不是所有的clientChannel都在一个线程中处理。我们知道，阻塞I/O模型也用到了多线程，跟这里的区别在于，不管有没有数据可读，阻塞I/O模型中的每个clientSocket都会一直占用线程。而这里的多线程只会处理经过Selector筛选之后有可读数据的clientChannel，并且处理完之后就释放回线程池，线程的利用率更高。**


#### 3）异步I/O模型（AIO）
实际上，上述问题使用java.nio的异步Channel实现起来更加优雅。如下代码所示。通过异步Channel调用accept()、read()、write()函数。当有连接建立、数据读取完成、数据写入完成时，底层会通过线程池执行对应的回调函数。这种服务器的实现方式叫做异步I/O模型。
```Java
public class AioEchoServer {
  public static void main(String[] args) throws IOException, InterruptedException {
    AsynchronousServerSocketChannel serverChannel = AsynchronousServerSocketChannel.open();
    serverChannel.bind(new InetSocketAddress("127.0.0.1", 1192));
    // 异步accept()
    serverChannel.accept(null, new AcceptCompletionHandler(serverChannel));
    Thread.sleep(Integer.MAX_VALUE);
  }

  private static class AcceptCompletionHandler
      implements CompletionHandler<AsynchronousSocketChannel, Object> {
    private AsynchronousServerSocketChannel serverChannel;
    public AcceptCompletionHandler(AsynchronousServerSocketChannel serverChannel) {
      this.serverChannel = serverChannel; 
    }
    
    @Override
    public void completed(AsynchronousSocketChannel clientChannel, Object attachment) {
      // in order to accept other client's connections
      serverChannel.accept(attachment, this);
      ByteBuffer buffer = ByteBuffer.allocate(1024);
      // 异步read()
      clientChannel.read(buffer, buffer, new ReadCompletionHandler(clientChannel)); 
    }

    @Override
    public void failed(Throwable exc, Object attachment) {
      // log exc exception
    }
  }

  private static class ReadCompletionHandler 
      implements CompletionHandler<Integer, ByteBuffer> {
    private AsynchronousSocketChannel clientChannel;
    public ReadCompletionHandler(AsynchronousSocketChannel clientChannel) {
      this.clientChannel = clientChannel;
    }
    
    @Override
    public void completed(Integer result, ByteBuffer buffer) {
      buffer.flip();
      // 异步write()。回调函数为null，写入完成就不用回调了
      clientChannel.write(buffer, null, null); // echo
    }

    @Override
    public void failed(Throwable exc, ByteBuffer attachment) {
      // log exc exception
    }
  }
}
```


实际上，**在平时的开发中，我们一般不会直接使用底层的java.nio类库，而是使用Netty等框架来进行网络编程，这些框架封装了网络编程的复杂性，使用起来更加简单，开发效率更高。**除了以上三种常见的I/O模型之外，实际上，还有更多更加复杂的I/O模型，比如**Netty框架提供的Reactor模型。**关于Netty等网络编程知识，我们就不深入讲解了。毕竟专栏的重点不在这里。


你可能还听过其他I/O模型的分类，比如在《Unix网络编程》一书中，介绍了Unix操作系统的5种I/O模型：阻塞I/O模型、非阻塞I/O模型、多路复用I/O模型、信号驱动I/O模型、异步I/O模型。那么，Unix操作系统下的I/O模型跟Java I/O模型有什么联系呢？


**实际上，不同的操作系统会提供不同的I/O模型。Java是一种跨平台语言，为了屏蔽各个操作系统I/O模型的差异，设计了3种新的I/O模型：BIO（阻塞I/O）、NIO（非阻塞I/O）、AIO（异步I/O），并且提供了I/O类库来支持这3种I/O模型的代码实现。而Java的I/O类库底层需要依赖操作系统的I/O接口（专业名称为系统调用）来实现，因此，从本质上来讲，Java I/O模型只是对操作系统I/O模型的重新封装。**



## 三、对比java.io与java.nio
在功能上来看，java.nio完全可以替代java.io，那么，在平时开发中，我们是不是应该首选java.nio呢？在新开发的项目中，是不是就不应该使用老的java.io呢？


实际上，在某些情况下，我们确实必须使用java.nio，比如网络编程。尽管使用java.io，并配合java.net，也可以进行网络编程，但java.io只支持阻塞模式，只能实现阻塞I/O模型，对于大部分网络编程来说，都是不够的。而java.nio提供了非阻塞模式、Selector多路复用器、异步模式，能够实现更加高性能的网络模型，比如非阻塞I/O模型、异步I/O模型。相比java.io而言，在网络编程方面，java.nio的优势更加明显。


但是，在某些情况下，到底使用java.io还是java.nio，会有一些争论，比如文件读写。前面提到，文件读写只支持阻塞模式，因此，使用java.io或java.nio都可以。有些人认为，使用java.io进行文件读写，代码编写更加简单。有些人则认为，java.nio的文件读写功能更加丰富。我个人认为，既然有争论，就说明两者没有哪个更有绝对优势，不然也就不会有争论了。因此，对于使用java.io还是java.nio进行文件读写，按照你的喜好或者团队的编程习惯来选择就好。


**总结一下的话，对于网络编程，我们首选java.nio，对于文件读写，java.io和java.nio都可以。**


四、课后思考题
java.io提供了BufferedInputStream、BufferedOutputStream，用于支持缓存的文件读写，那么，类似功能，java.nio是如何实现的呢？