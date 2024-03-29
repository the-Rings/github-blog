---
title: 19. io类库：java.io类库如此庞大，怎么才能全面系统的掌握它？
date: 2023-06-12 18:47:57
categories:
- 编程之美（王争）
---

相对于其他Java基础知识，大部分程序员对Java I/O可能没那么了解，毕竟平时的工作很少会编写I/O相关的代码，比如读写文件、网络编程等。跟Java容器类似，java.io类库，也非常庞大，如此多类看得眼花缭乱，想要清晰的掌握，需要对其有个系统性的认识。本节，我就带你一块剖析一下java.io类库，给你构建一个java.io类库的全景图。


## 一、io类库整体结构
I/O全称为Input/Output，中文为输入/输出。在计算机中，常用的I/O设备有硬盘、网络、键盘、显示器等，在操作系统层面，I/O系统有文件、网络、标准输入和输出（对应键盘和显示器）、管道等。Java提供的I/O类库就是用来读写这些I/O系统的。Java I/O类库主要有两个：java.io类库和java.nio类库。


在JDK1.4之前，Java引入了java.io类库。在JDK1.4中，**Java引入了java.nio类库，支持非阻塞I/O模型的开发。在JDK7中，Java对java.nio类库进行了升级，引入了更多的类，支持异步I/O模型的开发。**关于java.nio和I/O模型，我们下一节再讲。本节聚焦在java.io类库上。

java.io类库中包含的类非常多。在介绍Java容器时，我们画了一张比较复杂的类图，当时，我也提醒你说，一定要将它搞清楚。对于java.io类库，我们同样花了一张类图，如下图所示，也比较复杂，同理，你也要搞搞清楚。搞清楚了这张图，基本上也就掌握了java.io类库。

![java.io.png](1.png)



上图包含的类比较多，我们分类讲解。从不同的维度，java.io类库有不同的分类方式。我们按照不同的分类方式，拆解整个java.io类库，并逐一讲解。


## 二、输入流和输出流
按照**数据流向**来分类，java.io类库中类可以分为以下两类。

#### 1）输入流：InputStream、Reader

#### 2）输出流：OutputStream、Writer

**所谓输入流，指的是将文件、网络、标准输入（System.in）、管道中的数据，输入到内存中。所谓输出流，指的是将内存中的数据输出到文件、网络、标准输出（System.out、System.err）、管道中。**


输入流的读取方式，如下示例代码所示。**我们通过try-catch-resources语句打开InputStream，这样在try代码块结束后，JVM会自动调用InputStream的close()函数关闭输入流。**为了简化代码编写，本节后续示例代码均省略异常的捕获处理。
```Java
try (InputStream in = new FileInputStream("/Users/wangzheng/in.txt")) {
  byte[] data = new byte[1024];
  while (in.read(data) != -1) {
    //处理data数组...
  }
} catch (FileNotFoundException e) {
  e.printStackTrace();
} catch (IOException e) {
  e.printStackTrace();
}
```


输出流的写入方式，如下示例代码所示。
```Java
OutputStream out = new  FileOutputStream("/Users/wangzheng/out.txt")；
byte[] data = new byte[128];
out.write(data);
```


## 三、字节流和字符流
按照数据流的**读写单位**来分类，java.io类库中类可以分为以下两类。

#### 1）字节流：InputStream、OutputStream

#### 2）字符流：Reader、Writer

所谓字节流，指的是一个字节一个字节的从输入流中读取数据，或者将数据写入输出流。**所谓字符流，指的是一个字符一个字符的从输入流中读取数据，或者将数据写入输出流。实际上，字符流比起字节流来说，只是多了一个字符编码转换的环节。**我们拿文件读写来举例解释。



前面讲过，**Java中的char类型数据使用UTF16编码，而文件的编码方式有可能是UTF8、GBK等，所以，当从文件中读取数据到Java内存中的char数组时，我们需要将其从文件的编码方式转换为UTF16编码方式，同理，当我们将Java内存中的字符串写入到文件时，需要将UTF16编码转化为文件的编码方式。**示例代码如下所示。在写入完成之后，我们打开a.txt文件，查看文件内容的16进制格式，发现存储的是“王a争”这几个字符的UTF-8编码。
```Java
Writer writer = new FileWriter("/Users/wangzheng/a.txt");
String s = "王a争";
writer.write(s);
```

