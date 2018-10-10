---
title: arraylist linkedlist
date: 2018-09-14 13:16:37
tags: [Java,Collection] #文章标签，多于一项时用这种格式
toc: true
---

Java中常用到ArrayList和LinkedList，各自的使用场景。要想清楚的明白他们的区别，那还是得从源码入手。

## List接口

先列出这几个重要的方法:

```Java
public interface List<E> extends Collection<E> {
    ...
    //增
    boolean add(E e); 
    void add(int index, E element);
    //查
    E get(int index);
    //改
    E set(int index, E element);
    //删
    E remove(int index);
    boolean remove(Object o);
    ...
}
```

## ArrayList

### 全局变量
```Java
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    private static final long serialVersionUID = 8683452581122892189L;

    /**
     * Default initial capacity.
     */
    private static final int DEFAULT_CAPACITY = 10;

    /**
     * Shared empty array instance used for empty instances.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * The array buffer into which the elements of the ArrayList are stored.
     * The capacity of the ArrayList is the length of this array buffer. Any
     * empty ArrayList with elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA
     * will be expanded to DEFAULT_CAPACITY when the first element is added.
     */
    transient Object[] elementData; // non-private to simplify nested class access

    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;

    /**
     * The maximum size of array to allocate.
     * Some VMs reserve some header words in an array.
     * Attempts to allocate larger arrays may result in
     * OutOfMemoryError: Requested array size exceeds VM limit
     */
    private static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
        }
```

我们看到 transient Object[] elementData; 前面有个修饰符 transient，transient标识之后是不被序列化 。 
elementData 就是ArrayList的存储容器，为什么不能被序列化？
我们在看ArrayList源码中可以看到这个2个方法
ArrayList在序列化的时候会调用writeObject()，直接将size和element写入ObjectOutputStream；反序列化时调用readObject()，从ObjectInputStream获取size和element，再恢复到elementData。
主要是从效率考虑，elementData是一个缓存数组，一般都是会比实际存储大一些，它通常会预留一些容量，等容量不足时再扩充容量（当前容量的1.5倍），那么有些空间可能就没有实际存储元素，采用上面的方式来实现序列化时，就可以保证只序列化实际存储的那些元素，而不是整个数组，从而节省空间和时间。

```Java
    /**
     * Save the state of the <tt>ArrayList</tt> instance to a stream (that
     * is, serialize it).
     *
     * @serialData The length of the array backing the <tt>ArrayList</tt>
     *             instance is emitted (int), followed by all of its elements
     *             (each an <tt>Object</tt>) in the proper order.
     */
    private void writeObject(java.io.ObjectOutputStream s)
        throws java.io.IOException{
        // Write out element count, and any hidden stuff
        int expectedModCount = modCount;
        s.defaultWriteObject();

        // Write out size as capacity for behavioural compatibility with clone()
        s.writeInt(size);

        // Write out all elements in the proper order.
        for (int i=0; i<size; i++) {
            s.writeObject(elementData[i]);
        }

        if (modCount != expectedModCount) {
            throw new ConcurrentModificationException();
        }
    }
    /**
     * Reconstitute the <tt>ArrayList</tt> instance from a stream (that is,
     * deserialize it).
     */
    private void readObject(java.io.ObjectInputStream s)
        throws java.io.IOException, ClassNotFoundException {
        elementData = EMPTY_ELEMENTDATA;

        // Read in size, and any hidden stuff
        s.defaultReadObject();

        // Read in capacity
        s.readInt(); // ignored

        if (size > 0) {
            // be like clone(), allocate array based upon size not capacity
            ensureCapacityInternal(size);

            Object[] a = elementData;
            // Read in all elements in the proper order.
            for (int i=0; i<size; i++) {
                a[i] = s.readObject();
            }
        }
    }

```


### 构造函数

ArrayList底层使用的是动态数组，我们常用到的构造方法一般是如下两种:

