# Collection

> Java中的Collection由List、Set、Queue组成，这里把Map也放在当前目录下，是因为目录表示的集合而非Collection接口！在Java中Map是单独的接口哦

肯定要思考一个问题，什么情况下用什么集合

1. 需要key->元素值的操作时，使用Map
2. 需要保证元素唯一时选择Set
3. 需要实现栈机制、排队机制时，使用Queue
4. 需要元素有序的时候使用List，方便查询用数组，方便修改用链表

## List

> 别名顺序表，分为数组和链表两类
>
> Java中直接的实现类有ArrayList、LinkedList、Vector

Vector是早期实现类，线程安全，但是因为只能在尾部进行插入、删除操作，效率比较低

LinkedList是链表，在实现List接口的基础上实现Deque接口，效率实际上不高，实际场景List接口应用不如ArrayList，Deque接口应用不如ArrayDeque

### ArrayList

#### 简介

1. 实现RandomAccess接口，随机访问快人一步！
2. 实现Cloneable接口，可以被克隆
3. 实现Serializable接口，支持序列化，可以通过序列化去传输

通过数组实现，在初始化时如果不设置参数或者参数设置为0，实质上会得到空数组，但当第一次自动扩容时会扩容到10(10是ArrayList源码默认的初始大小)

实际存放有元素的ArrayList的大小下限为10上限为Interger.MAX_VALUE

#### 扩容机制

源码背景jdk8：

1. 调用add方法添加元素，方法内调用ensureCapacityInternal方法获取当前的最小扩容量
2. 最小扩容量的值其为之前的最小扩容量和默认最小初始大小10中的最大值，方法内会调用ensureExplicitCapacity方法判断是否需要扩容
3. 即判断当前size + 1会不会比最小扩容量大，如果扩容的话调用grow方法

`grow()`方法：扩容的关键

1. 新容量为旧容量的1.5倍左右
2. 判断新容量能否满足最小需要容量，不满足的话会重复的调用grow方法。(这个时候推荐手动扩容，在ensureCapacity方法中介绍)
3. 判断新容量是否大于最大容量，最大容量为Integer.MAX_VALUE - 8，ArrayList数组容量的理论最大值也只能是Integer.MAX_VALUE

```java
private void grow(int minCapacity) {
        // oldCapacity为旧容量，newCapacity为新容量
        int oldCapacity = elementData.length;
        //将oldCapacity 右移一位，其效果相当于oldCapacity /2
        //我们知道位运算的速度远远快于整除运算，整句运算式的结果就是将新容量更新为旧容量的1.5倍
        int newCapacity = oldCapacity + (oldCapacity >> 1);
        //然后检查新容量是否大于最小需要容量，若还是小于最小需要容量，那么就把最小需要容量当作数组的新容量
        if (newCapacity - minCapacity < 0)
            newCapacity = minCapacity;
       // 如果新容量大于 MAX_ARRAY_SIZE,进入(执行) `hugeCapacity()` 方法来比较 minCapacity 和 MAX_ARRAY_SIZE
       //如果minCapacity大于最大容量，则新容量则为`Integer.MAX_VALUE`，否则，新容量大小则为 MAX_ARRAY_SIZE 即为 `Integer.MAX_VALUE - 8`
        if (newCapacity - MAX_ARRAY_SIZE > 0)
            newCapacity = hugeCapacity(minCapacity);
    	// 调用Arrays.copyOf方法实现真正的扩容
        elementData = Arrays.copyOf(elementData, newCapacity);
    }
```

`ensureCapacity()`方法：手动扩容，避免重复调用grow方法

```java
public void ensureCapacity(int minCapacity) {
    int minExpand = (elementData != DEFAULTCAPACITY_EMPTY_ELEMENTDATA)
        // any size if not default element table
        ? 0
        // larger than default for default empty table. It's already
        // supposed to be at default size.
        : DEFAULT_CAPACITY;

    if (minCapacity > minExpand) {
        ensureExplicitCapacity(minCapacity);
    }
}
```

#### fail-fast机制

迭代器中实现checkForComodification方法，调用next方法时，如果modCount != expectedModCount，则会直接报出ConcurrentMOdificationException错误

注意expectedModCount是初始化时设置的定值，modCount则是动态的，是线程共享的声明volatile的变量，所以二者可能会不同！

### CopyOnWriteArrayList

#### 快照机制-读

为解决iterator迭代时，其它线程对List元素修改而导致的读取元素错误。内部实现COWIterator，实例化该迭代器时会生成一个final的snapshot数组，迭代器读取元素是固定不会更改(迭代器没有添加、移除、更新的功能)

#### 锁机制-写

使用ReentrantLock，在进行数据更新操作时获取锁。特殊方法中会比较当前数组与之前数组有无不同，比如addIfAbsent方法中就会比较当前数组与旧数组是否相同，如果不相同就会观察当前数组是否添加了目标值，添加了就直接释放锁。一般情况下会执行有关操作，最后再释放锁。

用到Arrays.copyOf方法！很关键

## Set

> 集合与数学定义中的集合有所类似，也就是不能有重复的元素

常用的有HashSet，是通过对HashMap直接进行封装得到的，简单来说就是只要了HashMap的Key，没要HashMap的Value

## Queue

> Queue是单端队列，也就是一端出另一端进的队列
>
> 在Queue的基础上还有Deque，即双端队列，在队列两端均可以插入或删除元素

常用的有ArrayDeque、LinkedList、PriorityDeque

前二者常常被用来作比较，二者的通性在于同样实现了Deque接口，二者的最主要的不同如下：

1. ArrayDeque是基于可变长数组和双指针实现的，LinkedList是基于链表实现
2. ArrayDeque不支持存储Null数据，LinkedList支持

实际上ArrayDeque实现队列的性能比LinkedList好

对于PriorityQueue而言，默认是小顶堆，可以通过重写Comparator中的reversed方法初始化为大顶堆。堆的概念在排序算法中已经捋过一遍，在源码中也是通过数组实现的。就是从叶子节点的根节点向上，确保每一个根节点都是当前子树中最大的，循环迭代就得到堆了

## Map

### HashMap

HashMap主要存放键值对，是非线程安全的嘞。默认初始化大小为16，每次扩容时容量变为原来2倍。这个2是有讲究嘚！HashMap使用2的幂作为哈希表的大小

#### 底层结构

##### JDK1.8前

数组+链表

HashMap通过key的hashcode经过扰动函数处理后得到hash值，`(n - 1) & hash`的值是当前元素存放的位置(n是数组长度)。如果当前位置存在元素，需要判断该元素hash值和要存入元素hash值(还有key)是否相同，相同就覆盖，不同通过拉链法解决冲突

简单介绍一下拉链法：如果有冲突的值创建一个链表，将值加到链表中就行

##### JDK1.8及之后

数组+链表+红黑树

首先hash值为key的哈希值的前16位与后16位异或的值+原来前16位的值，极大程度上保留原有数据的特性。其次考虑到链表长度过长会导致查询时间变慢，所以当链表长度大于8或自己设定的阈值时，会将链表转换为红黑树(注意：前提还要数组长度大于或等于64，否则优先考虑进行扩容)。但是如果红黑树的节点数小于等于6或设定阈值时，红黑树也会退化成链表

需要注意，理论上Map.Entry遍历是不会有顺序的，实现红黑树节点时引入的是LinkedHashMap.Entry

#### 扩容

loadfactor：存放元素的数组下标数/数组的长度的值即为装载因子，根据泊松分布的计算，一般设置当装载因子达到0.75f时需要进行扩容

threshold：capacity*loadFactor，当size大于等于threshold时就需要对数组进行扩容

进行扩容时，会进行重新hash分配，遍历hash表所有元素，尽量避免resize

```java
final Node<K,V>[] resize() {
    Node<K,V>[] oldTab = table;
    int oldCap = (oldTab == null) ? 0 : oldTab.length;
    int oldThr = threshold;
    int newCap, newThr = 0;
    if (oldCap > 0) {
        // 超过最大值就不再扩充了，就只好随你碰撞去吧
        if (oldCap >= MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return oldTab;
        }
        // 没超过最大值，就扩充为原来的2倍
        else if ((newCap = oldCap << 1) < MAXIMUM_CAPACITY && oldCap >= DEFAULT_INITIAL_CAPACITY)
            newThr = oldThr << 1; // double threshold
    }
    else if (oldThr > 0) // initial capacity was placed in threshold
        newCap = oldThr;
    else {
        // signifies using defaults
        newCap = DEFAULT_INITIAL_CAPACITY;
        newThr = (int)(DEFAULT_LOAD_FACTOR * DEFAULT_INITIAL_CAPACITY);
    }
    // 计算新的resize上限
    if (newThr == 0) {
        float ft = (float)newCap * loadFactor;
        newThr = (newCap < MAXIMUM_CAPACITY && ft < (float)MAXIMUM_CAPACITY ? (int)ft : Integer.MAX_VALUE);
    }
    threshold = newThr;
    @SuppressWarnings({"rawtypes","unchecked"})
        Node<K,V>[] newTab = (Node<K,V>[])new Node[newCap];
    table = newTab;
    if (oldTab != null) {
        // 把每个bucket都移动到新的buckets中
        for (int j = 0; j < oldCap; ++j) {
            Node<K,V> e;
            if ((e = oldTab[j]) != null) {
                oldTab[j] = null;
                if (e.next == null)
                    newTab[e.hash & (newCap - 1)] = e;
                else if (e instanceof TreeNode)
                    ((TreeNode<K,V>)e).split(this, newTab, j, oldCap);
                else {
                    Node<K,V> loHead = null, loTail = null;
                    Node<K,V> hiHead = null, hiTail = null;
                    Node<K,V> next;
                    do {
                        next = e.next;
                        // 原索引
                        if ((e.hash & oldCap) == 0) {
                            if (loTail == null)
                                loHead = e;
                            else
                                loTail.next = e;
                            loTail = e;
                        }
                        // 原索引+oldCap
                        else {
                            if (hiTail == null)
                                hiHead = e;
                            else
                                hiTail.next = e;
                            hiTail = e;
                        }
                    } while ((e = next) != null);
                    // 原索引放到bucket里
                    if (loTail != null) {
                        loTail.next = null;
                        newTab[j] = loHead;
                    }
                    // 原索引+oldCap放到bucket里
                    if (hiTail != null) {
                        hiTail.next = null;
                        newTab[j + oldCap] = hiHead;
                    }
                }
            }
        }
    }
    return newTab;
}

```