从java.io类图中，我们可以发现，Java分别为字符流和字节流设计了两套类。这两套类在代码实现上有些重复，毕竟I/O读写操作都是相同的，唯一的区别只是数据解析的方式不同。实际上，为了字节流和字符流设计两套类完全是没有必要的。java.nio利用“组合优于继承”的设计思想，引入Channel和Buffer的概念，对此设计进行了优化，关于这一点，我们在下节中会详细讲解。


## 四、原始类和装饰器类
**java.io类库的设计用到了设计模式中的装饰器模式，从这个角度，我们可以将java.io类库中的类分为原始类和装饰器类。**在《设计模式之美》一书中，我详细介绍了java.io类库如何使用装饰器模式简化类的设计，建议你去读一下。这里我们就不再赘述。装饰器类和原始类的区别在于，装饰器类是对原始类的功能增强，不能独立使用。**比如，FileInputStream为原始类，可以独立使用，BufferedInputStream为装饰器类，支持缓存功能，不能独立使用，必须嵌套原始类或其他装饰器类。**示例代码如下所示。
```Java
InputStream in = new FileInputStream("/Users/wangzheng/in.txt");
InputStream bin = new BufferedInputStream(in);
byte[] data = new byte[1024];
while (bin.read(data) != -1) {
    //处理data数组...
}
```


## 五、原始类分类介绍
对于原始类，我们还可以按照读写的I/O系统的不同，将其分为如下几类。注意这里不涉及装饰器类，因为装饰器类主要用于增强功能。

#### 1）文件
跟文件读写相关的类有FileInputStream、FileOutputStream、FileReader、FileWriter。前面已经给出了一些文件读写的示例代码。这里就不再赘述。

#### 2）网络
实际上，java.io类库并没有提供专门的类用于网络I/O的读写，而是直接复用InputStream类、OutputStream类进行网络I/O的读写。除此之外，单独使用java.io类库也并不能完成网络编程，需要借助java.net类库的配合。java.net类库用来管理网络连接，比如创建连接、关闭连接、监听连接等。java.io类库只负责读写已经建立的网络连接。示例代码如下所示。java.io类库在网络编程中的表现非常差劲，正因为如此，才有了java.nio类库的出现。
```Java
// Socket类位于java.net包中
Socket socket = new Socket("127.29.2.4", 8090);
OutputStream out = socket.getOutputStream();
out.write("hi~".getBytes());

InputStream in = socket.getInputStream();
byte[] data  = new byte[1024];
while (in.read(data) != - 1) {
  // do something
}
```

实际上，从java.io的类图中，我们也可以发现，InputStream、OutputStream是所有字节流类的父类，它既可以读写文件，也可以读写网络，还可以读写其他I/O。这充分体现了“抽象”的设计思想。尽管深入到硬件层面，各个I/O设备的读写方式各不相同，但是，上层应用开发并不关系底层实现细节，大部分I/O设备的访问都可以抽象为打开、读、写、关闭等这几个操作。因此，Java将所有的I/O设备都抽象为“Stream（流）”，并为不同I/O设备的读写设计了一套统一的接口。从而对于不同I/O设备的读写，我们可以使用同样的代码实现，代码更加统一、简洁。

#### 3）内存
跟内存读写相关的类有：ByteArrayInputStream、ByteArrayOutputStream、CharArrayReader、CharArrayWriter、StringReader、StringWriter。我们将内存看做一种特殊的I/O系统，也可以像文件一样，当作Stream来读写。

在大部分情况下，我们都不需要这些内存读写类，直接对byte数组、char数组进行读写即可，没必要将它们封装成流来操作。这些类的主要作用是实现兼容。比如，我们使用第三方类库中的某个函数，来处理byte数组中的数据，但这个函数的输入参数是InputStream类型的，为了兼容这个函数的定义，我们就可以将待处理的byte数组，封装成ByteArrayInputStream对象，再传递给这个函数处理，如下代码所示。
```Java
byte[] source = "学技术信小争哥就对了".getBytes();
InputStream in = new ByteArrayInputStream(source);
// 接下来就可以跟处理其他InputStream一样处理source了
```


