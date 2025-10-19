---
title: Java IO 相关类解析
link:
date: 2017-08-16 21:57:58
tags:
---

## OutputStream

Java.io.OutputStream类声明了三个基本方法用来把byte数据写入到流中。当然也有用于关闭和刷新的流

```java
  public abstract void write(int b) throws IOException
  public void write(byte[] data) throws IOException
  public void write(byte[] data, int offset, int length) throws IOException
  public void flush() throws IOException
  public void close() throws IOException
```

OutputStreams是一个抽象类，子类提供方法的实现。大多数情况下，你只需要知道你处理的对象是一个OutputStream就足够了。
OutputStream中最基本的方法是write()

  public abstract void write(int b) throws IOException
这个方法书写了一个无符号byte（0-255），如果你传入了大于255或者小于0的数值，会对256取模直到得到一个合适的值。
通常来说，对大量的数据，用byte来传递会更快一些。这正是两个write方法的用途
第一个写入整个byte数组。第二个只写入数组的一部分，从offset开始写入length长度的数据。
相反地，如果你尝试一次性写入太多的数据，性能上就会出现问题。文件最好一次一次地写入小的块，典型地数值像512，1024，2048.网络连接通常只需要更小的块，128或者256.
输出流缓冲区用来提高性能。比起把每一个字节送到它想去的终点，字节们先在内存缓冲区中集合。当缓冲区被填满，数据被传送出去。flush方法强迫缓冲区没有满的时候输出。
如果你只使用流很短的时间，你不需要明确地调用flush方法。它应该在流关闭的时候被flush。一旦你关闭了流，你就不能再向其中写入数据，如果你尝试这么做，就会引起IOException.

## InputStream

Java.io.InputStream类声明了三个基本方法用来把byte数据写入到流中。当然也有用于关闭和刷新的流,查看还有多少数据可以读，略过一些输入，在流中标记一个位置然后重置到那个位置，还有决定标记和重设是否是支持的。
```
  public abstract int read() throws IOException
  public int read(byte[] data) throws IOException
  public int read(byte[] data, int offset, int length) throws IOException
  public long skip(long n) throws IOException
  public int available() throws IOException
  public void close() throws IOException
  public synchronized void mark(int readLimit)
  public synchronized void reset() throws IOException
  public boolean markSupported()
```
  InputStream中最基本的方法是read,这个方法读入一个无符号的byte类型，然后返回它的整型值。就像大多数的IO方法一样，read方法也会有异常抛出，如果read中无数据可读，你不会受到异常，而是返回-1。用这个作为流结尾的标志。如下的代码展示了如何catch IOException和查看是否为结尾。
```
    try{
        int[] data = new int[10];
        for(int i=0;i<data.length;i++) {
            int datum = System.in.read();
            if (datum == -1) break;
            data[i] = datum;
        }
    } catch (IOException e ) {
        System.err.println("Couldn't read from System.in!");
    }
```
  read方法等待或者阻塞直到byte数据可用而且准备就绪。Input和Output可能会很慢，所以如果你的程序在执行其他重要的事情。你应该把IO操作放在它们自己的线程当中。下面这个类
```
public class StreamPrinter {
  InputStream theInput;
  
  public static void main(String[] args) {
    StreamPrinter sr = new StreamPrinter(System.in);
    sr.print();
  }

  public StreamPrinter(InputStream in) {
    theInput = in;
  }
  
  public void print() {
    try{
      while(true) {
        int datum = theInput.read();
        if (datum == -1) break;
        System.out.println(datum);
      }
    } catch (IOException e) {
      System.err.println("couldn't read from system in")
   }
  }
}
```
  第一个read方法读入一批连续数据到byte数组中，第二个尝试读入一定长度的data从offset开始到byte数组。它们两个都不保证读入任意数量的byte。
  如果你打算从System.in读入10byte的数据，如下的代码可以完成操作：
```
  byte[] b = new byte[10];
  System.in.read(b);
```
  但是，并不是每次read都可以拿到你想要的那么多数据。但是这行代码也不能阻止你试图往read中写入超过容量的数据，如果你这么做了，就会导致ArrayIndexOutOfBoundsException.
  如下的代码利用循环，确保尽可能多得获得数据：