#### 线程不安全问题

头插法带来的问题：

主要问题在transfer函数中

HashMap的扩容操作（**先扩容在头插法插入**）会重新定位每个桶的下标，并采用头插法将元素迁移到新数组中。头插法会将链表的顺序翻转，这也是造成死循环和数据丢失的关键。

线程A的扩容方法执行到一半，由B来执行

尾插法带来的问题：

JDK1.8直接在resize函数中完成了数据迁移。在进行元素插入时使用的是**尾插法然后在扩容**。但是仍可能出现数据覆盖。

### ConcurrentHashMap

横向比较Hashtable，后者效率低下使用synchronized关键字对整个Hash加锁。ConcurrentHashMap的底层结构什么的和HashMap大差不差，主要讲如何实现并发机制

#### 并发机制

##### JDK1.7之前

ConcurrentHashMap由默认16个Segment组成，每个Segement都类似于一个HashMap。需要注意的是Segment个数一旦初始化就不能改变。对每个Segement都加锁，以一种分段锁的机制实现并发

扩容时会持有segment的独占锁(reenTrantLock)，不用担心并发问题。重点在于考虑在get的时候，同一个segment中发生put或remove操作

put操作保证线程安全：

1. 使用CAS初始化Segment的数组
2. 使用UNSAFE.putOrderedObject保证get方法可以读到到最新插入表头的节点
3. table属性使用volatile关键字，保证扩容时迁移数据时get方法不出错

remove方法是类似的，此处不深入

##### JDK1.8及之后

使用CAS和synchronized实现并发

初始化通过自旋和CAS操作完成，保证同时只有一个线程执行初始化方法

put方法使用synchronized，保证同时只有一个线程在添加/覆盖节点

# IO

## 简介

用户程序执行I/O操作必须通过**系统调用**，一般来说磁盘IO和网络IO是接触到更多的。**从应用程序的视角来看的话，我们的应用程序对操作系统的内核发起 IO 调用（系统调用），操作系统负责的内核执行具体的 IO 操作。也就是说，我们的应用程序实际上只是发起了 IO 操作的调用而已，具体 IO 的执行是由操作系统的内核来完成的。**

当应用程序发起 I/O 调用后，会经历两个步骤：

1. 内核等待 I/O 设备准备好数据
2. 内核将数据从内核空间拷贝到用户空间。

UNIX中常有5种模型：同步阻塞I/O、同步非阻塞I/O、I/O多路复用、信号驱动I/O、异步I/O。**Java中有BIO、NIO、AIO三种常见模型**

同步和异步的差距主要在于后者是通过通知的形式

阻塞和非阻塞就显而易见了，非阻塞表示在等待I/O前还能够做其它事情

## I/O多路复用

即I/O线程一次性处理多个I/O请求，Redis即同时监听多个socket，按照处理顺序将套接字放入一个队列中。

### epoll

`epoll`是Linux 特有的 I/O 多路复用函数，是对select和poll的增强。有没有最大并发连接的限制；只处理活跃的连接；使用共享内存，省略内存拷贝步骤的优势。本质上是通过Linux内核申请简易文件系统，将select/poll调用拆分成三个部分

- 调用epoll_create()建立一个epoll对象(在epoll文件系统中为这个句柄对象分配资源)
- 调用epoll_ctl向epoll对象中添加连接的套接字
- 调用epoll_wait收集发生的事件的连接（只处理活跃连接，不像之前会遍历所有连接）

epoll对象都有一个独立的**eventpoll结构体**，用于存放通过epoll_ctl方法向epoll对象中添加进来的事件。这些事件都会挂载在红黑树中，如此，重复添加的事件就可以通过红黑树而高效的识别出来

```c
struct eventpoll { 	
    //sys_epoll_wait用到的等待队列
    wait_queue_head_t wq;
    //接收就绪的描述符都会放到这里
    struct list_head rdllist;
    //每个epoll对象中都有一颗红黑树
    struct rb_root rbr;
    ......
}
```

而对于每一个事件，都会建立一个epitem结构体

```c
struct epitem { 	
    // 红黑树节点
    struct rb_node rbn;
    // 双向链表节点
    struct list_head rdllink;
    // 事件句柄信息
    struct epoll_filefd ffd;
    // 指向其所属的eventpoll对象
    struct eventpoll *ep;
    // 期待发生的事件类型
    struct epoll_event event;
}
```

# JVM

> 结合深入理解Java虚拟机，配合pdai.tech、javaGudie一同学习，非常好用

## 类字节码

### 目标

实现“一次编译，到处运行”运行的口号，计算机无法直接识别java语言，通过javac编译器将java语言编译成**.class(字节码)**之后，JVM虚拟机可以去读取执行。

### 实质

是以8位字节为基础单位的二进制流

### 结构属性

1. 魔数与class文件版本：很经典的咖啡宝贝(0xCAFEBABE)，标记当前文件为class文件
2. 常量池：存储有变量的属性，类型和名称，方法的属性、类型和名称等等
3. 访问标志：标记属性和访问类型
4. 索引：class文件通过类索引、父类索引、接口索引三项数据确定类的继承关系
5. 字段表属性：描述接口或类中声明的变量，比如变量作用域、是否是静态变量、可变性、数据类型等等
6. 方法表属性：与字段表类似，但是是方法表
7. 属性表属性：用于描述某些场景的专有信息。比如字段表中特殊属性、方法表中的特殊的属性等等

### 反编译工具

神奇的javap，用法: `javap <options> <classes>`

## 类加载机制

类加载大体上分为加载、验证、准备、解析、初始化五步，其中验证、准备、解析被统称为连接

完整的生命周期如下

![img](笔记.assets\类加载过程-完善.png?msec=1657797480992)

### 加载-查找并加载类的二进制文件

1. 通过类的全限定名获取类的二进制字节流
2. 将字节流代表的静态存储结构转换为方法区的运行时数据结构
3. 在内存中生成一个代表该类的 `Class` 对象，作为方法区这些数据的访问入口

### 验证-确保被加载的类的正确性

1. 文件格式验证：验证魔数、版本号、常量类型等等
2. 元数据验证：进行语义分析，保证描述信息符合Java语言规范要求
3. 字节码验证：确保程序语义合法、符合逻辑的。保证操作数栈和指令代码序列都能配合工作
4. 符号引用验证：确保解析动作能正确执行

### 准备-正式为类变量分配内存并设置类初始值

1. 内存分配类变量（静态变量，不分配实例变量
2. 类变量使用内存在方法区中分配（字符串常量池、静态变量会移动到堆中
3. 初始值“通常情况”是数据类型默认零值

### 解析-将常量池符号引用替换为直接引用

符号引用就是一组描述目标的字面量。比如引用的类名或者啥的。直接引用就是直接指向目标的指针、相对偏移量或一个间接定位到目标的句柄。

### 初始化-执行初始化<cinit()>方法

这一步JVM才开始真正执行类中定义的字节码，只有主动使用类，比如new生成一个对象、读取或设置类的静态字段或者调用静态方法(还有反射、子类初始化出发父类的初始化、用户指定执行的子类)，才会执行类的初始化

## 类加载器

### 种类

1. 启动类加载器：最顶层的加载类，加载lib目录下的jar包和类，也可以加载被-Xbootclasspath参数指定路径中的所有类
2. 扩展类加载器：加载lib/ext目录下的jar包和类，也可以加载java.ext.dirs系统变量所指定的路径下的jar包
3. 应用程序类加载器：加载classpath下的所有jar包和类
4. 用户自定义类加载器：加载用户自己指定jar包和类

### 双亲委派模型

#### 什么是双亲委派模型

ClassLoader进行类加载时会自底向上检查类是否被加载，并自顶向下的尝试加载类。因此所有的请求最终会传送到顶层的启动类加载器`BootstrapClassLoader`中。父类加载器存在且无法处理时，会由当前的加载器进行处理，当父类加载器为null时，会让启动类加载器来进行类的加载

```java
private final ClassLoader parent;
protected Class<?> loadClass(String name, boolean resolve)
        throws ClassNotFoundException
    {
        synchronized (getClassLoadingLock(name)) {
            // 首先，检查请求的类是否已经被加载过
            Class<?> c = findLoadedClass(name);
            if (c == null) {
                long t0 = System.nanoTime();
                try {
                    if (parent != null) {//父加载器不为空，调用父加载器loadClass()方法处理
                        c = parent.loadClass(name, false);
                    } else {//父加载器为空，使用启动类加载器 BootstrapClassLoader 加载
                        c = findBootstrapClassOrNull(name);
                    }
                } catch (ClassNotFoundException e) {
                   //抛出异常说明父类加载器无法完成加载请求
                }

                if (c == null) {
                    long t1 = System.nanoTime();
                    //自己尝试加载
                    c = findClass(name);

                    // this is the defining class loader; record the stats
                    sun.misc.PerfCounter.getParentDelegationTime().addTime(t1 - t0);
                    sun.misc.PerfCounter.getFindClassTime().addElapsedTimeFrom(t1);
                    sun.misc.PerfCounter.getFindClasses().increment();
                }
            }
            if (resolve) {
                resolveClass(c);
            }
            return c;
        }
    }

```

#### 如何打破双亲委派模型

这个问题一般出现在使用自定义的类加载器时。从源码中不难看出，使用双亲委派模型的前提是调用loadClass方法，如果想打破双亲委派模型，让loadClass方法中直接让自定义类加载器进行加载也不失为一种选择。

ps：不想打破双亲委派模型的话，只用重写findClass方法就好嘞，无法被父类加载器加载的类还是会回来通过当前类加载器加载的

#### tomcat为何打破双亲委派模型

要分析这个问题，我们需要考虑双亲委派模型带来了什么？tomcat需要什么？

##### 双亲委派模型

优点：

1. 类型安全，避免自定义的类覆盖核心类库
2. 避免类的重复加载，保证类的唯一性

缺点：

1. 无法同时支持不同版本类库（只加载全限定类名，不关注版本）

##### Tomcat解决的问题