在编写单元测试时，这些内存读写类也非常有用，可以替代文件或网络，将测试数据放置于内存，准备起来更加容易。如下代码所示，假设要为readFromFile()这个函数编写单元测试代码，我们需要创建文件，写入测试数据，并且放置到合适的地方，做一堆准备工作才能完成测试。如果使用ByteArrayInputStream，我们便可以在内存中构建测试数据，这样就方便了很多。
```Java
// 待测试函数
public int readFromFile(InputStream inputStream) { ... }

// 测试代码
public void test_readFromFile() {
  byte[] testData = new byte[512];
  //...构建测试数据，填入testData数组...
  InputStream in = new ByteIntputStream(testData);
  int res = readFromFile(in);
  //...assert...判断返回值是否符合预期...
}
```


#### 4）管道
跟管道读写相关的类有：PipedInputStream、PipedOutputStream、PipedReader、PipedWriter。**这里的管道跟Unix操作系统中的管道不同，Unix操作系统中的管道是进程间通信的工具，而这里的管道是Java提供的为同一个进程内两个线程之间通信的工具。一个线程通过PipedOutputStream写入的数据，另一个线程就可以通过PipedInputStream读取数据，示例代码如下所示。尽管Java已经提供了很多线程间通信的方式，比如常用的有共享变量，但是，一般来说，对于两个线程之间非对象的原始数据的传输，我们更倾向于使用管道来实现。**
```Java
PipedOutputStream out = new PipedOutputStream();
PipedInputStream in = new PipedInputStream(out);
new Thread(new Runnable() {
  @Override
  public void run() {
    try {
      out.write("Hi wangzheng~".getBytes());
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}).start();

new Thread(new Runnable() {
  @Override
  public void run() {
    byte[] buffer = new byte[512];
    try {
      in.read(buffer);
      System.out.println(new String(buffer));
    } catch (IOException e) {
      e.printStackTrace();
    }
  }
}).start();
```



#### 5）标准输入输出
**在操作系统中，一般会有三个标准I/O系统：标准输入、标准输出、标准错误输出。标准输入对应I/O设备中的键盘，标准输出和标准错误输出对应I/O设备中的屏幕。**Java中的标准输入为System.in，它是一个定义在System类中的静态InputStream对象。Java中的标准输出和标准错误输出分别为System.out和System.err，它们都是定义在System类中的PrintStream对象。PrintStream为装饰器类，需要嵌套OutputStream来使用，支持按照格式输出数据，待会会讲到。System.in、System.out、System.err的使用示例如下所示。
```Java
Scanner s = new Scanner(System.in);
System.out.println("echo: " + s.nextLine());
//System.err显示的字符串为红色，以表示出错
System.err.println("echo: " + s.nextLine());
```



## 六、装饰器类分类介绍
装饰器类用于增强原始类的功能。我们按照功能的不同分类讲解装饰器类。


#### 1）支持读写缓存功能的装饰器类
支持读写缓存功能的装饰器类有：BufferedInputStream、BufferedOutputStream、BufferedReader、BufferedWriter。这4个类的作用非常相似，我们拿BufferedInputStream、BufferedOutputStream举例讲解。

**对比InputStream，BufferedInputStream会在内存中维护一个8192字节大小的缓存。如果缓存中没有足够的数据，那么read()函数会向操作系统内核请求数据**（关于I/O读写的实现原理，我们在第21节中讲解），读取8192字节存储到缓存中，然后read()函数再从缓存中返回需要的数据量。如果缓存中有足够多的数据，read()函数直接从缓存中读取数据，不再请求操作系统。