```
  byte[] b = new byte[100];
  int offset = 0;
  while(offset <b.length) {
    int bytesRead = System.in.read(b, offset,b.length-offset);
    if(bytesRead==-1) break;
    offset+=bytesRead;
  }
```
  尽管上述的代码可以尽可能多得获取数据，但是并不能规避异常的发生。所以，如果在你尝试读它们之前，你可以知道有多少数据将要被读，这将会非常方便。InputStream中的available方法可以告诉你答案
  你可以手动操作代码来忽略掉一部分的数据，但JAVA还是提供了skip方法用来跳过给定byte数的方法
```
  public long skip(long bytesToSkip) throws IOException
```
  返回值是实际略去的byte数。如果返回-1，则证明剩下的部分都被忽略了。通常来说skip方法比自己手动忽略要快。
  并不是所有的流都需要被关闭，比如System.in。但是跟文件或者网络相关的连接还是需要被关闭的。

## FileInputStream

java.io.FileInputStream是InputStream的具体实现，提供具体文件的输入流
```
  public class FileInputStream extends InputStream
```
  FileInputStream 实现了InputStream的常用方法
```
  public int read() throws IOException
  public int read(byte[] data) throws IOException
  public int read(byte[] data, int offset, int length) throws IOException
  public native long skip(long n) throws IOException
  public native int available() throws IOException
  public native void close() throws IOException
```
  这些方法都是Java Native Code，除了read方法，但这些方法还是把参数传给了native方法。所以实际上，这些方法都是Native方法。
  FileInputStream有三种构造方法，区别在于文件是如何指定的：
```
  public FileInputStream(String fileName) throws IOException
  public FileInputStream(File file) throws FileNotFoundException
  public FileInputStream(FileDescriptor fdObj)
```
  第一个构造函数使用文件的名称，文件的名称跟平台相关，所以硬编码文件名不是一个好的方案，相比之下，后两个就要好很多。
  读取文件，我们只需要把文件名称传递给构造函数。然后像平常那样调用read方法即可。
```
  FileInputStream fis = new FileInputStream("README.TXT");
  int n;
  while ((n=fis.available())>0) {
    byte[] b = new byte[n];
    int result = fis.read(b);
    if( result == -1) break;
    String s = new String(b);
    System.out.print(s);
   }
```
  Java在当前的工作路径寻找文件，通常来说，就是你键入java programName时的路径。在FileInputStream的构造函数中传入相对路径和绝对路径都是可行的。
  如果你试图打开一个并不存在的文件，就会抛出FileNotFoundException。如果因为其他原因无法写入（比如权限不足）其他类型的异常会被抛出。下面是一个通过控制台获取文件名，然后把文件打印到控制台的例子
```
public class FileTyper {
  public static void main(String[] args) {
    if(args.length==0) {
      System.err.println("no file is determined");
      return;
    }
    for (int i=0;i<args.length;i++) {
      try{
        typeFile(args[i]);
        if(i+1<args.length) {
          System.out.println();
          System.out.println("--------");
        }
       } catch (IOException e) {System.err.println(e);} 
     }
  }
  public static void typeFile(String filename) throws IOException {
    FileInputStream fin = new FileInputStream(filename);
    StreamCopier.copy(fin,System.out);
    fin.close();
  }
}
```
  如果需要的话，你也可以对同一个文件同时打开多个流。每一个流维护一个单独的指针，指向文件中的当前位置。读取文件并不会更改文件，如果是写文件的话，那就是另一回事了。

## FileOutputStream

  java.io.FileOutputStream是java.io.OutputStream的具体实现,提供连接到文件的输出流。
```
public class FileOutputStream extends OutputStream
```
  类中实现了OutputStream的所有常用方法
```
public native void write(int b) throws IOException
public void write(byte[] data) throws IOException
public void write(byte[] data, int offset, int length) throws IOException
public native void close() throws IOException
```
  跟之前的FileInputStream相同，FileOutputStream的四个方法也都是实际上的native方法。如下的三种构造器，区别在于文件是如何指定的:
