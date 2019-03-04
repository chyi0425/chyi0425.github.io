---
title: Java-enum
date: 2019-01-31 15:02:47
tags: [Java,enum]
toc: true
---

## 枚举的本质

我们先来写一个简单的枚举类：

```Java
public enum Fruit{
    APPLE(1),ORANGE(2),BANANA(3);
    int code;

    Fruit(int code){
        this.code=code;
    }
}
```

使用Jad命令反编译Fruit.class文件之后，可以得到如下的内容：

```Java
public final class Fruit extends Enum
{

    public static Fruit[] values()
    {
        return (Fruit[])$VALUES.clone();
    }

    public static Fruit valueOf(String s)
    {
        return (Fruit)Enum.valueOf(Fruit, s);
    }

    private Fruit(String s, int i, int j)
    {
        super(s, i);
        code = j;
    }

    public static final Fruit APPLE;
    public static final Fruit ORANGE;
    public static final Fruit BANANA;
    int code;
    private static final Fruit $VALUES[];

    static
    {
        APPLE = new Fruit("APPLE", 0, 1);
        ORANGE = new Fruit("ORANGE", 1, 2);
        BANANA = new Fruit("BANANA", 2, 3);
        $VALUES = (new Fruit[] {
            APPLE, ORANGE, BANANA
        });
    }
}
```

可见，Jvm编译器背地里是使用上面的方式来处理枚举的。它做了几件事：

* 定义一个继承自Enum类的Fruit类，Fruit类是用final修饰的
* 为每个枚举实例对应创建一个类对象，这些类对象是用public static final修饰的。同时生成一个数组，用于保存全部的类对象
* 生成一个静态代码块，用于初始化类对象和类对象数组
* 生成一个构造函数，构造函数包含自定义参数和两个默认参数（下文会讲解这两个默认参数）
* 生成一个静态的values()方法，用于返回所有的类对象
* 生成一个静态的valueOf()方法，根据name参数返回对应的类实例（下文会讲解name参数）

关于最基本的Enum类，Jdk文档是这样描述的：

> This is the common base class of all Java language enumeration types. More information about enums, including descriptions of the implicitly declared methods synthesized by the compiler, can be found in section 8.9 ofThe Java™ Language Specification.
> 这是Java语言中所有枚举类型的基础类，更多的关于枚举的信息，包括一些编译器隐式声明的方法的描述，可以在The Java™ Language Specification文档的8.9小节找到。

话不多说，来看源码：

```Java
public abstract class Enum<E extends java.lang.Enum<E>>
        implements Comparable<E>, Serializable {
    private final String name;
    private final int ordinal;
    
    public final String name() {
        return name;
    }
    
    public final int ordinal() {
        return ordinal;
    }
    
    protected Enum(String name, int ordinal) {
        this.name = name;
        this.ordinal = ordinal;
    }
    public String toString() {
        return name;
    }
    public final boolean equals(Object other) {
        return this == other;
    }
    public final int hashCode() {
        return super.hashCode();
    }
    protected final Object clone() throws CloneNotSupportedException {
        throw new CloneNotSupportedException();
    }

    public final int compareTo(E o) {
        java.lang.Enum other = (java.lang.Enum) o;
        java.lang.Enum self = this;
        if (self.getClass() != other.getClass() && // optimization
                self.getDeclaringClass() != other.getDeclaringClass())
            throw new ClassCastException();
        return self.ordinal - other.ordinal;
    }

    public final Class<E> getDeclaringClass() {
        Class clazz = getClass();
        Class zuper = clazz.getSuperclass();
        return (zuper == java.lang.Enum.class) ? clazz : zuper;
    }

    public static <T extends java.lang.Enum<T>> T valueOf(Class<T> enumType,
                                                          String name) {
        T result = enumType.enumConstantDirectory().get(name);
        if (result != null)
            return result;
        if (name == null)
            throw new NullPointerException("Name is null");
        throw new IllegalArgumentException(
                "No enum constant " + enumType.getCanonicalName() + "." + name);
    }

    protected final void finalize() {
    }

    private void readObject(ObjectInputStream in) throws IOException,
            ClassNotFoundException {
        throw new InvalidObjectException("can't deserialize enum");
    }

    private void readObjectNoData() throws ObjectStreamException {
        throw new InvalidObjectException("can't deserialize enum");
    }
}

```

在Enum源代码中，有以下几个值得关注的点：

* Enum类有两个成员变量：name和ordinal。其中，name用于记录枚举常量的名字。比如APPLE、ORANGE和BANANA。ordinal用于记录枚举常量在声明时的顺序(从0开始)。比如APPLE是0、ORANGE是1、BANANA是2。
* Enum类有一个构造函数，它有两个入参，分别为name和ordianl赋值。
* Enum类重写了toString()方法，返回枚举常量的name值。
* Enum类重写了equals()方法，直接用等号比较。
* Enum类不允许克隆，clone()方法直接抛出异常。（保证枚举永远是单例的）
* Enum类实现了Comparable接口，直接比较枚举常量的ordinal的值。
* Enum类有一个静态的valueOf()方法，可以根据枚举类型以及name返回对应的枚举常量。
* Enum类不允许反序列化，为了保证枚举永远是单例的。



```flow
st=>start: 开始
op=>operation: My Operation
cond=>condition: Yes or No?
e=>end
st->op->cond
cond(yes)->e
cond(no)->op
&```