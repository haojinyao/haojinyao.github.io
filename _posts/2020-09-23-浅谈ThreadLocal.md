---
title: 浅谈ThreadLocal 
tags: [Java,多线程]
date: 2020-09-23 21:30:04
categories:
- Java
---


&emsp;&emsp;项目开发中，遇到了一个上传文件，之后进行归一化、关键信息提取和分析的场景。该任务可总结为需要在多个任务间传递消息，但是这些任务实例在多线程环境下运行。因此学习了一下多线程之间隔离的消息传递，于是看到了ThreadLocal。


&emsp;&emsp;ThreadLocal是一个本地线程副本变量工具类。主要用于将私有线程和该线程存放的副本对象做一个映射，各个线程之间的变量互不干扰，在高并发场景下，可以实现无状态的调用，特别适用于各个线程依赖不同的变量值完成操作的场景。（摘自<https://www.ilren.cn/article/Use-ThreadLocal-Transfer-Parameters.html>，我也找不出一个更好的描述方式了）

```java
public T get()

public void set(T value)

public void remove()
```
&emsp;&emsp;ThreadLocal主要提供以上三种方法，最常见的场景就是在前端请求时，在filter阶段将userId等参数set在当前线程为key的ThreadLocalMap中，在当前线程需要使用时则调用get()方法取出，最后在拦截器阶段将其remove掉，防止下次该线程获取到脏数据。

### ThreadLocal原理

&emsp;&emsp;Java中，Thread类有一个类型为ThreadLocal.ThreadLocalMap的实例变量threadLocals，即每个线程持有自己的ThreadLocalMap。ThreadLocalMap有自己的独立实现，可以简单地将它的key视作ThreadLocal，value为代码中放入的值（实际上key是ThreadLocal<?> k ，继承自WeakReference，是当前线程的一个弱引用）。每个线程在往ThreadLocal里放值的时候，都会往自己的ThreadLocalMap里存，读也是以ThreadLocal作为引用，在自己的map里找对应的key，从而实现了线程隔离。

&emsp;&emsp;关于ThreadLocal是弱引用的一些问题，在网上有很多相关分析，等以后有时间对四种引用做一次更深入的了解。

#### ThreadLocal Hash算法
&emsp;&emsp;ThreadLocalMap作为一个Map结构，不像HashMap是由数组+链表来实现，而是仅通过数组。当然，也要实现自己的hash算法来解决散列表数组冲突问题。
`int i = key.threadLocalHashCode & (len-1);`
i就是当前key在散列表中对应的数组下标位置,threadLocalHashCode的值由ThreadLocal中有一个属性HASH_INCREMENT = 0x61c88647得到：
```java
public class ThreadLocal<T> {
    private final int threadLocalHashCode = nextHashCode();

    private static AtomicInteger nextHashCode = new AtomicInteger();

    private static final int HASH_INCREMENT = 0x61c88647;

    private static int nextHashCode() {
        return nextHashCode.getAndAdd(HASH_INCREMENT);
    }

    static class ThreadLocalMap {
        ThreadLocalMap(ThreadLocal<?> firstKey, Object firstValue) {
            table = new Entry[INITIAL_CAPACITY];
            int i = firstKey.threadLocalHashCode & (INITIAL_CAPACITY - 1);

            table[i] = new Entry(firstKey, firstValue);
            size = 1;
            setThreshold(INITIAL_CAPACITY);
        }
    }
}
```
&emsp;&emsp;每当创建一个ThreadLocal对象，这个ThreadLocal.nextHashCode 这个值就会增长 0x61c88647 。这个值很特殊，它是**斐波那契数**，也叫黄金分割数。将hash增量设置为这个数字，带来的好处就是 hash 分布非常均匀。

&emsp;&emsp;但即使ThreadLocalMap中使用了黄金分隔数来作为hash计算因子，大大减少了Hash冲突的概率，但是仍然会存在冲突。HashMap中解决冲突的方法是在数组上构造一个链表结构，冲突的数据挂载到链表上，如果链表长度超过一定数量则会转化成红黑树。而ThreadLocalMap中并没有链表结构，所以这里不能适用HashMap解决冲突的方式了。

> 注明： 下面所有示例图中，绿色块Entry代表正常数据，灰色块代表Entry的key值为null，已被垃圾回收。白色块表示Entry为null。