```
public FileOutputStream(String filename) throws IOException
public FileOutputStream(File file) throws IOException
public FileOutputStream(FileDescriptor fd)
```
  和FileInputStream不同的是，如果指定的文件不存在，那么FileOutputStream会创建它，如果文件存在，FileOutputStream会覆盖它。这个特性让我们使用的时候有些不太方便，有的时候，往往我们需要往一个文件里面添加一些数据，比如向日志文件里面存储记录。这时候，第四个构造函数就体现了它的作用
```
  public FileOutputStream(String name, boolean append) throws IOException
```
  如果append设置为true，那么如果文件存在，FileOutputStream会向文件的末尾追加数据，而不是覆盖。
  下面的程序获取两个文件名作为参数，然后把第一个文件复制到第二个文件：
```
public class FileCopier {
  public static void main(String[] args) {
    if(args!=2) {
      System.err.println("error input");//输入异常。
      return;
    }
    try {
      copy(args[0],args[1]);//调用复制文件的方法
    } catch (IOException e) {System.err.println(e);}
  }
  public static void copy (String inFile, String outFile) throws IOException {
    FileInputStream fin = null;
    FileOutputStream fout = null;
    try{
      fin = new FileInputStream(inFile);
      fout = new FileOutputStream(outFile);
      StreamCopier.copy(fin,fout);
    }
    finally {
      try {
        if (fin != null) fin.close();
      } catch (IOException e) {}
      try {
        if (fout != null) fout.close();
      } catch (IOException e) {}
  }
}
public class StreamCopier {
  public static void copy(InputStream in,OutputStream out) throws IOException {
    //Do not allow other threads read from the input or write the output
    synchronized (in) {
      synchronized (out) {
        byte[] buffer = new byte[256];
        while(true) {
          int bytesRead = in.read(buffer);
          if(bytesRead == -1) break;
          out.write(buffer,0,bytesRead);
        }
      }
  }
}
```

## URLS

java.net.URL类是标准资源定位符。每一个URL明确地指定了因特网上一个资源的位置。URL有四个构造函数，每一个都声明了MalformedURLException
```
  public URL(String u) throws MalformedURLException
  public URL(String protocol, String host, String file) throws MalformedURLException
  public URL(String protocol, String host, int port, String file) throws MalformedURLException
  public URL(URL context, String u) throws MalformedURLException
```
  如果构造器没有给定一个URL，MalformedURLException会被抛出。如果给你一个绝对的URL比如"http://www.jianshu.com/u/9e21abacd418",你会这样构造一个URL对象:
```
  URL u = null;
  try {
    u = new URL("http://www.jianshu.com/u/9e21abacd418");
  } catch (MalformedURLException e)　{}
```
  你也可以把协议，host和路径分开传入
```
  URL u = null;
  try {
    u = new URL("http","www.jianshu.com","/u/9e21abacd418");
  } catch (MalformedURLException e)　{}
```
  一般情况下，你不需要特地指定协议的端口，大多数协议有他们默认的端口，比如HTTP的协议的默认端口是80.如果端口改变了，可以使用下面的构造方法:
```
    u = new URL("http","www.jianshu.com",8080,"/u/9e21abacd418");
```
  一旦URL对象被构造，有两种方式获得它的内容。openStream()方法返回原始的数据流，getContent()方法返回一个对象代表数据。当你调用getContent()方法的时候，JAVA根据它的MIME类型，寻找一个content handler，然后返回一个可用的数据对象。
  openStream()方法和URL代表的服务器和端口建立了一个Socket连接，返回一个可以获取数据的InputStream，允许你从服务器上下载数据。所有的头文件，跟数据无关的东西在流打开的时候都被跳过了。
```
  public final InputStream openStream() throws IOException
```
  使用reader或者InputStream来获取数据:
```
try {
  URL u = new URL("http://www.amnesty.org/");
  InputStream in = u.openStream();
  int b;
  while ((b = in.read()) != -1) {
    System.out.write(b);
  }
 }
catch (MalformedURLException e) {System.err.println(e);}  
catch (IOException e) {System.err.println(e);}
```