1. 保证各个应用程序的类库独立、相互隔离，即能够加载同一个第三方类库的不同版本
2. 部署在同一个Web容器的相同类库的相同版本共享，避免JVM重复加载
3. web容器的类库需要和应用程序的类库隔离
4. 能够支持jsp文件修改的热部署

很明显的双亲委派模型无法解决1、3、4三个问题，于是tomcat引入Common类加载器、Catalina类加载器、Shared类加载器、WebApp类加载器、Jsp类加载器来解决问题，图示如下。左侧对应webserver的加载器、右侧对应web容器的加载器

![tomcat类加载机制](笔记.assets\tomcat类加载机制.png)

CommonClassLoader：加载/common/*目录下的jar包和类，是tomcat最基本的类加载器，加载路径中的class可以被tomcat和各个webapp访问

CatalinaClassLoader：加载/server/*目录下的jar包和类，是tomcat私有的类加载器，webapp不能访问其加载路径下的class，即对webapp不可见

SharedClassLoader：加载/shared/*目录下的jar包和类，是各个webapp共享的类加载器，对tomcat不可见

WebappClassLoader：加载自己目录下的class文件，webapp私有的类加载器，只对当前webapp可见

每个Web应用程序对应一个WebApp类加载器，每一个Jsp文件对应一个Jsp类加载器

**对应1、3、4问题的解决方法：**

1. CommonClassLoader加载的类可以被CatalinaClassLoader和SharedClassLoader共用，同时它们又可以各自独立的加载类，并互相隔离(实现web容器与应用程序的类库隔离)
2. WebAppClassLoader为Web应用程序私有，通过该加载器加载的类和jar包在各个应用实例之间相互隔离
3. JasperLoader加载范围仅仅是JSP文件编译出的.Class文件，一旦JSP文件修改，马上替换当前JasperLoader，重新进行加载(实现热部署)

如果有特殊情况，tomcat需要父类加载器加载子类加载器中的类，可以使用线程上下文类加载器请求子类加载器完成类加载的动作

## 内存结构

根据《Java 虚拟机规范》中的说法，Java 虚拟机的内存结构可以分为公有和私有两部分。公有指的是所有线程都共享的部分，指的是 Java 堆、方法区、常量池。私有指的是每个线程的私有数据，包括：PC寄存器、Java 虚拟机栈、本地方法栈。

### 运行时数据区域

#### 程序计数器

> 程序计数器是一块较小的内存空间，会指示当前线程所执行的字节码的行号。

因为线程切换时不一定执行完当前任务，所以需要每条线程都有一个独立的程序计数器，保证线程切换后可以恢复到正确的执行位置。补充一下，字节码行号是给字节码解释器提供的，后者通过行号来选取需要执行的字节码指令，分支、循环、跳转、异常处理、线程恢复等功能都需要程序计数器实现

注意一点，程序计数器不会出现OOM(是唯一不会出现OOM的区域哦)

#### Java虚拟机栈

> 除了一些Native方法外的Java方法调用都通过虚拟机栈实现。

栈由一个个栈帧组成，每一次方法调用都会有对应的栈帧被压入栈中，而调用结束后，对应栈帧被弹出

栈帧由**局部变量表、操作数栈、动态链接、方法返回地址**构成

- 局部变量表：存放编译期可知的各种数据类型(基础数据类型)、对象引用
- 操作数栈：存放方法执行过程中产生的中间计算结果，以及一些计算过程中产生的临时变量
- 动态链接：之前提到，在字节码文件中所有变量和方法用都作为符号引用保存在Class文件的常量池里。动态链接的作用是将符号引用转换为调用方法的直接引用(**比如相对偏移量、直接指向目标的指针或能间接定位到目标的句柄**)
- 方法返回地址：存放调用该方法的PC寄存器的值。无论是正常执行完成，还是发现异常退出，都应该返回到该方法被调用的位置。区别在于**通过异常完成出口退出的不会给上层调用者返回值，需要通过异常表缺点返回地址**，而正常情况下会返回到调用者存放当前欲执行指令的地址。

虚拟机栈帧的大小需要动态扩展，可能会面临OOM异常。此外，如果函数调用陷入无限循环，还会面对StackOverFlowError错误

#### 本地方法栈

> 实现Native方法的调用

细节上与虚拟机栈几乎相同，在HotSpot虚拟机中和Java虚拟机栈合二为一

#### 堆

> 线程共享，在虚拟机启动时创建。**此内存区域的唯一目的就是存放对象实例，几乎所有的对象实例以及数组都在这里分配内存。**

堆是GC的主要区域，在JDK1.7及之前由新生代、老年代、永久代构成，而在JDK1.8及之后，永久代被元空间取代。新生代中多是小的、年轻的对象；老年代则由大的、年迈的对象占领；元空间会存放一些方法中的操作临时对象。详细内容会在GC部分提到。

基本上最容易出现的就是OOM异常，可能由于GC回收失败、或者创建对象过大等

#### 方法区

> 线程共享，存储已被虚拟机加载的类信息、字段信息、方法信息、常量、静态变量、即时编译器编译后的代码缓存等数据。（Java虚拟机规范把方法堆的一个逻辑部分，但是它却有一个别名叫 **Non-Heap**（非堆））

堆中有永久代和元空间，它们都是方法区的具体实现，为什么使用元空间来代替永久代？换言之为什么使用直接内存来替代JVM分配？

1. 使用直接内存，元空间可以存储更多的信息
2. 合并HotSpot和JRockit时，后者并未设置永久代

当然，使用直接内存的缺点是，如果不加控制，可能会耗尽所有系统内存，造成OOM异常

#### 运行时常量池

> Class 文件中除了有类的版本、字段、方法、接口等描述信息外，还有用于存放编译期生成的各种字面量（Literal）和符号引用（Symbolic Reference）的 **常量池表(Constant Pool Table)** 。

作为方法区的一部分，也受到OOM异常的影响

#### 字符串常量池

> **字符串常量池** 是 JVM 为了提升性能和减少内存消耗针对字符串（String 类）专门开辟的一块区域，主要目的是为了避免字符串的重复创建。

JDK1.7 之前，字符串常量池存放在永久代。JDK1.7 字符串常量池和静态变量从永久代移动了 Java 堆中。

### 对象创建

以HotSpot虚拟机为例，主要经历5个步骤

1. 类加载检查：没有类加载需要优先考虑进行类加载
2. 分配内存：有**指针碰撞、空闲列表**两种分配方式。为保证线程安全，会采用CAS+失败重试保证操作的原子性，并首先在TLAB分配内存
3. 初始化零值：将内存空间初始化为零值
4. 设置对象头：即设置对象的哈希码、GC分代年龄，是否启用偏向锁等等
5. 执行init方法：构造方法中会创建<init>方法，用于初始化实例变量

### 对象内存布局

由对象头、实例数据、对齐填充组成

1. 对象头：存储运行时数据+存储类型指针
2. 实例数据：对象存储的信息，属性、方法等
3. 访问定位：主流的方式有两种，一种是使用句柄，另一种是使用直接指针

#### 内存分配方式

- 指针碰撞 ：
	- 适用场合 ：堆内存规整（即没有内存碎片）的情况下。
	- 原理 ：用过的内存全部整合到一边，没有用过的内存放在另一边，中间有一个分界指针，只需要向着没用过的内存方向将该指针移动对象内存大小位置即可。
	- 使用该分配方式的 GC 收集器：Serial, ParNew
- 空闲列表 ：
	- 适用场合 ： 堆内存不规整的情况下。
	- 原理 ：虚拟机会维护一个列表，该列表中会记录哪些内存块是可用的，在分配的时候，找一块儿足够大的内存块儿来划分给对象实例，最后更新列表记录。
	- 使用该分配方式的 GC 收集器：CMS

#### TLAB

线程的私有缓存区域，包含在Eden空间内。在多线程环境时，使用TLAB可以避免线程安全问题，提高内存分配效率。(使用TLAB进行内存分配被称作**快速分配策略**)

#### 句柄和指针的区别

句柄可以理解为间接地址。如果使用句柄的话，那么 Java 堆中将会划分出一块内存来作为句柄池，reference 中存储的就是对象的句柄地址，而句柄中包含了对象实例数据与对象类型数据各自的具体地址信息。其优势在于句柄地址稳定，移动对象时不会改变句柄中的实例数据指针。

直接指针就是对象的地址，reference中存储的是对象的地址。使用直接地址的好处是速度快，节省了一次指针定位的时间开销。HotSpot使用的是直接指针的方法

### 逃逸分析

分析一个新的对象引用的使用范围从而决定是否要将对象分配到栈上，基本行为就是分析对象的动态作用域

1. 对象在方法中被定义后，对象只在方法内部使用，则认为没有发生逃逸(作为返回值返回也是逃逸，因为其它线程可以访问，被称为线程逃逸)
2. 当一个对象在方法中被定义后，它被外部方法所引用，则认为发生逃逸

当满足条件1时，如何进行分配的优化呢？

1. **栈上分配**：将堆分配转化为栈分配
2. **同步省略**：如果一个对象被发现只能从一个线程被访问到，那么对于这个对象的操作可以不考虑同步
3. **分离对象或标量替换**：有的对象可能不需要作为一个连续的内存结构存在也可以被访问到，那么对象的部分（或全部）可以不存储在内存，而存储在 CPU 寄存器

## 内存模型

> 内存模型，又称JMM。由线程私有的栈和共用的堆组成

栈：存储基本类型局部变量和引用类型局部变量的引用

堆：存储静态类变量、类定义、成员变量、对象本身

### 可见性实现

> 一个线程对共享变量做了修改之后，其他的线程立即能够看到（感知到）该变量的这种修改（变化）

实现可见性的常用方法有两种

1. 使用synchronized关键字：既保证多线程的并发有序性，又保证多线程的内存可见性
2. 使用volatile关键字：只能保证内存可见性，不保证并发有序性（不具有原子性）

**`volatile`**

1. 防止指令重排序
2. 实现可见性
3. 保证单次读/写的原子性

原理：基于内存屏障实现

如果对声明了 volatile 的变量进行写操作，JVM 就会向处理器发送一条 lock 前缀的指令，将这个变量所在缓存行的数据写回到系统内存。为了保证各个处理器的缓存是一致的，实现了缓存一致性协议(MESI)，每个处理器通过嗅探在总线上传播的数据来检查自己缓存的值是不是过期了，当处理器发现自己缓存行对应的内存地址被修改，就会将当前处理器的缓存行设置成无效状态，当处理器对这个数据进行修改操作的时候，会重新从系统内存中把数据读到处理器缓存里。

### 重排序

> 有序性：对于一个线程的代码而言，我们总是以为代码的执行是从前往后的，依次执行的。这么说不能说完全不对，在单线程程序里，确实会这样执行；但是在多线程并发时，程序的执行就有可能出现乱序。用一句话可以总结为：在本线程内观察，操作都是有序的；如果在一个线程中观察另外一个线程，所有的操作都是无序的。前半句是指“线程内表现为串行语义（WithIn Thread As-if-Serial Semantics）”,后半句是指“指令重排”现象和“工作内存和主内存同步延迟”现象

重排序是为提高执行程序的性能而生，可以分为三种类型（后两者统称为处理器重排序）：

1. 编译器优化：不改变单线程程序语义下的重排序
2. 指令级并行：如果不存在**数据依赖**性，可以改变语句对应机器指令的重排序
3. 内存系统的重排序：处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行

JMM通过禁止特定类型的编译器重排序和处理器重排序，为程序员提供一致的内存可见性保证

实现重排序的方法是**内存屏障**，由于写缓冲区的存在，处理器排序后执行内存读/写操作顺序和实际并不一致，所以一些情况下需要禁止重排序。一般来说store-load的重排序是被允许的，大部分情况下对数据依赖的操作做重排序是不允许的。

#### 数据依赖

如果两个操作访问同一个变量，且这两个操作中有一个为写操作，此时这两个操作之间就存在数据依赖性。即以下三种情况：

1. 写一个变量后，再读这个变量
2. 或者写一个变量后，再写这个变量
3. 或者读一个变量后，再写这个变量

#### 内存屏障

JMM有以下四类内存屏障：

| **屏障类型**        | 指令示例                   | **说明**                                                     |
| ------------------- | -------------------------- | ------------------------------------------------------------ |
| LoadLoad Barriers   | Load1; LoadLoad; Load2     | 确保 Load1 数据的装载，之前于 Load2 及所有后续装载指令的装载。 |
| StoreStore Barriers | Store1; StoreStore; Store2 | 确保 Store1 数据对其他处理器可见（刷新到内存），之前于 Store2 及所有后续存储指令的存储。 |
| LoadStore Barriers  | Load1; LoadStore; Store2   | 确保 Load1 数据装载，之前于 Store2 及所有后续的存储指令刷新到内存。 |
| StoreLoad Barriers  | Store1; StoreLoad; Load2   | 确保 Store1 数据对其他处理器变得可见（指刷新到内存），之前于 Load2 及所有后续装载指令的装载。 |

#### as-if-serial语义

as-if-serial 语义的意思指：不管怎么重排序（编译器和处理器为了提高并行度），（单线程）程序的执行结果不能被改变。编译器，runtime 和处理器都必须遵守 as-if-serial 语义。

为了遵守 as-if-serial 语义，编译器和处理器不会对存在数据依赖关系的操作做重排序，因为这种重排序会改变执行结果。但是，如果操作之间不存在数据依赖关系，这些操作可能被编译器和处理器重排序。

#### happens-before

> 如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须存在 happens-before 关系

常用的有以下四个：

- 程序顺序规则：一个线程中的每个操作，happens- before 于该线程中的任意后续操作。
- 监视器锁规则：对一个监视器锁的解锁，happens- before 于随后对这个监视器锁的加锁。
- volatile 变量规则：对一个 volatile 域的写，happens- before 于任意后续对这个 volatile 域的读。
- 传递性：如果 A happens- before B，且 B happens- before C，那么 A happens- before C。

### 顺序一致性

顺序一致性内存模型是一个被计算机科学家理想化了的理论参考模型，它为程序员提供了极强的内存可见性保证。顺序一致性内存模型有两大特性：

1. 一个线程中的所有操作必须按照程序的顺序来执行
2. 所有线程都只能看到一个单一的操作执行顺序

## 垃圾回收

### 内存分配和回收原则

1. 对象优先在Eden区分配：大部分对象在新生代的Eden区分配，当Eden区没有足够空间进行分配时，会发起Minor GC
2. 大对象或长期存活的对象进入老年代：大对象直接进入老年代是为了避免分配担保机制带来的复制降低效率；长期存活对象的判断标准一般是年龄大小(这个数据在对象头里)，大于surivor区的平均值(可以设置)
3. 主要进行GC的区域：分门派，有Partial GC和Full GC两类。其中Partial GC分为Minor GC、Major GC、Mixed GC
4. 空间分配担保：在Minor GC前，虚拟机会检查老年代的最大可用连续空间是否大于新生代所有对象总空间

### 死亡对象判断方法

1. 引用计数法：基本原理是，判断有多少地方引用该对象，如果没有就可以GC，但难以解决相互循环引用问题
2. 可达性分析算法：通过GC Roots对象作为起点从这些节点开始向下搜索，节点所走过的路径称为引用链，当一个对象到 GC Roots 没有任何引用链相连的话，则证明此对象是不可用的，需要被回收。

常见的GC Roots有:

- 虚拟机栈(栈帧中的本地变量表)中引用的对象
- 本地方法栈(Native 方法)中引用的对象
- 方法区中类静态属性引用的对象
- 方法区中常量引用的对象
- 所有被同步锁持有的对象

#### 引用类型

1. 强引用：不会被GC回收，直接new出来的引用
2. 软引用：如果内存不足可能会被回收，可以用作内存敏感的高速缓存，或者和一个引用队列（ReferenceQueue）联合使用，如果软引用所引用的对象被垃圾回收，JAVA 虚拟机就会把这个软引用加入到与之关联的引用队列中。
3. 弱引用：被发现就会被回收，一般和一个引用队列（ReferenceQueue）联合使用，如果弱引用所引用的对象被垃圾回收，Java 虚拟机就会把这个弱引用加入到与之关联的引用队列中
4. 虚引用：顾名思义，就是形同虚设，与其他几种引用都不同，虚引用并不会决定对象的生命周期。如果一个对象仅持有虚引用，那么它就和没有任何引用一样，在任何时候都可能被垃圾回收。

**虚引用主要用来跟踪对象被垃圾回收的活动**

**虚引用与软引用和弱引用的一个区别在于：** 虚引用必须和引用队列（ReferenceQueue）联合使用。当垃圾回收器准备回收一个对象时，如果发现它还有虚引用，就会在回收对象的内存之前，把这个虚引用加入到与之关联的引用队列中。程序可以通过判断引用队列中是否已经加入了虚引用，来了解被引用的对象是否将要被垃圾回收。程序如果发现某个虚引用已经被加入到引用队列，那么就可以在所引用的对象的内存被回收之前采取必要的行动。

特别注意，在程序设计中一般很少使用弱引用与虚引用，使用软引用的情况较多，这是因为**软引用可以加速 JVM 对垃圾内存的回收速度，可以维护系统的运行安全，防止内存溢出（OutOfMemory）等问题的产生**

#### 判断废弃常量

假如在字符串常量池中存在字符串 "abc"，如果当前没有任何 String 对象引用该字符串常量的话，就说明常量 "abc" 就是废弃常量，如果这时发生内存回收的话而且有必要的话，"abc" 就会被系统清理出常量池了。

#### 判断废弃类

1. 不存在类的实例
2. 不存在类的加载器
3. 不存在类的Class对象被引用的地方

满足以上三个条件即为无用的类，但不一定会被回收

### 垃圾回收算法

#### 标记-清除算法

最基础的，首先标记出不需要回收的对象，然后清除所有未被标记的对象

缺点：

1. 效率低
2. 产生大量不连续碎片

#### 标记-复制算法

将内存分为大小相同的两块，每次使用其中的一块。当这一块的内存使用完后，就将还存活的对象复制到另一块去，然后再把使用的空间一次清理掉。这样就使每次的内存回收都是对内存区间的一半进行回收。

**复制到另一边可以避免产生不连续碎片**

#### 标记-整理算法

适合老年代的标记算法，标记过程仍然与“标记-清除”算法一样，但后续步骤不是直接对可回收对象回收，而是让所有存活的对象向一端移动，然后直接清理掉端边界以外的内存

#### 分代收集算法

对于不同特征的对象，用不同的垃圾回收方法。**在新生代中，每次收集都会有大量对象死去，所以可以选择”标记-复制“算法，只需要付出少量对象的复制成本就可以完成每次垃圾收集。而老年代的对象存活几率是比较高的，而且没有额外的空间对它进行分配担保，所以我们必须选择“标记-清除”或“标记-整理”算法进行垃圾收集。**

### 垃圾回收器

#### Serial收集器

**单线程**，这不仅仅意味着它只会使用一条垃圾收集线程去完成垃圾收集工作，更重要的是它在进行垃圾收集工作的时候必须暂停其他所有的工作线程（ **"Stop The World"** ），直到它收集结束。

**新生代采用标记-复制算法，老年代采用标记-整理算法。**

在单线程收集器中效率高且简单，对于运行在Client模式下的虚拟机还挺好用

#### ParNew收集器

ParNew 收集器其实就是 **Serial 收集器的多线程版本**，除了使用多线程进行垃圾收集外，其余行为（控制参数、收集算法、回收策略等等）和 Serial 收集器完全一样。注意这里的并发，是GC线程的并发，应用程序会暂停

#### Parallel Scavenge

Parallel Scavenge 收集器**关注点是吞吐量（高效率的利用 CPU）**。CMS 等垃圾收集器的关注点更多的是用户线程的停顿时间（提高用户体验）。所谓吞吐量就是 CPU 中用于运行用户代码的时间与 CPU 总消耗时间的比值

**新生代采用标记-复制算法，老年代采用标记-整理算法。**

#### CMS收集器

**垃圾收集线程与用户线程（基本上）同时工作。**

基于标记-清除算法实现，分为以下四步：

- **初始标记：** 暂停所有的其他线程，并记录下直接与 root 相连的对象，速度很快 ；
- **并发标记：** 同时开启 GC 和用户线程，用一个闭包结构去记录可达对象。但在这个阶段结束，这个闭包结构并不能保证包含当前所有的可达对象。因为用户线程可能会不断的更新引用域，所以 GC 线程无法保证可达性分析的实时性。所以这个算法里会跟踪记录这些发生引用更新的地方。
- **重新标记：** 重新标记阶段就是为了修正并发标记期间因为用户程序继续运行而导致标记产生变动的那一部分对象的标记记录，这个阶段的停顿时间一般会比初始标记阶段的时间稍长，远远比并发标记阶段时间短
- **并发清除：** 开启用户线程，同时 GC 线程开始对未标记的区域做清扫

有三个缺点：

- **对 CPU 资源敏感；**
- **无法处理浮动垃圾；**（浮动垃圾是垃圾回收适产生的新的来及，只能等下次处理）
- **它使用的回收算法-“标记-清除”算法会导致收集结束时会有大量空间碎片产生。**

#### G1回收器

- **并行与并发**：G1 能充分利用 CPU、多核环境下的硬件优势，使用多个 CPU（CPU 或者 CPU 核心）来缩短 Stop-The-World 停顿时间。部分其他收集器原本需要停顿 Java 线程执行的 GC 动作，G1 收集器仍然可以通过并发的方式让 java 程序继续执行。
- **分代收集**：虽然 G1 可以不需要其他收集器配合就能独立管理整个 GC 堆，但是还是保留了分代的概念。
- **空间整合**：与 CMS 的“标记-清理”算法不同，G1 从整体来看是基于“标记-整理”算法实现的收集器；从局部上来看是基于“标记-复制”算法实现的。
- **可预测的停顿**：这是 G1 相对于 CMS 的另一个大优势，降低停顿时间是 G1 和 CMS 共同的关注点，但 G1 除了追求低停顿外，还能建立可预测的停顿时间模型，能让使用者明确指定在一个长度为 M 毫秒的时间片段内。

运作步骤与CMS收集器是类似的，但是其**在后台维护了一个优先列表，每次根据允许的收集时间，优先选择回收价值最大的 Region(这也就是它的名字 Garbage-First 的由来)** 。

## 调优排错

### 调优参数

对于堆而言

1. 调整最大堆内存和最小堆内存：通常会将 -Xms 与 -Xmx两个参数配置成相同的值。避免重新分隔计算堆区大小
2. 调整新生代和老年代的比值：一般新生代占整个堆的五分之一
3. 调整s区和Eden区的比值：一般一个s区占新生代代的十分之一
4. 设置新生代和老年代的大小：前面设置好了这里就不用设置了

当然还可以对其它参数进行调优，此处不再多提

### 排错

1. jps：查看Java进程的启动类、传入参数、Java虚拟机参数等
2. jmap：输出所有内存中对象的工具，甚至可以将VM 中的heap，以二进制输出成文本。打印出某个java进程（使用pid）内存内的，所有‘对象’的情况
3. jstack：java虚拟机自带的一种堆栈跟踪工具。常用于分析线程问题(如线程间死锁[deadLock])
4. jstat：JVM statistics Monitoring，用于监视虚拟机运行时状态信息的命令，它可以显示出虚拟机进程中的类装载、内存、垃圾收集、JIT编译(即使编译)等运行数据。
5. arthas：在线分析，阿里巴巴的！

# Java多线程

> 借助《深入浅出Java多线程一书》，重新整理下自己学过的JUC相关知识

## 基础篇

### 1.进程与线程的基本概念

> **背景知识**：一开始的计算机是用户输入指令，对应计算机执行一个操作，效率相当低下。在引入批处理操作系统后，用户可以把一系列需要进行操作的指令写下来，形成一个清单，一次性交给计算机，计算机一次性执行所有操作。**但是，批处理操作系统的指令运行方式仍然是串行的，内存中始终只有一个程序在运行。**

#### 基本概念

内存中能不能有多个程序同时运行？如何同时运行？这里引入进程的概念

**进程是应用程序在内存中分配的空间，也就是正在运行的程序。**塔噶，在一开始CPU是采用时间片轮转的方式来运行进程，CPU会为每个进程分配一个时间段。这个时候从宏观上来看，一个大的时间片内同时有多个任务在执行，但是**实际上对于单核CPU而言，具体时刻下还是只有一个任务能够使用CPU资源。**并且，如果一个进程有多个子任务，也只能逐个执行，效率低下

有没有可能，一个进程可以同时执行多个子任务呢？这里引入线程的概念

**线程负责单个的子任务，一个进程包含多个线程**。

用操作系统里的话说，**进程是资源的最小分配单位，线程是CPU调度的最小单位**

#### 线程vs进程

1. 并发

	1. 进程间通信比较复杂，线程间通信比较简单
	2. 进程重量级，线程轻量级，后者开销较小

2. 区别（**是否单独占有内存地址空间及其它系统资源（比如I/O）**）

	总的来说：进程单独占有一定内存空间，进程间存在内存隔离；线程共享所属进程占有的内存地址空间和资源

	1. 前者同步简单，数据共享复杂；后者同步复杂，数据共享简单
	2. 一个进程出现问题不影响其它进程，可靠性高；一个线程出现问题可能导致整个程序崩溃，可靠性低
	3. 进程创建销毁需要寄存器、栈信息的基础上，还需要资源分配回收和页调度，开销较大；线程只需要保存寄存器和栈信息，开销较小

#### 上下文切换

> 上下文切换指CPU从一个进程或线程切换到另一个进程或线程
>
> **上下文是指某一时间点CPU寄存器和程序计数器的内容**

寄存器是cpu内部的闪存，通常存储计算过程的中间值

程序计数器在JVM内存结构中也有提到类似的，会指示CPU正在执行的位置，存储正在执行指令位置或即将执行指令位置

### 2.多线程入门类和接口

> 大概两种，一种是有返回值的异步模型，另一种是没有返回值的
>
> 1. Thread、Runnable
> 2. Callable、Future、FutureTask

#### Thread类和Runnable接口

通过Thread和Runnable实现多线程，需要重写或者实现run方法。

需要注意的是Runnable是函数式接口，可以这样进行Thread类的构造

```java
public static void main(String[] args) {
    new Thread(() -> {
        System.out.println("Java 8 匿名内部类");
    }).start();
}
```
Thread类一般常用的构造方法是new Thread(Runnable target, String name);

常用的方法如下：

- currentThread()：静态方法，返回对当前正在执行的线程对象的引用；
- start()：开始执行线程的方法，java虚拟机会调用线程内的run()方法；
- yield()：yield在英语里有放弃的意思，同样，这里的yield()指的是当前线程愿意让出对当前处理器的占用。这里需要注意的是，就算当前线程调用了yield()方法，程序在调度的时候，也还有可能继续运行这个线程的；
- sleep()：静态方法，使当前线程睡眠一段时间；
- join()：使当前线程等待另一个线程执行完毕之后再继续执行，内部调用的是Object类的wait方法实现的；

**`Thread` pk `Runnable`**

1. Runnable作为接口，比Thread使用更灵活
2. Runnable更轻量，只提供了run方法
3. 可以对一个实现Runnable的类创建多个线程，降低线程对象和任务的耦合性，更符合面向对象的特点

#### Callable接口

`Callable`与`Runnable`类似，同样是只有一个抽象方法的函数式接口。不同的是，`Callable`提供的方法是**有返回值**的，而且支持**泛型**。

```java
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;
}
```

`Callable`一般配合线程池工具`ExecutorService`来使用。后者使用`submit`方法来让一个`Callable`接口执行。它会返回一个`Future`，我们后续的程序可以通过这个`Future`的`get`方法得到结果。

#### Future接口和FutureTask类

Future接口代码如下：

```java
public abstract interface Future<V> {
    public abstract boolean cancel(boolean paramBoolean);
    public abstract boolean isCancelled();
    public abstract boolean isDone();
    public abstract V get() throws InterruptedException, ExecutionException;
    public abstract V get(long paramLong, TimeUnit paramTimeUnit)
            throws InterruptedException, ExecutionException, TimeoutException;
}
```

cancel方法**试图取消**一个线程的执行，参数paramBoolean表示是否采用终端方式取消线程执行，返回值则是是否能取消成功的意思

FutureTask实现RunnableFuture接口（即实现Runnable接口和Future接口）

```java
class TaskTest implements Callable<Integer> {
    @Override
    public Integer call() throws Exception {
        // 模拟计算需要一秒
        Thread.sleep(1000);
        return 2;
    }

    @Test
    void futureTest() throws ExecutionException, InterruptedException {
        ExecutorService executor = Executors.newCachedThreadPool();
        TaskTest task = new TaskTest();
        Future<Integer> result = executor.submit(task);
        // 注意调用get方法会阻塞当前线程，直到得到结果。
        // 所以实际编码中建议使用可以设置超时时间的重载get方法。
        System.out.println(result.get());
    }
    @Test
    void futureTaskTest() throws ExecutionException, InterruptedException {
        ExecutorService executor = Executors.newCachedThreadPool();
        FutureTask<Integer> futureTask = new FutureTask<>(new TaskTest());
        executor.submit(futureTask);
        // 注意调用get方法会阻塞当前线程，直到得到结果。
        // 所以实际编码中建议使用可以设置超时时间的重载get方法。
        System.out.println(futureTask.get());
    }

}
```

FuturTask的任务运行状态

```java
/**
  *
  * state可能的状态转变路径如下：
  * NEW -> COMPLETING -> NORMAL
  * NEW -> COMPLETING -> EXCEPTIONAL
  * NEW -> CANCELLED
  * NEW -> INTERRUPTING -> INTERRUPTED
  */