```Java
    /**
     * Shared empty array instance used for empty instances.
     */
    private static final Object[] EMPTY_ELEMENTDATA = {};

    /**
     * Shared empty array instance used for default sized empty instances. We
     * distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when
     * first element is added.
     */
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    /**
     * Constructs an empty list with an initial capacity of ten.
     */
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA;
    }

    /**
     * Constructs an empty list with the specified initial capacity.
     *
     * @param  initialCapacity  the initial capacity of the list
     * @throws IllegalArgumentException if the specified initial capacity
     *         is negative
     */
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: "+
                                               initialCapacity);
        }
    }

    /**
     * Constructs a list containing the elements of the specified
     * collection, in the order they are returned by the collection's
     * iterator.
     *
     * @param c the collection whose elements are to be placed into this list
     * @throws NullPointerException if the specified collection is null
     */
    public ArrayList(Collection<? extends E> c) {
        elementData = c.toArray();
        if ((size = elementData.length) != 0) {
            // c.toArray might (incorrectly) not return Object[] (see 6260652)
            if (elementData.getClass() != Object[].class)
                elementData = Arrays.copyOf(elementData, size, Object[].class);
        } else {
            // replace with empty array.
            this.elementData = EMPTY_ELEMENTDATA;
        }
    }


```

从源码来看，区别就是前者给了个空数组，后者如果initialCapacity大于0，即给定一个initialCapacity大小的数组。

需要注意的是，无参的构造函数，空数组是**DEFAULTCAPACITY_EMPTY_ELEMENTDATA**,参数为0的空数组是**EMPTY_ELEMENTDATA**。

>Shared empty array instance used for default sized empty instances. We distinguish this from EMPTY_ELEMENTDATA to know how much to inflate when first element is added.

#### 参数为Collection的构造方法

1. 将collection对象转换成数组，然后将数组的地址的赋给elementData。
2. 更新size的值，同时判断size的大小，如果是size等于0，直接将空对象EMPTY_ELEMENTDATA的地址赋给elementData。
3. 如果size的值大于0，c.toArray 可能(错误地)不返回 Object[]类型的数组 参见 jdk 的 bug 列表(6260652),则执行Arrays.copy方法，把collection对象的内容copy到elementData中。

### add

#### add(E e)

```Java
    /**
     * The size of the ArrayList (the number of elements it contains).
     *
     * @serial
     */
    private int size;
    /**
     * Appends the specified element to the end of this list.
     *
     * @param e element to be appended to this list
     * @return <tt>true</tt> (as specified by {@link Collection#add})
     */
    public boolean add(E e) {
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        elementData[size++] = e;
        return true;
    }

    private void ensureCapacityInternal(int minCapacity) {
        if (elementData == DEFAULTCAPACITY_EMPTY_ELEMENTDATA) {
            // 进入此处表示构造方法是无参的构造方法 且minCapacity入参为1，最后minCapacity肯定是10
            minCapacity = Math.max(DEFAULT_CAPACITY, minCapacity);
        }

        ensureExplicitCapacity(minCapacity);
    }

    //判断是否需要扩容
    private void ensureExplicitCapacity(int minCapacity) {
        //列表结构被修改的次数,用于保证线程安全,如果在迭代的时候该值意外被修改,那么会报ConcurrentModificationException错
        modCount++;
        // 如果最小需要空间比elementData的内存空间要大，则需要扩容
        // overflow-conscious code
        if (minCapacity - elementData.length > 0)
            grow(minCapacity);
    }
    /**
     * Increases the capacity to ensure that it can hold at least the
     * number of elements specified by the minimum capacity argument.
     *
     * @param minCapacity the desired minimum capacity
     */
    private void grow(int minCapacity) {
        // overflow-conscious code
        // 1 记录之前的数组长度
        int oldCapacity = elementData.length;
        // 2. 新数组的大小=老数组大小+老数组大小的一半
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        // 3. 判断扩容之后的大小newCapacity是否能装够minCapacity个元素，不够就将数组长度设置为需要的长度
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;

        // 4. 判断新数组容量是否大于最大值 如果新数组容量比最大值(Integer.MAX_VALUE-8)还大，那么交给hugeCapacity()去处理，该抛异常就抛异常
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
        // minCapacity is usually close to size, so this is a win:
        // 复制数组
        elementData = Arrays.copyOf(elementData, newCapacity);
    }

    // 巨大容量
    private static int hugeCapacity(int minCapacity) {
        // 判断是否溢出
        if (minCapacity < 0) // overflow
            throw new OutOfMemoryError();
        return (minCapacity > MAX_ARRAY_SIZE) ?
            Integer.MAX_VALUE :
            MAX_ARRAY_SIZE;
    }
```
add的主要逻辑如下
1. 如果原数组为空，那么第一次添加元素给定默认大小10.
2. 修改次数modCount标致自增1，如果当前数组已使用长度size加一后大于当前的数组长度，则调用grow方法，增长数组，grow方法会将当前数组的长度变为原来容量的1.5倍。 
3. 确保新增的数据有地方存储之后，则将新元素添加到位于size的位置上。 
4. 返回添加成功布尔值。