![](https://raw.githubusercontent.com/haojinyao/pic/master/1600504035.png)

&emsp;&emsp;如上图所示，如果我们插入一个value=27的数据，通过hash计算后应该落入第4个槽位中，而槽位4已经有了Entry数据。

&emsp;&emsp;此时就会线性向后查找，一直找到Entry为null的槽位才会停止查找，将当前元素放入此槽位中。当然迭代过程中还有其他的情况，比如遇到了Entry不为null且key值相等的情况，还有Entry中的key值为null的情况等等都会有不同的处理，后面会一一详细讲解。

&emsp;&emsp;这里还画了一个Entry中的key为null的数据（Entry=2的灰色块数据），因为key值是弱引用类型，所以会有这种数据存在。在set过程中，如果遇到了key过期的Entry数据，实际上是会进行一轮探测式清理操作的，具体操作方式后面会讲到。

#### ThreadLocal.set()
&emsp;&emsp;ThreadLocal中的set方法逻辑，主要是判断ThreadLocalMap是否存在，然后使用ThreadLocal中的set方法进行数据处理。
```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}

void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```
&emsp;&emsp;往ThreadLocalMap中set数据（新增或者更新数据）分为好几四种情况：

1. 通过hash计算后的槽位对应的Entry数据为空：
   这里直接将数据放到该槽位即可。
2. 槽位数据不为空，key值与当前ThreadLocal通过hash计算获取的key值一致：
3. 槽位数据不为空，往后遍历过程中，在找到Entry为null的槽位之前，没有遇到key过期的Entry：
4. 槽位数据不为空，往后遍历过程中，在找到Entry为null的槽位之前，遇到key过期的Entry，如下图，往后遍历过程中，一到了index=7的槽位数据Entry的key=null（这种情况最复杂）：
   ![](https://raw.githubusercontent.com/haojinyao/pic/master/1600504767.png) 
         
&emsp;&emsp;散列数组下标为7位置对应的Entry数据key为null，表明此数据key值已经被垃圾回收掉了，此时就会执行replaceStaleEntry()方法，该方法含义是替换过期数据的逻辑，以index=7位起点开始遍历，进行探测式数据清理工作。
&emsp;&emsp;初始化探测式清理过期数据扫描的开始位置：slotToExpunge = staleSlot = 7。以当前staleSlot开始 向前迭代查找，找其他过期的数据，然后更新过期数据起始扫描下标slotToExpunge。for循环迭代，直到碰到Entry为null结束。如果找到了过期的数据，继续向前迭代，直到遇到Entry=null的槽位才停止迭代，如下图所示，slotToExpunge被更新为0：
   ![]( https://raw.githubusercontent.com/haojinyao/pic/master/1600506518.png) 
&emsp;&emsp;上面向前迭代的操作是为了更新探测清理过期数据的起始下标slotToExpunge的值，这个值在后面会讲解，它是用来判断当前过期槽位staleSlot之前是否还有过期元素。接着开始以staleSlot位置(index=7)向后迭代，如果找到了相同key值的Entry数据：
  ![]( https://raw.githubusercontent.com/haojinyao/pic/master/1600865207.png)
&emsp;&emsp;从当前节点staleSlot向后查找key值相等的Entry元素，找到后更新Entry的值并交换staleSlot元素的位置(staleSlot位置为过期元素)，更新Entry数据，然后开始进行过期Entry的清理工作，如下图所示：
  ![]( https://raw.githubusercontent.com/haojinyao/pic/master/1600865796.png)  
&emsp;&emsp;向后遍历过程中，如果没有找到相同key值的Entry数据，则从当前节点staleSlot向后查找key值相等的Entry元素，直到Entry为null则停止寻找。创建新的Entry，替换table[stableSlot]位置：
  ![]( https://raw.githubusercontent.com/haojinyao/pic/master/1600866032.png)  
&emsp;&emsp;替换完成后也是进行过期元素清理工作，清理工作主要是有两个方法：expungeStaleEntry()和cleanSomeSlots()
##### ThreadLocalMap过期key的探测式清理流程
&emsp;&emsp;ThreadLocalMap的有两种过期key数据清理方式：探测式清理和启发式清理。  
&emsp;&emsp;探测式清理，也就是expungeStaleEntry方法，遍历散列数组，从开始位置向后探测清理过期数据，将过期数据的Entry设置为null，沿途中碰到未过期的数据则将此数据rehash后重新在table数组中定位，如果定位的位置已经有了数据，则会将未过期的数据放到最靠近此位置的Entry=null的桶中，使rehash后的Entry数据距离正确的桶的位置更近一些。它属于线性探测清理。
&emsp;&emsp;启发式清理，即cleanSomeSlots()，则是跳跃性地探测过期数据，遇到则进行擦除：
  ![]( https://raw.githubusercontent.com/haojinyao/pic/master/1600867084.png) 
&emsp;&emsp;如图所示，第一个index每次向后移一格，第二个index每次除以2（16->8->4->2->1结束），如果遇到清理过期数据的情况，则重新将第二个index置为数组长度。源码如下：
```java
private boolean cleanSomeSlots(int i, int n) {
    boolean removed = false;
    Entry[] tab = table;
    int len = tab.length;
    do {
        i = nextIndex(i, len);
        Entry e = tab[i];
        if (e != null && e.get() == null) {
            n = len;
            removed = true;
            i = expungeStaleEntry(i);
        }
    } while ( (n >>>= 1) != 0);
    return removed;
}
```
#### ThreadLocalMap扩容机制
&emsp;&emsp;在ThreadLocalMap.set()方法的最后，如果执行完启发式清理工作后，未清理到任何数据，且当前散列数组中Entry的数量已经达到了列表的扩容阈值(len*2/3)，就开始执行rehash()逻辑：
```java
if (!cleanSomeSlots(i, sz) && sz >= threshold)
    rehash();
	
private void rehash() {
    expungeStaleEntries();

    if (size >= threshold - threshold / 4)
        resize();
}

private void expungeStaleEntries() {
    Entry[] tab = table;
    int len = tab.length;
    for (int j = 0; j < len; j++) {
        Entry e = tab[j];
        if (e != null && e.get() == null)
            expungeStaleEntry(j);
    }
}	
```
&emsp;&emsp;在rehash()中，首先是会进行探测式清理工作，从table的起始位置往后清理。清理完成之后，table中可能有一些key为null的Entry数据被清理掉，所以此时通过判断size >= threshold - threshold / 4 也就是size >= threshold* 3/4 来决定是否扩容。

#### ThreadLocalMap.get()详解
&emsp;&emsp;ThreadLocalMap.get()有两种情况。通过查找key值计算出散列表中slot位置，然后该slot位置中的Entry.key和查找的key一致，则直接返回。但若slot位置中的Entry.key和要查找的key不一致，就继续往后迭代查找，如果有过期数据就触发一次探测式数据回收操作，直到找到了key值相等的Entry数据并返回。
```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}

private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```
#### InheritableThreadLocal
&emsp;&emsp;我们使用ThreadLocal的时候，在异步场景下是无法给子线程共享父线程中创建的线程副本数据的。

&emsp;&emsp;为了解决这个问题，JDK中还有一个InheritableThreadLocal类。实现原理是子线程是通过在父线程中通过调用new Thread()方法来创建子线程，Thread#init方法在Thread的构造方法中被调用。在init方法中拷贝父线程数据到子线程中:
```java
private void init(ThreadGroup g, Runnable target, String name,
                      long stackSize, AccessControlContext acc,
                      boolean inheritThreadLocals) {
    if (name == null) {
        throw new NullPointerException("name cannot be null");
    }

    if (inheritThreadLocals && parent.inheritableThreadLocals != null)
        this.inheritableThreadLocals =
            ThreadLocal.createInheritedMap(parent.inheritableThreadLocals);
    this.stackSize = stackSize;
    tid = nextThreadID();
}
```
&emsp;&emsp;但InheritableThreadLocal仍然有缺陷，一般我们做异步化处理都是使用的线程池，而InheritableThreadLocal是在new Thread中的init()方法给赋值的，而线程池是线程复用的逻辑，所以这里会存在问题。针对这个问题，阿里巴巴开源了一个TransmittableThreadLocal组件，解决了这个问题。

### ThreadLocal应用
&emsp;&emsp;通过以上的分析，我们发现，ThreadLocal类的使用虽然是用来解决多线程的问题的，但是还是有很明显的针对性。最明显的，ThreadLoacl变量的活动范围为某线程，并且我的理解是该线程“专有的，独自霸占”，对该变量的所有操作均有该线程完成！也就是说，ThreadLocal不是用来解决共享，竞争问题的。典型的应用莫过于Spring，Hibernate等框架中对于多线程的处理了。

&emsp;&emsp;下面是hibernate中典型的ThreadLocal的应用：
```java
    private static final ThreadLocal threadSession = new ThreadLocal();    
        
    public static Session getSession() throws InfrastructureException {    
        Session s = (Session) threadSession.get();    
        try {    
            if (s == null) {    
                s = getSessionFactory().openSession();    
                threadSession.set(s);    
            }    
        } catch (HibernateException ex) {    
            throw new InfrastructureException(ex);    
        }    
        return s;    
    }
```
&emsp;&emsp;这段代码，每个线程有自己的ThreadLocalMap，每个ThreadLocalMap中根据需要初始加载threadSession,这样的好处就是介于singleton与prototype之间，应用singleton无法解决线程，应用prototype开销又太大，有了ThreadLocal之后就好了，对于需要线程“霸占”的变量用ThreadLocal，而该类实例的方法均可以共享。

&emsp;&emsp;此外，我们知道在一般情况下，只有无状态的Bean才可以在多线程环境下共享，在Spring中，绝大部分Bean都可以声明为singleton作用域。就是因为Spring对一些Bean（如RequestContextHolder、TransactionSynchronizationManager、LocaleContextHolder等）中非线程安全的“状态性对象”采用ThreadLocal进行封装，让它们也成为线程安全的“状态性对象”，因此有状态的Bean就能够以singleton的方式在多线程中正常工作了。

&emsp;&emsp;一般的Web应用划分为展现层、服务层和持久层三个层次，在不同的层中编写对应的逻辑，下层通过接口向上层开放功能调用。在一般情况下，从接收请求到返回响应所经过的所有程序调用都同属于一个线程。这样用户就可以根据需要，将一些非线程安全的变量（比如Connection）以ThreadLocal存放，在同一次请求响应的调用线程中，所有对象所访问的同一ThreadLocal变量都是当前线程所绑定的。

&emsp;&emsp;在使用ThreadLocal时，很多人会发现内存泄漏：虽然ThreadLocalMap已经使用了weakReference，但是还是建议能够显示的使用remove方法。

### 与Thread同步机制的比较
&emsp;&emsp;ThreadLocal和线程同步机制相比有什么优势呢？ThreadLocal和线程同步机制都是为了解决多线程中相同变量的访问冲突问题。

&emsp;&emsp;在同步机制中，通过对象的锁机制保证同一时间只有一个线程访问变量。这时该变量是多个线程共享的，使用同步机制要求程序缜密地分析什么时候对变量进行读写，什么时候需要锁定某个对象，什么时候释放对象锁等繁杂的问题，程序设计和编写难度相对较大。

&emsp;&emsp;而ThreadLocal则从另一个角度来解决多线程的并发访问。ThreadLocal为每一个线程提供一个独立的变量副本，从而隔离了多个线程对访问数据的冲突。因为每一个线程都拥有自己的变量副本，从而也就没有必要对该变量进行同步了。ThreadLocal提供了线程安全的对象封装，在编写多线程代码时，可以把不安全的变量封装进ThreadLocal。

&emsp;&emsp;由于ThreadLocal中可以持有任何类型的对象，低版本JDK所提供的get()返回的是Object对象，需要强制类型转换。但JDK 5.0通过泛型很好的解决了这个问题，在一定程度上简化ThreadLocal的使用。

&emsp;&emsp;概括起来说，对于多线程资源共享的问题，同步机制采用了“以时间换空间”的方式：访问串行化，对象共享化。而ThreadLocal采用了“以空间换时间”的方式：访问并行化，对象独享化。前者仅提供一份变量，让不同的线程排队访问，而后者为每一个线程都提供了一份变量，因此可以同时访问而互不影响。
  
  
  

&emsp;&emsp;拖更了许久，仅拼凑出这样一篇文章。纵然可能忙于很多事，但不可否认、不可推卸的是偷懒了。写到后面，也懒得转述为自己的语句了。总之，确实是做的很不好。以后宁愿不做很深入的研究分析，也要保持习惯，争取多更新。

摘自：  
1.<https://mp.weixin.qq.com/s/z-7y1FBAvy-7wC2_jndbkw>  
2.<https://blog.51cto.com/2179425/2082743>