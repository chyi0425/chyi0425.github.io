---
title: Java-IO
date: 2018-07-18 15:26:15
tags: [Java,IO,NIO] #文章标签，多于一项时用这种格式
toc: true
---
## JAVA IO
### 分类
Java的I/O大概可以分为以下几类
* 磁盘操作：File
* 字节操作：InputStream和OutputStream
* 字符操作：Reader和Writer
* 对象操作：Serializable
* 网络操作：Socket
* 新的输入/输出：NIO

### 磁盘操作
File类可以用于表示文件和目录的信息，但是它不表示文件的内容。
递归输出一个目录下的所有文化：
```Java
    public static void listAllFiles(File dir) {
        if (dir == null || !dir.exists()) {
            return;
        }
        if (dir.isFile()) {
            System.out.println(dir.getName());
        }
        for (File file : dir.listFiles()) {
            listAllFiles(file);
        }
    }
```
### 字节操作
使用字节流操作进行文件复制：
```Java
    public static void copyFile(String src,String dist) throws IOException {
        FileInputStream in = new FileInputStream(src);
        FileOutputStream out = new FileOutputStream(dist);
        byte[] buffer = new byte[20*1024];
        // read() 最多读取buffer.length个字节
        // 返回的是实际读取的个数
        // 返回-1的时候表示对到eof，即文化尾
        while (in.read(buffer,0,buffer.length)!=-1){
            out.write(buffer);
        }
        in.close();
        out.close();
    }
```
![示意图](/img/DP-Decorator-java.io.png)
Java I/O使用了装饰器模式来实现。以InputStream为例，InputStream是抽象组件，FileInputStream是InputStream的子类，属于具体组件，提供了字节流的输入操作。FilterInputStream属于抽象装饰者，装饰者用于装饰组件，为组件提供额外的功能，例如BufferedInputStream为FileInputStream提供缓存的功能。
实现一个具有缓存功能的字节流对象时，只需要在FileInputStream对象上再套一层BufferedInputStream对象即可。
```Java
FileInputStream fileInputStream = new FileInputStream(filePath);
BufferedInputStream bufferedInputStream = new BufferedInputStream(fileInputStream);
```
DataInputStream 装饰者提供了对更多数据类型进行输入的操作，比如 int、double 等基本类型。
### 字符操作
不管是磁盘还是网络传输，最小的存储单元都是字节，而不是字符。但是在程序中操作的通常是字符形式的数据，因此需要提供对字符进行操作的方法。
* InputStreamReader 实现从字节流解码成字符流；
* OutputStreamWriter 实现字符流编码成为字节流。
逐行输出文本文件的内容：
```Java
    public static void readFileContent(String filePath) throws IOException {
        FileReader fileReader = new FileReader(filePath);
        BufferedReader bufferedReader = new BufferedReader(fileReader);
        String line;
        while ((line=bufferedReader.readLine())!=null){
            System.out.println(line);
        }
        // 装饰器模式使得BUfferedReader组合了一个Reader对象
        // 在调用bufferedReader的close()方法的时候回去调用fileReader的close()方法
        // 因此只要一个close()调用即可
        bufferedReader.close();
    }
```
编码就是把字符转换为字节，而解码是把字节重新组合成字符。

如果编码和解码过程使用不同的编码方式那么就出现了乱码。

* GBK 编码中，中文字符占 2 个字节，英文字符占 1 个字节；
* UTF-8 编码中，中文字符占 3 个字节，英文字符占 1 个字节；
* UTF-16be 编码中，中文字符和英文字符都占 2 个字节。

UTF-16be 中的 be 指的是 Big Endian，也就是大端。相应地也有 UTF-16le，le 指的是 Little Endian，也就是小端。

Java 使用双字节编码 UTF-16be，这不是指 Java 只支持这一种编码方式，而是说 char 这种类型使用 UTF-16be 进行编码。char 类型占 16 位，也就是两个字节，Java 使用这种双字节编码是为了让一个中文或者一个英文都能使用一个 char 来存储。

Java 使用双字节编码 UTF-16be，这不是指 Java 只支持这一种编码方式，而是说 char 这种类型使用 UTF-16be 进行编码。char 类型占 16 位，也就是两个字节，Java 使用这种双字节编码是为了让一个中文或者一个英文都能使用一个 char 来存储。

String 可以看成一个字符序列，可以指定一个编码方式将它转换为字节序列，也可以指定一个编码方式将一个字节序列转换为 String。