#### add(int index, E element)

在ArrayList的index位置，添加元素element

```Java
    public void add(int index, E element) {
        // 判断是否越界
        rangeCheckForAdd(index);
        // 判断是否需要扩容
        ensureCapacityInternal(size + 1);  // Increments modCount!!
        // 将elementData从index位置开始，复制到elementData的index+1开始的连续空间
        System.arraycopy(elementData, index, elementData, index + 1,
                         size - index);
        // 在elementData的index位置赋值element
        elementData[index] = element;
        // 记录当前真实数据的个数
        size++;
    }

    //index不合法时,抛IndexOutOfBoundsException
    private void rangeCheckForAdd(int index) {
        if (index > size || index < 0)
            throw new IndexOutOfBoundsException(outOfBoundsMsg(index));
    }
```

1. 确保数插入的位置小于等于当前数组长度，并且不小于0，否则抛出异常 
2. 确保数组已使用长度（size）加1之后足够存下 下一个数据 
3. 修改次数（modCount）标识自增1，如果当前数组已使用长度（size）加1后的大于当前的数组长度，则调用grow方法，增长数组 
4. grow方法会将当前数组的长度变为原来容量的1.5倍。 
5. 确保有足够的容量之后，使用System.arraycopy 将需要插入的位置（index）后面的元素统统往后移动一位。 
6. 将新的数据内容存放到数组的指定位置（index）上

#### addAll(Collection<? extends E> c)

```Java
    public boolean addAll(Collection<? extends E> c) {
        // 生成一个包含集合c所有元素的数组
        Object[] a = c.toArray();
        // 记录需要插入的数组长度
        int numNew = a.length;
        // 判断一下是否需要扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount
        // 将a数组全部复制到elementData末尾处
        System.arraycopy(a, 0, elementData, size, numNew);
        // 标记当前elementData已有的元素个数
        size += numNew;
        // 是否插入成功
        return numNew != 0;
    }
```

#### addAll(int index, Collection<? extends E> c)

```Java
    public boolean addAll(int index, Collection<? extends E> c) {
        // 检查下标是否越界
        rangeCheckForAdd(index);

        // 生成包含集合c的所有元素的数组
        Object[] a = c.toArray();
        // 记录需要插入的数组长度
        int numNew = a.length;
        // 判断是否需要扩容
        ensureCapacityInternal(size + numNew);  // Increments modCount

        // 需要往后移动的元素个数
        int numMoved = size - index;
        if (numMoved > 0) // 后面有元素才需要复制，否则相当于插入到末尾
            // 将elementData的从index开始的numMoved个元素赋值到index+numNew处
            System.arraycopy(elementData, index, elementData, index + numNew,
                             numMoved);
        // 将a复制到elementData的index处
        System.arraycopy(a, 0, elementData, index, numNew);
        // 标记当前elementData已有元素的个数
        size += numNew;
        // 是否插入成功: c集合不为空就行
        return numNew != 0;
    }
```

#### 删除元素