## UrlConnection

URL Connection和URL有着密切的联系，就像名字一样。你通过URL的openConnection()方法得到一个URL Connection的引用。
在大多数情况下，URL只是对URL Connection对象的一种封装。然而URL提供了更多的控制。

  URL Connection不仅仅提供了让客户端读取服务器上信息的能力，而且提供了OutputStream使得，客户端的文件可以发送向服务器。

  java.net.URLConnection类是一个处理多种不同类型服务器的抽象类，比如FTP服务器和web服务器。
  一.从URL Connection中读取数据
  1.构造URL对象
  2.通过openConnection()方法创建一个URLConnection对象
  3.连接的参数和需要的属性已经设置完毕
  4.使用connect()方法建立连接，可能是使用socket的网络连接，也可能是文件读入流的本地连接。响应的头部信息从服务器传入。
  5.使用InputStream来读取数据，或者使用相应(MIME 类型)content handler的getContent()方法。
  举个如下的例子:
```
public class Main {
    public static void main(String[] args) throws IOException {
        URL url = new URL("http://www.huawei.com");
        URLConnection uc = url.openConnection();
        uc.connect();
        InputStream in = uc.getInputStream();
        //...after operation
        //close the stream
        in.close();
    }
}
```
  如果连接无法被建立，会抛出一个IOException。

  二.向URL中写入数据
    1.构造URL对象
    2.通过openConnection()方法创建一个URLConnection对象
    3.调用setDoOutput(boolean doOutput)方法并传入true表明这个连接会被用于写入数据
    4.如果你仍然想从InputStream中读取数据，调用setDoInput(boolean doInput)方法并传入true表明这个连接会被用于读取数据
    5.创建你想要写入的数据
    6.调用getOutputStream拿到OutputStream对象。把第5步中的数据写入其中
    7.关闭输出流

  下面是一个例子:
```
public class MailClient {
  public static void main(String[] args) {
    if (args.length == 0) {
      System.err.println("Usage: java MailClient username@host.com");
      return;
    }
    try {
      URL u = new URL("mailto:" + args[0]);
      URLConnection uc = u.openConnection();
      uc.setDoOutput(true);
      uc.connect();
      OutputStream out = uc.getOutputStream();
      StreamCopier.copy(System.in, out);
      out.close();
     }
    catch (IOException e) {System.err.println(e);}
  }
}
```

## Sockets

在数据在互联网中从一个主机到另一个主机的传递之时，它被分割成大小不同但是有限的数据包中(datagrams)。如果要发送的数据大于了数据包的最大大小，它就会被分割成数个包发送，这样做的好处是，如果其中有一个包丢失，那么只需要重传一个包，而不必把所有的包重传。如果包抵达的顺序不同，也会在接收点重新组转完毕。

这一操作对程序员来说是透明的，我们工作在高层抽象的socket上。socket提供了两个主机之间可靠地连接。这样子，你就不需要考虑数据包的编码， 数据包的分割，数据包的重传或者是数据包的组装。Scoket提供这四种基本操作:


1.远程连接到一个机器
2.发送数据
3.接收数据
4.关闭连接

java.net.socket是一个network socket提供了这四种基本操作。在这四种操作中，没有一个抽象了协议，这个类就是为了网络客户端和服务器的连接设计的。为了创建一个连接，你调用socket构造函数中的一种。每一个socket对象仅仅连接到一个指定的远程主机。如果要连接到不同的主机，你必须创建一个新的socket对象：
```
  public Socket (String host, int port) throws UnknownHostException, IOException
  public Socket(InteAddress address, int port) throws IOException
  public Socket(String host, int port, InetAddress localAddr, int localPort) throws IOException
  public Socket(InetAddress address, int port, InetAddress localAddr, int localPort) throws IOException
```

host可以是像“www.huawei.com"这样子的，或者127.0.9.1这样。这个参数也可以通过java.net.InetAddress传入。

port参数指的是远程主机要连接的端口，0～65535.每一个服务都有他们指定的端口。许多知名的服务都运行在知名的端口上。比如HTTP运行在80端口中。

