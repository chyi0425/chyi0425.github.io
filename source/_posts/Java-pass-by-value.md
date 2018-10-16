---
title: Java-pass-by-value
date: 2018-10-16 10:47:59
tags: [Java] #文章标签，多于一项时用这种格式
toc: true
---

### 基本类型和引用类型的不同之处

```Java
int num = 10;
String str = "Hello";
```

![示意图](/img/pass-by-value-1.jpg)

如图所示，num是基本类型，值就直接保存在变量中。而str是引用类型，变量中保持的只是实际对象的地址。一般称这种变量为**“引用”**，引用指向实际对象，实际对象中保存着内容。

### 赋值运算符(=) 的作用

```Java
num = 20;
str= "Java";
```

![示意图](/img/pass-by-value-2.jpg)

对于基本类型num，赋值运算符会直接改变变量的值，原来的值被覆盖掉。
对于引用类型str，赋值运算符会改变引用中所保存的地址，原来的地址被覆盖。**但是原来的对象不会被改变**。
如上图所示，"hello"字符串对象并没有被改变。

### 调用方法时发生了什么。
**参数传递基本上就是赋值操作。**

```Java
// 第一个例子：基本类型
void foo(int value){
    value = 100;
}
foo(num); // num没有被改变

// 第二个例子：没有提供改变自身方法的引用类型
void foo(String text){
    text = "windows";
}
foo(str); // str 也没有被改变

// 第三个例子：提供了改变自身方法的引用类型
StringBuilder sb = new StringBuilder("iphone");
void foo(StringBuilder builder){
    builder.append("4");
}
foo(sb); // sb 被改变了，变成了iphon4

// 第四个例子：提供了改变自身方法的引用类型，但是不使用，而是使用赋值运算符。
StringBuilder sb = new StringBuilder("iphone");
void foo(StringBuilder builder){
    builder = new StringBuilder("ipad");
}
foo(sb); // sb 没有改变，还是"iphone"
```

第三个例子和第四个例子为什么结果不同

下面是第三个例子的图解

![示意图](/img/pass-by-value-3.jpg)

builder.append("4")之后

![示意图](/img/pass-by-value-4.jpg)

下面是第四个例子的图解：

![示意图](/img/pass-by-value-5.jpg)

builder = new StringBuilder("ipad")之后

![示意图](/img/pass-by-value-6.jpg)

###　局部变量&&方法参数

局部变量和方法参数在JVM中存储的方法是相同的，都是在栈上开辟空间来存储的，随着进入方法开辟，退出方法回收。以32位JVM为例，boolean/byte/short/char/int/float以及引用都是分配4字节空间，long/double分配8字节空间。对于每个方法来说，最多占用多少空间是一定的，这在编译时就可以计算好。

在JVM内存模型中有stack和heap存在，更准确的说，每个线程分配一个独享的stack，所有线程共享一个heap。对于每个方法的局部变量来说，是绝对无法被其他方法，甚至其他线程的同一方法所访问到的，更遑论修改。

当我们在方法中声明一个int = 0,或者Object obj = null时，仅仅涉及到stack，不影响heap，当我们new Object()时，会在heap中开辟一段内存并初始化Object对象。当我们将这个对象赋予obj变量时，仅仅是stack中代表obj的那4个字节变更为这个对象的地址。

### 数组类型引用和对象

当我们声明一个数组时，如int[] arr = new int[10]，因为数组也是对象，arr实际上是引用，stack上仅仅占用4字节空间，new int[10]会在heap中开辟一个数组对象，然后arr指向它。

当我们声明一个二维数组时，如int[][] arr2 = new int[2][4],arr2同样仅在stack中占用4个字节，会在内存中开辟一个长度为2类型为int[]的数组，然后arr2指向这个数组。这个数组内部有两个引用(大小为4字节)，分别指向两个长度为4的类型为int的数组。

![示意图](/img/pass-by-value-7.jpg)

所以当我们传递一个数组引用给一个方法时，数组的元素是可以被改变的，但是无法让数组引用指向新数组。

你还可以这样声明：int[][] arr3 = new int[3][],这时内存情况如下图

![示意图](/img/pass-by-value-8.jpg)

你还可以这样arr3[0]= new int[5]; arr3[1] = arr2[0]

![示意图](/img/pass-by-value-9.jpg)

### 关于String

原本回答关于String的图解是简化过的，实际上String对象内部仅需要维护三个变量，char[] chars,int startIndex,int length。而chars在某些情况下是可以共用的。但是String被设计为不可变类型，所以你思考时把String对象简化考虑也是可以的

String str = new String("hello");

![示意图](/img/pass-by-value-10.jpg)

当然某些JVM实现会把"hello"字面量生成的String对象放到常量池中，而常量池中的对象可以实际分配在heap中，有些实现也许会分配在方法区，当然这对我们的理解影响不大。