```Java
String str1 = "中文";
byte[] bytes = str1.getBytes("UTF-8");
String str2 = new String(bytes, "UTF-8");
System.out.println(str2);
```

在调用无参数 getBytes() 方法时，默认的编码方式不是 UTF-16be。双字节编码的好处是可以使用一个 char 存储中文和英文，而将 String 转为 bytes[] 字节数组就不再需要这个好处，因此也就不再需要双字节编码。getBytes() 的默认编码方式与平台有关，一般为 UTF-8。

```Java
byte[] bytes = str1.getBytes();
```

### 对象操作
序列化就是将一个对象转换成字节序列，方便存储和传输。
* 序列化：ObjectOutputStream.writeObject()
* 反序列化：ObjectInputStream.readObject()
序列化的类需要实现 Serializable 接口，它只是一个标准，没有任何方法需要实现，但是如果不去实现它的话而进行序列化，会抛出异常。
```Java
    public static void main(String[] args) throws IOException, ClassNotFoundException {
        A a1 = new A(123,"abc");
        String objectFile = "file/a1";
        ObjectOutputStream objectOutputStream = new ObjectOutputStream(new FileOutputStream(objectFile));
        objectOutputStream.writeObject(a1);
        objectOutputStream.close();

        ObjectInputStream objectInputStream = new ObjectInputStream(new FileInputStream(objectFile));
        A a2 = (A)objectInputStream.readObject();
        objectInputStream.close();
        System.out.println(a2);
    }
    private static class A implements Serializable {
        private int x;
        private String y;

        public A(int x, String y) {
            this.x = x;
            this.y = y;
        }

        @Override
        public String toString() {
            return "x = " + x + " y=" + y;
        }
    }
```
不会对静态变量进行序列化，因为序列化只是保存对象的状态，静态变量属于类的状态。

transient 关键字可以使一些属性不会被序列化。

ArrayList 序列化和反序列化的实现 ：ArrayList 中存储数据的数组是用 transient 修饰的，因为这个数组是动态扩展的，并不是所有的空间都被使用，因此就不需要所有的内容都被序列化。通过重写序列化和反序列化方法，使得可以只序列化数组中有内容的那部分数据。

```Java
private transient Object[] elementData;
```

### 网络操作
可以直接从URL中读取字节流数据
```Java
    public static void readFromURL() throws IOException {
        URL url = new URL("http://www.baidu.com");
        // 字节流
        InputStream is = url.openStream();
        // 字符流
        InputStreamReader isr = new InputStreamReader(is,"utf-8");
        BufferedReader br = new BufferedReader(isr);
        String line = br.readLine();
        while (line!=null){
            System.out.println(line);
            line = br.readLine();
        }
        br.close();
    }
```

### Sockets
* ServerSocket：服务器端类
* Socket：客户端类
* 服务端和客户端通过InputStream和OutputStream进行输入输出。

![示意图](/img/ClienteServidorSockets1521731145260.jpg)

### Datagram
* DatagramPacket：数据包类
* DatagramSocket：通信类

### NIO

#### Buffer
一个 Buffer 本质上是内存中的一块，我们可以将数据写入这块内存，之后从这块内存获取数据。

java.nio 定义了以下几个 Buffer 的实现
![Buffer实现](/img/nio-6-png)
核心是最后的 ByteBuffer，前面的一大串类只是包装了一下它而已，我们使用最多的通常也是 ByteBuffer。

我们应该将 Buffer 理解为一个数组，IntBuffer、CharBuffer、DoubleBuffer 等分别对应 int[]、char[]、double[] 等。

MappedByteBuffer 用于实现内存映射文件

操作 Buffer 和操作数组、类集差不多，只不过大部分时候我们都把它放到了 NIO 的场景里面来使用而已。下面介绍 Buffer 中的几个重要属性和几个重要方法。

##### position、limit、capacity
数组有数组容量，每次访问元素要指定下标，Buffer 中也有几个重要属性：position、limit、capacity。

![position,limit,capacity](/img/nio-5-png)
capacity，它代表这个缓冲区的容量，一旦设定就不可以更改。比如 capacity 为 1024 的 IntBuffer，代表其一次可以存放 1024 个 int 类型的值。一旦 Buffer 的容量达到 capacity，需要清空 Buffer，才能重新写入值。

position 和 limit 是变化的。