通过socket来发送接收数据是通过InputStream和 OuputStream
```
  public InputStream getInputStream() throws IOException
  public OutputStream getOutputStream() throws IOException
```
同样的，也有关闭socket连接的方法
```
  public synchronized void close() throws IOException
```

如下的代码连接到一个网络服务器然后下载一个特定的URL地址。然而这里使用的是Socket而不是URL Connection，所以头部信息需要我们自己处理。

```
public class SocketTyper {

    public static void main(String[] args) {

        if(args.length==0) {
            System.err.println("Usage: java SocketTyper url1 url2..");
            return;
        }

        for(int i=0;i<args.length;i++) {
            if(args.length<1) {
                System.out.println(args[i]+":");
            }
            try {
                URL u = new URL(args[i]);
                if(u.getProtocol().equalsIgnoreCase("http")){
                    System.err.println("ONLY support http");
                    continue;
                }
                String host = u.getHost();
                int port = u.getPort();
                String file = u.getFile();
                if(port<=0||port>65535) {
                    port = 80;
                }
                Socket s = new Socket(host,port);
                String request = "GET"+file+"HTTP/1.0\r\n"+"User-Agent:MechaMozilla\r\nAccept:text*\r\n\r\n";
                byte[] b = request.getBytes();
                OutputStream out = s.getOutputStream();
                InputStream in = s.getInputStream();
                out.write(b);
                out.flush();
                StreamCopier.copy(in,System.out);
                in.close();
                out.close();
                s.close();
            } catch (MalformedURLException e){

            } catch (IOException e) {
                
            }
        }

    }

}
```

## ServerSocket

两种连接终端，客户端初始化连接，还有服务端，响应连接。实现一个服务器，你需要书写一个等待其他主机连接的程序。一个ServerSocket连接到本机的一个特定端口，一旦它顺利地绑定到了一个端口上，如果监听到了来自其他主机(客户端)的请求，就会建立连接。

一个端口同时可以连接多个客户端。传递来的数据会根据客户端的ip和端口来区分，ip和端口的组合是唯一的。有且只能有一个客户端监听同一主机上的同一端口。通常情况下，ServerSocket只用来接收连接，而和客户端的通信放在独立的线程里去处理。而把将要建立的连接放在队列里面逐个处理。

java.net.ServerSocket类代表着一个ServerSocket。有如下的构造函数，可以确定监听的端口，队列的长度以及ip地址（默认为本机）

```
  public ServerSocket (int port) throws IOException
  public ServerSocket (int port, int backlog) throws IOException
  public ServerSocket (int port, int backlog, InetAddress bindAddr) throws IOException
```

通常情况下你指定你想要监听的端口
```
        try {
            ServerSocket ss = new ServerSocket(99);
        } catch (IOException e) {
            e.printStackTrace();
        }
```

如果此时该端口已经被其他程序占用，就会导致BindException。当你拥有了ServerSocket之后，你需要等待连接，通过调用accept方法，这个方法会阻塞，直到一个连接到来，然后返回一个Socket，你可以使用它来和客户端通信，调用close方法会关闭ServerSocket

```
  public Socket accept() throws IOException
  public void close() throws IOException
```

下面的例子展示了一个程序，监听端口，当建立连接之时，它返回客户端和自身的ip和端口，然后关闭连接

```
public class HelloServer {

    public final static int DEFAULT_PORT = 2345;

    public static void main(String[] args) {
        int port = DEFAULT_PORT;

        try {
            port = Integer.parseInt(args[0]);
        } catch (Exception e) {}
        if(port<=0||port>=65536) {
            port = DEFAULT_PORT;
        }

        try {
            ServerSocket ss = new ServerSocket(port);
            while (true) {
                try{
                    Socket s = ss.accept();
                    String response = s.getInetAddress()+""+s.getPort()+"\n";
                    response += s.getLocalAddress()+""+s.getLocalPort();
                    OutputStream out = s.getOutputStream();
                    out.write(response.getBytes());
                    out.flush();
                    out.close();
                } catch (IOException e) {}
            }
        } catch (IOException e) {}
    }

}
```