**读取同样多的数据，利用BufferedInputStream，向操作系统内核请求数据的次数减少。我们知道，向操作系统内核请求数据，需要使用系统调用，引起用户态和内核态的切换，是非常耗时的，所以，尽量减少系统调用，会提高程序的性能**（关于这一部分内容的详细介绍，我们留在第21节中讲解）。**不过，如果read()函数每次请求的数据量都大于等于8192字节，那么BufferedInputStream就不起作用了。**


如下代码所示，如果文件中的数据大小是8192字节，那么，读取所有数据需要调用8次read()函数，但因为缓存的存在，所以仅需要向操作系统内核请求一次数据。

```Java
InputStream in = new FileInputStream("/Users/wangzheng/in.txt"));
InputStream bin = new BufferedInputStream(in);
byte[] data = new byte[1024];
while (bin.read(data) != -1) {
    //处理data数组...
}
```

同理，针对OutputStream，java.io类库提供了BufferedOutputStream，用来缓存写入的数据，当积攒到一定量（默认为8192字节）时，再一次性将其写入操作系统内核缓冲区，减少系统调用次数，提高程序的性能。




#### 2）支持基本类型数据读写的装饰器类

DataInputStream支持将从输入流中读取的数据解析为基本类型（byte、char、short、int、float、double等），DataOutputStream类支持将基本类型数据转化为字节数组写入输出流。示例代码如下所示。
```Java
DataOutputStream out = new DataOutputStream(new FileOutputStream("/Users/wangzheng/a.txt"));
out.writeInt(12);
out.writeChar('a');
out.writeFloat(12.12f);
out.close();

DataInputStream in = new DataInputStream(new FileInputStream("/Users/wangzheng/a.txt"));
System.out.println(in.readInt());
System.out.println(in.readChar());
System.out.println(in.readFloat());
in.close();
```

调用DataOutputStream的readChar()、writeChar()函数，我们也可以按字符为单位读取、写入数据，但跟字符流类不同的地方是，DataOutputStream类一次只能处理一个字符，而字符流类可以处理char数组，并且字符流类提供的函数更多，功能更加丰富。


#### 3）支持对象读写的装饰器类

ObjectInputStream支持将从输入流中读取的数据反序列化为对象，ObjectOutputStream支持将对象序列化之后写入到输出流。示例代码如下所示。
```Java
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("/Users/wangzheng/a.txt"));
out.writeObject(new Person(12, "wangzheng"));

ObjectInputStream in = new ObjectInputStream(new FileInputStream("/Users/wangzheng/a.txt"));
Person p = (Person) in.readObject();
System.out.println(p.getId());
System.out.println(p.getName());
```


#### 4）支持格式化打印数据的装饰器类

PrintStream和PrintWriter可以将数据按照一定的格式，转化为字符串，写入到输出流。前面讲到System.out、System.err就是PrintStream类型的。示例代码如下所示。
```Java
PrintStream printStream =new PrintStream(new FileOutputStream("/Users/wangzheng/a.txt"));
printStream.print(124); //int->Integer->toString(), 写入字符串"124"
printStream.printf("hello %d", 43); //写入字符串"hello 43"
```


除了以上装饰器类之外，还有一组原始类，其功能非常类似装饰器类，那就是InputStreamReader、OutputStreamWriter。InputStreamReader可以充当InputStream的装饰器类，OutputStreamWriter可以充当OutputStream的装饰器类。它们可以将字节流转化为字符流。示例代码如下所示。从这一点上，我们也可以看出，java.io类库的设计有很多不合理的地方，更晚开发的java.nio类库在设计上明显要合理很多，下节会详细讲到。
```Java
OutputStream outStream = new FileOutputStream("/Users/wangzheng/a.txt");
OutputStreamWriter writer = new OutputStreamWriter(outStream, "gbk");
writer.write("王a争"); //按照gbk编码将字符串写入文件
```


对于其他装饰器类，比如PushbackInputStream、PushbackReader、SequenceInputStream、LineNumberReader，因为使用的比较少，我们就不再介绍了，如果感兴趣，可以自行查阅。


七、课后思考题
在本节中，我们频繁提到“Stream（流）”这个字眼，比如输入流、输出流、字节流、字符流等，这里的“流”到底是什么意思？为什么把I/O看作“流”？