private volatile int state;
private static final int NEW          = 0;
private static final int COMPLETING   = 1;
private static final int NORMAL       = 2;
private static final int EXCEPTIONAL  = 3;
private static final int CANCELLED    = 4;
private static final int INTERRUPTING = 5;
private static final int INTERRUPTED  = 6;
```

### 3.线程组和线程优先级

#### ThreadGroup

ThreadGroup和Thread的关系就如同他们的字面意思一样简单粗暴，每个Thread必然存在于一个ThreadGroup中，Thread不能独立于ThreadGroup存在。执行main()方法线程的名字是main，如果在new Thread时没有显式指定，那么默认将父线程（当前执行new Thread的线程）线程组设置为自己的线程组。

不难看出，ThreadGroup是标准的**向下引用的树状结构，可以有效防止上级线程被下级线程引用而无法回收**

此外，线程组还可以对线程处理异常进行统一，检查线程权限(checkAcess方法)

#### 线程优先级

通常情况下，高优先级的线程将会比低优先级的线程有**更高的几率**得到执行。但是**Java程序中对线程所设置的优先级只是给操作系统一个建议，操作系统不一定会采纳。而真正的调用顺序，是由操作系统的线程调度算法决定的**。

Java提供一个**线程调度器**来监视和控制处于**RUNNABLE状态**的线程。线程的调度策略采用**抢占式**，优先级高的线程比优先级低的线程会有更大的几率优先执行。在优先级相同的情况下，按照“先到先得”的原则。每个Java程序都有一个默认的主线程，就是通过JVM启动的第一个线程main线程。

**`守护线程`**：默认优先级比较低。当所有的非守护线程结束时，守护线程也会自动结束，就免去了还要继续关闭子线程的麻烦。

一个线程默认是非守护线程，需要通过Thread类的setDaemon方法来设置

### 4.线程状态及主要转换方法

在现在的操作系统中，线程是被视为轻量级进程的，所以**操作系统线程的状态其实和操作系统进程的状态是一致的**。

![img](笔记.assets\系统进程状态转换图.png)

主要的状态有三个

1. ready：线程等待使用CPU，可以由调度进入running状态
2. running：线程正在使用CPU
3. waiting：线程在等待其他资源或者事件调用

#### Java线程状态

```java
// Thread.State 源码
public enum State {
    NEW,
    RUNNABLE,
    BLOCKED,
    WAITING,
    TIMED_WAITING,
    TERMINATED;
}
```

1. NEW：处于NEW状态的线程此时尚未启动。这里的尚未启动指的是还没调用Thread实例的start()方法。(多次调用start方法，会因为线程状态不为NEW而报出异常)
2. RUNNABLE：表示当前线程正在运行中。处于RUNNABLE状态的线程在Java虚拟机中运行，也有可能在等待其他系统资源（比如I/O）(一眼可以看出包含ready和waiting两个状态)
3. BLOCKED：阻塞状态。处于BLOCKED状态的线程正等待锁的释放以进入同步区。
4. WAITING：等待状态。处于等待状态的线程变成RUNNABLE状态需要其他线程唤醒。进入WAITING状态可以调用如下三个方法
	1. Object.wait()：使当前线程处于等待状态直到另一个线程唤醒它；
	2. Thread.join()：等待线程执行完毕，底层调用的是Object实例的wait方法；
	3. LockSupport.park()：除非获得调用许可，否则禁用当前线程进行线程调度。  
5. TIMED_WAITING：超时等待状态。线程等待一个具体的时间，时间到后会被自动唤醒。调用如下方法会使线程进入超时等待状态
	1. hread.sleep(long millis)：使当前线程睡眠指定时
	2. Object.wait(long timeout)：线程休眠指定时间，等待期间可以通过notify()/notifyAll()唤醒；
	3. Thread.join(long millis)：等待当前线程最多执行millis毫秒，如果millis为0，则会一直执行；
	4. LockSupport.parkNanos(long nanos)： 除非获得调用许可，否则禁用当前线程进行线程调度指定时间
	5. LockSupport.parkUntil(long deadline)：同上，也是禁止线程进行调度指定时间；
6. TERMINATED：终止状态。此时线程已执行完毕。

由上不难得出线程状态的转换图：

![img](笔记.assets\线程状态转换图.png)

#### 线程中断

> 在某些情况下，我们在线程启动后发现并不需要它继续执行下去时，需要中断线程。目前在Java里还没有安全直接的方法来停止线程，但是Java提供了线程中断机制来处理需要中断线程的情况。
>
> 线程中断机制是一种协作机制。需要注意，通过中断操作并不能直接终止一个线程，而是通知需要被中断的线程自行处理。

Thread类提供的线程中断方法：

Thread.interrupt()：中断线程。这里的中断线程并不会立即停止线程，而是设置线程的中断状态为true（默认是flase）；

Thread.interrupted()：测试当前线程是否被中断。线程的中断状态受这个方法的影响，意思是调用一次使线程中断状态设置为true，连续调用两次会使得这个线程的中断状态重新转为false；

Thread.isInterrupted()：测试当前线程是否被中断。与上面方法不同的是调用这个方法并不会影响线程的中断状态。

### 5.线程间的通信

线程间的通信大体上两类，一种是共享内存，另一种就是通知的机制

#### 锁与同步

基于锁的机制，实现线程同步。一个锁同一时间只能被一个线程持有，只有获取锁的线程可以执行。

线程同步是线程之间按照**一定的顺序**执行。

以下代码可以实现ThreadA执行完后，ThreadB才会执行

```java
public class ObjectLock {
    private static Object lock = new Object();

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 100; i++) {
                    System.out.println("Thread A " + i);
                }
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 100; i++) {
                    System.out.println("Thread B " + i);
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(10);
        new Thread(new ThreadB()).start();
    }
}
```

#### 等待/通知机制

基于Object类的wait方法和notify、notifyAll方法实现

wait方法释放锁，notify方法随机通知一个在等待的线程继续执行，notifyAll会叫醒所有正在等待的线程

> 需要注意的是等待/通知机制使用的是使用同一个对象锁，如果你两个线程使用的是不同的对象锁，那它们之间是不能用等待/通知机制通信的。

```java
public class WaitAndNotify {
    private static Object lock = new Object();

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 5; i++) {
                    try {
                        System.out.println("ThreadA: " + i);
                        lock.notify();
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                lock.notify();
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            synchronized (lock) {
                for (int i = 0; i < 5; i++) {
                    try {
                        System.out.println("ThreadB: " + i);
                        lock.notify();
                        lock.wait();
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
                lock.notify();
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(1000);
        new Thread(new ThreadB()).start();
    }
}

// 输出：
ThreadA: 0
ThreadB: 0
ThreadA: 1
ThreadB: 1
ThreadA: 2
ThreadB: 2
ThreadA: 3
ThreadB: 3
ThreadA: 4
ThreadB: 4
```

#### 信号量

JDK提供了一个类似于“信号量”功能的类`Semaphore`。当然，也可以通过volatile提供内存可见性来实现自己的信号量

```java
public class Signal {
    private static volatile int signal = 0;

    static class ThreadA implements Runnable {
        @Override
        public void run() {
            while (signal < 5) {
                if (signal % 2 == 0) {
                    System.out.println("threadA: " + signal);
                    synchronized (this) {
                        signal++;
                    }
                }
            }
        }
    }

    static class ThreadB implements Runnable {
        @Override
        public void run() {
            while (signal < 5) {
                if (signal % 2 == 1) {
                    System.out.println("threadB: " + signal);
                    synchronized (this) {
                        signal = signal + 1;
                    }
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        new Thread(new ThreadA()).start();
        Thread.sleep(1000);
        new Thread(new ThreadB()).start();
    }
}

// 输出：
threadA: 0
threadB: 1
threadA: 2
threadB: 3
threadA: 4
```

推荐使用信号量的时机：当多于2个线程需要相互合作时（简单的锁和等待/通知机制不太方便）

#### 管道

管道是基于“管道流”的通信方式。JDK提供了`PipedWriter`、 `PipedReader`、 `PipedOutputStream`、 `PipedInputStream`。其中，前面两个是基于字符的，后面两个是基于字节流的。

```java
public class Pipe {
    static class ReaderThread implements Runnable {
        private PipedReader reader;

        public ReaderThread(PipedReader reader) {
            this.reader = reader;
        }

        @Override
        public void run() {
            System.out.println("this is reader");
            int receive = 0;
            try {
                while ((receive = reader.read()) != -1) {
                    System.out.print((char)receive);
                }
            } catch (IOException e) {
                e.printStackTrace();
            }
        }
    }

    static class WriterThread implements Runnable {

        private PipedWriter writer;

        public WriterThread(PipedWriter writer) {
            this.writer = writer;
        }

        @Override
        public void run() {
            System.out.println("this is writer");
            int receive = 0;
            try {
                writer.write("test");
            } catch (IOException e) {
                e.printStackTrace();
            } finally {
                try {
                    writer.close();
                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        }
    }

    public static void main(String[] args) throws IOException, InterruptedException {
        PipedWriter writer = new PipedWriter();
        PipedReader reader = new PipedReader();
        writer.connect(reader); // 这里注意一定要连接，才能通信

        new Thread(new ReaderThread(reader)).start();
        Thread.sleep(1000);
        new Thread(new WriterThread(writer)).start();
    }
}

// 输出：
this is reader
this is writer
test
```

执行流程：

1. 线程ReaderThread开始执行，
2. 线程ReaderThread使用管道reader.read()进入”阻塞“，
3. 线程WriterThread开始执行，
4. 线程WriterThread用writer.write("test")往管道写入字符串，
5. 线程WriterThread使用writer.close()结束管道写入，并执行完毕，
6. 线程ReaderThread接受到管道输出的字符串并打印，
7. 线程ReaderThread执行完毕。

推荐使用管道通信的时机：当一个线程需要先另一个线程发送一个信息（比如字符串）或者文件等等时（管道多班于I/O流有关）

#### 其它通信方法

1. join：基于wait实现，调用join方法会让当前线程陷入“等待”状态，等join的这个线程执行完成后，再继续执行当前线程。
2. sleep：调用sleep方法会让当前线程休眠一段事件，注意sleep方法是不会释放当前的锁的，而wait方法会
3. ThreadLocal：为每个线程都创建一个**副本**，每个线程可以访问自己内部的副本变量。一般来说如果需要将类的某个静态变量与线程状态绑定可以使用ThreadLocal，常用场景有保存UserId、数据库连接、Session等
4. InheritableThreadLocal：InheritableThreadLocal类与ThreadLocal类稍有不同，Inheritable是继承的意思。它不仅仅是当前线程可以存取副本值，而且它的子线程也可以存取这个副本值。

## 原理篇

### 1.补充知识

#### 并发问题出现根源

1. 可见性：一个线程对共享变量的修改，另外一个线程能够立刻看到。
2. 原子性：即一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。
3. 有序性：即程序执行的顺序按照代码的先后顺序执行

由于CPU缓存导致的主存与缓存并不时刻同步；分时复用的线程切换导致的线程并不一次执行完成；重排序导致的代码不一定按序执行，出现并发问题

#### 线程通信/同步

解决线程通信/同步的两种常见模型

![img](笔记.assets\两种并发模型的比较.png)

显然的，Java使用共享内存模型

#### 内存不可见问题

既然内存是共享的，为什么存在内存不可见问题？

> 线程之间的共享变量存在主内存中，每个线程都有一个私有的本地内存，存储了该线程以读、写共享变量的副本。本地内存是Java内存模型的一个抽象概念，并不真实存在。它涵盖了缓存、写缓冲区、寄存器等。

JMM规定：线程对共享变量的所有操作都必须在自己的本地内存中进行，不能直接从主内存中读取。当共享变量被其它线程更新时，当前线程并不能敏感的发现这一问题

#### JMM同步程序的顺序一致性效果补充

**JMM的具体实现方针是：在不改变（正确同步的）程序执行结果的前提下，尽量为编译期和处理器的优化打开方便之门**。

#### JMM未同步程序的顺序一致性效果补充

**JMM没有保证未同步程序的执行结果与该程序在顺序一致性中执行结果一致。因为如果要保证执行结果一致，那么JMM需要禁止大量的优化，对程序的执行性能会产生很大的影响。**

1. JMM不保证单线程内操作按顺序执行，但保证结果正确
2. JMM不保证所有线程能看到一致的操作执行顺序（不保证所有操作可见）
3. JMM不保证64位long、double型变量读、写操作具有原子性

#### happens-before原则补充

1. start规则：如果线程A执行操作ThreadB.start()启动线程B，那么A线程的ThreadB.start（）操作happens-before于线程B中的任意操作
2. join规则：如果线程A执行操作ThreadB.join（）并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回

#### 为什么允许store-load方法的重排序

由于写缓冲区仅对自己的处理器可见，它会导致处理器执行内存操作的顺序可能会与内存实际的操作执行顺序不一致。由于现代的处理器都会使用写缓冲区，因此现代的处理器都会允许对写 - 读操做重排序。

### 2.volatile

#### 概念回顾

1. 内存可见性：**指的是线程之间的可见性，当一个线程修改了共享变量时，另一个线程可以读取到这个修改后的值**。
2. 重排序：为优化程序性能，对原有的指令执行顺序进行优化重新排序。重排序可能发生在多个阶段，比如编译重排序、CPU重排序等
3. happens-before规则：是一个给程序员使用的规则，只要程序员在写代码的时候遵循happens-before规则，JVM就能保证指令在多线程之间的顺序性符合程序员的预期

#### 功能

- 保证变量的**内存可见性**
- 禁止volatile变量与普通变量**重排序**

如何实现内存可见性？

一个线程对`volatile`修饰的变量进行**写操作**时，JMM会立即把该线程对应的本地内存中的共享变量的值刷新到主内存；当一个线程对`volatile`修饰的变量进行**读操作**时，JMM会把立即该线程对应的本地内存置为无效，从主内存中读取共享变量的值。

如何实现重排序？

这里再深入一点进行介绍**内存屏障**！其有两个作用

1. 阻止屏障两侧指令重排序
2. 强制把写缓冲区/高速缓存中的脏数据等写回主内存，或者让缓存中相应的数据失效。

volatile使用内存屏障策略如下:

1. 在每个volatile写操作前插入一个StoreStore屏障；
2. 在每个volatile写操作后插入一个StoreLoad屏障；
3. 在每个volatile读操作后插入一个LoadLoad屏障；
4. 在每个volatile读操作后再插入一个LoadStore屏障。

#### 用途

1. 实现单个volatile变量的读/写原子性，相较于锁性能更好
2. 禁止重排序

在双重校验锁的单例模式中，就使用了volatile关键字

### 3.Synchronized与锁

**Java多线程的锁都是基于对象的**，Java中的每一个对象都可以作为一个锁。

还有一点需要注意的是，我们常听到的**类锁**其实也是对象锁。

#### Synchronized

通常来说，synchronized有以下三种形式

```java
// 关键字在实例方法上，锁为当前实例
public synchronized void instanceLock() {
    // code
}

// 关键字在静态方法上，锁为当前Class对象
public static synchronized void classLock() {
    // code
}

// 关键字在代码块上，锁为括号里面的对象
public void blockLock() {
    Object o = new Object();
    synchronized (o) {
        // code
    }
}
```

介绍一下“临界区”的概念。所谓“临界区”，指的是某一块代码区域，它同一时刻只能由一个线程执行。在上面的例子中，如果`synchronized`关键字在方法上，那临界区就是整个方法内部。而如果是使用synchronized代码块，那临界区就指的是代码块内部的区域。

#### 锁

对象有四种锁状态：无锁、偏向锁、轻量级锁、重量级锁，在对象头的Mark Word字段中会标记当前的锁状态

改变锁的状态有升级和降级两种方案！升级比较简单：竞争激烈就升级；而降级条件比较苛刻，此处不展开

1. 偏向锁：偏向锁会偏向于第一个访问锁的线程，如果在接下来的运行过程中，该锁没有被其他的线程访问，则持有偏向锁的线程将永远不需要触发同步。也就是说，**偏向锁在资源无竞争情况下消除了同步语句，连CAS操作都不做了，提高了程序的运行性能。**

	> 大白话就是对锁置个变量，如果发现为true，代表资源无竞争，则无需再走各种加锁/解锁流程。如果为false，代表存在其他线程竞争资源，那么就会走后面的流程。

	注意撤销偏向锁的开销比较大

2. 轻量级锁：线程通过适应性自旋的方法获取锁，简单来说就是线程如果自旋成功了，则下次自旋的次数会更多，如果自旋失败了，则自旋的次数就会减少。

3. 重量级锁：依赖于操作系统的互斥量（mutex） 实现的，而操作系统中线程间状态的转换需要相对比较长的时间，所以重量级锁效率很低，但被阻塞的线程不会消耗CPU。

总结升级流程如下：

1. 每一个线程在准备获取共享资源时： 第一步，检查MarkWord里面是不是放的自己的ThreadId ,如果是，表示当前线程是处于 “偏向锁” 。
2. 第二步，如果MarkWord不是自己的ThreadId，锁升级，这时候，用CAS来执行切换，新的线程根据MarkWord里面现有的ThreadId，通知之前线程暂停，之前线程将Markword的内容置为空。
3. 第三步，两个线程都把锁对象的HashCode复制到自己新建的用于存储锁的记录空间，接着开始通过CAS操作， 把锁对象的MarKword的内容修改为自己新建的记录空间的地址的方式竞争MarkWord。
4. 第四步，第三步中成功执行CAS的获得资源，失败的则进入自旋 。
5. 第五步，自旋的线程在自旋过程中，成功获得资源(即之前获的资源的线程执行完成并释放了共享资源)，则整个状态依然处于 轻量级锁的状态，如果自旋失败 。
6. 第六步，进入重量级锁的状态，这个时候，自旋的线程进行阻塞，等待之前线程执行完成并唤醒自己。

#### synchronized原理

synchronized会通过monitorenter 进行锁的竞争，加锁成功则进行临界区内操作共享资源，加锁失败则进入等待队列中等待，进入到临界区的线程当发现自己的执行条件不满足时，可以条用wait()方法释放锁，然后使用nofify()或者notifAll()唤醒等待队列的锁，如果条件满足那么就正常的执行临界区的逻辑，操作完毕后monitorexit 退出临界区，释放锁资源，同时通知等待队列的线程

重入时monitor计数器的值+1，每退出一次就会减一，直到完全退出。当然，可重入指的是当前线程可以再次申请锁，其它线程还是不行的

### 4.CAS与原子操作

​	引入乐观锁的概念，又称“无锁”。乐观锁总是假设对共享资源的访问没有冲突，线程可以不停地执行，无需加锁也无需等待。多用于“读多写少“的环境，避免频繁加锁影响性能；而悲观锁多用于”写多读少“的环境，避免频繁失败和重试影响性能。**不会出现死锁的情况**

#### CAS

实现乐观锁，CAS必不可少。比较并交换（Compare And Swap）。在CAS中，有这样三个值：

- V：要更新的变量(var)
- E：预期值(expected)
- N：新值(new)

比较并交换的过程如下：

判断V是否等于E，如果等于，将V的值设置为N；如果不等，说明已经有其它线程更新了V，则当前线程放弃更新，什么都不做。

CAS是一种原子操作，它是一种系统原语，是一条CPU的原子指令，从CPU层面保证它的原子性

**当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败，但失败的线程并不会被挂起，仅是被告知失败，并且允许再次尝试，当然也允许失败的线程放弃操作。**

**`实现原理`**：通过调用Unsafe类的一些native方法实现

```java
boolean compareAndSwapObject(Object o, long offset,Object expected, Object x);
boolean compareAndSwapInt(Object o, long offset,int expected,int x);
boolean compareAndSwapLong(Object o, long offset,long expected,long x);
```

当然，Unsafe类里面还有其它方法用于不同的用途。比如支持线程挂起和恢复的`park`和`unpark`， LockSupport类底层就是调用了这两个方法。还有支持反射操作的`allocateInstance()`方法。

**CAS实现原子操作的三大问题**

1. ABA问题，即就是一个值原来是A，变成了B，又变回了A。这个时候使用CAS是检查不出变化的，但实际上却被更新了两次。解决思路是：在变量前面追加上**版本号或者时间戳**。以`AtomicStampedReference`类为例，就会检查当前引用是否等于预期引用，并且检查当前标志是否等于预期标志。
2. 循环时间长开销大：长时间自旋而获取不到CPU会浪费大量资源。解决思路是：让JVM支持处理器提供的**pause指令**，让自旋失败时cpu睡眠一小段时间再继续自旋。
3. 只能保证一个共享变量的原子操作：

#### 原子操作

**`atomic包`**，下面的类的大概用途如下：

- 原子更新基本类型
- 原子更新数组
- 原子更新引用
- 原子更新字段（属性）

以`AtomicInteger`类的`getAndAdd(int delta)`方法为例，来看看Java是如何实现原子操作的。

```java
public final int getAndAdd(int delta) {
    return U.getAndAddInt(this, VALUE, delta);
}
```

U就是一个Unsafe对象：

```java
private static final jdk.internal.misc.Unsafe U = jdk.internal.misc.Unsafe.getUnsafe();
```

其实`AtomicInteger`类的`getAndAdd(int delta)`方法是调用`Unsafe`类的方法来实现的：

```java
@HotSpotIntrinsicCandidate
public final int getAndAddInt(Object o, long offset, int delta) {
    int v;
    do {
        v = getIntVolatile(o, offset);
    } while (!weakCompareAndSetInt(o, offset, v, v + delta));
    return v;
}
```

上述offset指获取某个字段相对于Java对象起始地址的偏移量

CAS是“无锁”的基础，它允许更新失败。所以经常会与while循环搭配，在失败后不断去重试。

这里声明了一个v，也就是要返回的值。从`getAndAddInt`来看，它返回的应该是原来的值，而新的值的`v + delta`。

这里使用的是**do-while循环**。这种循环不多见，它的目的是**保证循环体内的语句至少会被执行一遍**。这样才能保证return 的值`v`是我们期望的值。

### 5.AQS

> **AQS**是`AbstractQueuedSynchronizer`的简称，即`抽象队列同步器`，从字面意思上理解:
>
> - 抽象：抽象类，只实现一些主要逻辑，有些方法由子类实现；
> - 队列：使用先进先出（FIFO）队列存储数据；
> - 同步：实现了同步的功能。

AQS是用来构建锁和同步器的框架！

#### 数据结构

AQS内部使用了一个volatile的变量state来作为资源的标识。同时定义了几个获取和改版state的protected方法，子类可以覆盖这些方法来实现自己的逻辑：

都是原子操作

```java
getState()
setState()
compareAndSetState()
```

AQS类本身实现的是一些排队和阻塞的机制，比如具体线程等待队列的维护（如获取资源失败入队/唤醒出队等）。它内部使用了一个先进先出（FIFO）的双端队列，并使用了两个指针head和tail用于标识队列的头部和尾部。

![img](C:\Users\yato\Desktop\U know wt\interviewPre\dailyNote\笔记.assets\AQS数据结构.png)

本身不直接存储线程，而是存储拥有线程的Node节点

#### 资源共享模式

1. 独占模式：资源是独占的，一次只能一个线程获取。如ReentrantLock。
2. 共享模式：同时可以被多个线程获取，具体的资源个数可以通过参数指定。如Semaphore/CountDownLatch。

一般情况下，子类只需要根据需求实现其中一种模式，当然也有同时实现两种模式的同步类，如`ReadWriteLock`。

```java
static final class Node {
    // 标记一个结点（对应的线程）在共享模式下等待
    static final Node SHARED = new Node();
    // 标记一个结点（对应的线程）在独占模式下等待
    static final Node EXCLUSIVE = null; 

    // waitStatus的值，表示该结点（对应的线程）已被取消
    static final int CANCELLED = 1; 
    // waitStatus的值，表示后继结点（对应的线程）需要被唤醒
    static final int SIGNAL = -1;
    // waitStatus的值，表示该结点（对应的线程）在等待某一条件
    static final int CONDITION = -2;
    /*waitStatus的值，表示有资源可用，新head结点需要继续唤醒后继结点（共享模式下，多线程并发释放资源，而head唤醒其后继结点后，需要把多出来的资源留给后面的结点；设置新的head结点时，会继续唤醒其后继结点）*/
    static final int PROPAGATE = -3;

    // 等待状态，取值范围，-3，-2，-1，0，1
    volatile int waitStatus;
    volatile Node prev; // 前驱结点
    volatile Node next; // 后继结点
    volatile Thread thread; // 结点对应的线程
    Node nextWaiter; // 等待队列里下一个等待条件的结点


    // 判断共享模式的方法
    final boolean isShared() {
        return nextWaiter == SHARED;
    }

    Node(Thread thread, Node mode) {     // Used by addWaiter
        this.nextWaiter = mode;
        this.thread = thread;
    }

    // 其它方法忽略，可以参考具体的源码
}

// AQS里面的addWaiter私有方法
private Node addWaiter(Node mode) {
    // 使用了Node的这个构造函数
    Node node = new Node(Thread.currentThread(), mode);
    // 其它代码省略
}
```

不难看出，通过isShared方法可以判断当前节点是否出于共享模式

#### 主要方法

AQS基于模板方法实现，主要的方法有

- isHeldExclusively()：该线程是否正在独占资源。只有用到condition才需要去实现它。
- tryAcquire(int)：独占方式。尝试获取资源，成功则返回true，失败则返回false。
- tryRelease(int)：独占方式。尝试释放资源，成功则返回true，失败则返回false。
- tryAcquireShared(int)：共享方式。尝试获取资源。负数表示失败；0表示成功，但没有剩余可用资源；正数表示成功，且有剩余资源。
- tryReleaseShared(int)：共享方式。尝试释放资源，如果释放后允许唤醒后续等待结点返回true，否则返回false。

#### aquire方法

![img](C:\Users\yato\Desktop\U know wt\interviewPre\dailyNote\笔记.assets\acquire流程.jpg)



## 工具篇

### 1.线程池原理

#### 使用线程池的原因

1. 线程池可以**复用已创建的线程**
2. **控制并发的数量**
3. **可以对线程做统一管理**

#### 线程池参数

- **int corePoolSize**：该线程池中**核心线程数最大值**

	> 核心线程：线程池中有两类线程，核心线程和非核心线程。核心线程默认情况下会一直存在于线程池中，即使这个核心线程什么都不干（铁饭碗），而非核心线程如果长时间的闲置，就会被销毁（临时工）。

- **int maximumPoolSize**：该线程池中**线程总数最大值** 。

	> 该值等于核心线程数量 + 非核心线程数量。

- **long keepAliveTime**：**非核心线程闲置超时时长**。

	> 非核心线程如果处于闲置状态超过该值，就会被销毁。如果设置allowCoreThreadTimeOut(true)，则会也作用于核心线程。

- **TimeUnit unit**：keepAliveTime的单位。

TimeUnit是一个枚举类型 ，包括以下属性：

> NANOSECONDS ： 1微毫秒 = 1微秒 / 1000 MICROSECONDS ： 1微秒 = 1毫秒 / 1000 MILLISECONDS ： 1毫秒 = 1秒 /1000 SECONDS ： 秒 MINUTES ： 分 HOURS ： 小时 DAYS ： 天

- **BlockingQueue workQueue**：阻塞队列，维护着**等待执行的Runnable任务对象**。

	常用的几个阻塞队列：

	1. **LinkedBlockingQueue**

		链式阻塞队列，底层数据结构是链表，默认大小是`Integer.MAX_VALUE`，也可以指定大小。

	2. **ArrayBlockingQueue**

		数组阻塞队列，底层数据结构是数组，需要指定队列的大小。

		3.**SynchronousQueue**

		同步队列，内部容量为0，每个put操作必须等待一个take操作，反之亦然。

		4.**DelayQueue**

		延迟队列，该队列中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素 。

好了，介绍完5个必须的参数之后，还有两个非必须的参数。

- **ThreadFactory threadFactory**

	创建线程的工厂 ，用于批量创建线程，统一在创建线程时设置一些参数，如是否守护线程、线程的优先级等。如果不指定，会新建一个默认的线程工厂。

- **RejectedExecutionHandler handler**

	**拒绝处理策略**，线程数量大于最大线程数就会采用拒绝处理策略，四种拒绝处理的策略为 ：

	1. **ThreadPoolExecutor.AbortPolicy**：**默认拒绝处理策略**，丢弃任务并抛出RejectedExecutionException异常。
	2. **ThreadPoolExecutor.DiscardPolicy**：丢弃新来的任务，但是不抛出异常。
	3. **ThreadPoolExecutor.DiscardOldestPolicy**：丢弃队列头部（最旧的）的任务，然后重新尝试执行程序（如果再次失败，重复此过程）。
	4. **ThreadPoolExecutor.CallerRunsPolicy**：由调用线程处理该任务。

### 2.阻塞队列

#### 使用场景

生产者-消费者模式的实现，使用BlockingQueue实现

线程池中的参数

#### 常见实现类

1. ArrayBlockingQueue：**数组**结构组成的**有界**阻塞队列。内部结构是数组，故具有数组的特性。
2. LinkedBlockingQueue：由**链表**结构组成的**有界**阻塞队列。内部结构是链表，具有链表的特性。
3. DelayQueue：该队列中的元素只有当其指定的延迟时间到了，才能够从队列中获取到该元素 。注入其中的元素必须实现 java.util.concurrent.Delayed 接口。 
4. PriorityBlockingQueue：基于优先级的无界阻塞队列（优先级的判断通过构造函数传入的Compator对象来决定），内部控制线程同步的锁采用的是公平锁。
5. SynchronousQueue：**没有任何内部容量**，甚至连一个队列的容量都没有。**PriorityBlockingQueue**不会阻塞数据生产者（因为队列是无界的），而只会在没有可消费的数据时，阻塞数据的消费者。因此使用的时候要特别注意，**生产者生产数据的速度绝对不能快于消费者消费数据的速度，否则时间一长，会最终耗尽所有的可用堆内存空间。**

#### 原理

利用Lock锁的多条件（Condition，可以看作Object的wait/notify机制的增强）阻塞控制。

首先，构造器中有消费者和生产者监视器(Condition)，会标记当前线程身份。执行put操作的是生产者，执行take操作的是消费者

**具体put方法和take方法的执行核心要素**

1. put和take操作都需要**先获取锁**，没有获取到锁的线程会被挡在第一道大门之外自旋拿锁，直到获取到锁。
2. 就算拿到锁了之后，也**不一定**会顺利进行put/take操作，需要判断**队列是否可用**（是否满/空），如果不可用，则会被阻塞，**并释放锁**。
3. 在第2点被阻塞的线程会被唤醒，但是在唤醒之后，**依然需要拿到锁**才能继续往下执行，否则，自旋拿锁，拿到锁了再while判断队列是否可用（这也是为什么不用if判断，而使用while判断的原因）。