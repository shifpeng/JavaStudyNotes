

### ConCurrentHashMap(JDK1.7)

我们联想到HashTable，我们发现它使用了```synchronized```

```java
    /**
     * Maps the specified <code>key</code> to the specified
     * <code>value</code> in this hashtable. Neither the key nor the
     * value can be <code>null</code>. <p>
     *
     * The value can be retrieved by calling the <code>get</code> method
     * with a key that is equal to the original key.
     *
     * @param      key     the hashtable key
     * @param      value   the value
     * @return     the previous value of the specified key in this hashtable,
     *             or <code>null</code> if it did not have one
     * @exception  NullPointerException  if the key or value is
     *               <code>null</code>
     * @see     Object#equals(Object)
     * @see     #get(Object)
     */
    public synchronized V put(K key, V value) {
        // Make sure the value is not null
        if (value == null) {
            throw new NullPointerException();
        }

        // Makes sure the key is not already in the hashtable.
        Entry tab[] = table;
        int hash = hash(key);
        int index = (hash & 0x7FFFFFFF) % tab.length;
        for (Entry<K,V> e = tab[index] ; e != null ; e = e.next) {
            if ((e.hash == hash) && e.key.equals(key)) {
                V old = e.value;
                e.value = value;
                return old;
            }
        }

        modCount++;
        if (count >= threshold) {
            // Rehash the table if the threshold is exceeded
            rehash();

            tab = table;
            hash = hash(key);
            index = (hash & 0x7FFFFFFF) % tab.length;
        }

        // Creates the new entry.
        Entry<K,V> e = tab[index];
        tab[index] = new Entry<>(hash, key, value, e);
        count++;
        return null;
    }
```

如果对整个hashmap对象加锁的话，并发安全问题可能是解决了，但是效率就非常的低了 ；



ConCurrentHashMap:里面的对象就不是Entry了，而是Segment [] table;而Segment中有个属性HashEntry[] tab;

```java
    /* ---------------- Constants -------------- */

    /**
     * The default initial capacity for this table,
     * used when not otherwise specified in a constructor.
     */
    static final int DEFAULT_INITIAL_CAPACITY = 16;  //默认的数组容量大小

    /**
     * The default load factor for this table, used when not
     * otherwise specified in a constructor.
     */
    static final float DEFAULT_LOAD_FACTOR = 0.75f; //加载因子

    /**
     * The default concurrency level for this table, used when not
     * otherwise specified in a constructor.
     */
    static final int DEFAULT_CONCURRENCY_LEVEL = 16;  //并发级别
```



每个Segment有几个HashEntry实际上就是拿着两个参数计算出来的 ```initialCapacity/concurrencyLevel```,当然原理是这样的，其实源码中会稍微有些不一样：

```java
 /**
     * Creates a new, empty map with the specified initial
     * capacity, load factor and concurrency level.
     *
     * @param initialCapacity the initial capacity. The implementation
     * performs internal sizing to accommodate this many elements.
     * @param loadFactor  the load factor threshold, used to control resizing.
     * Resizing may be performed when the average number of elements per
     * bin exceeds this threshold.
     * @param concurrencyLevel the estimated number of concurrently
     * updating threads. The implementation performs internal sizing
     * to try to accommodate this many threads.
     * @throws IllegalArgumentException if the initial capacity is
     * negative or the load factor or concurrencyLevel are
     * nonpositive.
     */
    @SuppressWarnings("unchecked")
    public ConcurrentHashMap(int initialCapacity,
                             float loadFactor, int concurrencyLevel) {
        if (!(loadFactor > 0) || initialCapacity < 0 || concurrencyLevel <= 0)
            throw new IllegalArgumentException();
        if (concurrencyLevel > MAX_SEGMENTS)  //限制并发级别最大为2的12次幂
            concurrencyLevel = MAX_SEGMENTS;
        // Find power-of-two sizes best matching arguments
        int sshift = 0;
        int ssize = 1;  //Segment数组的大小
      //找大于等于并发级别的2的幂次方数
        while (ssize < concurrencyLevel) {
            ++sshift;   //统计1左移的次数
            ssize <<= 1;
        }
        this.segmentShift = 32 - sshift;
        this.segmentMask = ssize - 1;
        if (initialCapacity > MAXIMUM_CAPACITY)
            initialCapacity = MAXIMUM_CAPACITY;
        int c = initialCapacity / ssize;       //HashEntry的size
        if (c * ssize < initialCapacity)  //比如initialCapacity=17 ，ssize=16 c=1,但是明显不够存了，所以这里会判断c * ssize < initialCapacity，++c
            ++c;
        int cap = MIN_SEGMENT_TABLE_CAPACITY;
        while (cap < c)
            cap <<= 1;  //最终还是要以2的幂次方数作为数据的容量
        // create segments and segments[0]
        Segment<K,V> s0 =
            new Segment<K,V>(loadFactor, (int)(cap * loadFactor),
                             (HashEntry<K,V>[])new HashEntry[cap]);
        Segment<K,V>[] ss = (Segment<K,V>[])new Segment[ssize];
        UNSAFE.putOrderedObject(ss, SBASE, s0); // ordered write of segments[0]
        this.segments = ss;
    }
```

####  ``` UNSAFE```

先看一个例子：两个线程对同一个对象进行并发操作，如果是线程安全的话，那么打印出来的值肯定是不会重复的；但是我们会发现，打印出来的值是重复的，也就是线程不安全的

```java
public class Person {
    private Integer i = 0;

    public static void main(String[] args) {
        final Person person = new Person();
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    person.i++;
                    System.out.println(person.i);
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    person.i++;
                    System.out.println(person.i);
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}


// 控制台打印出来的值为以下：
1
2
3
4
5
6
7
8
9
10
11
11
12
12
13
13
```

解决这个问题：出现这个问题就是因为没有使用内存中的变量，而是使用的是线程中拷贝的变量；

我们 尝试使用unsafe解决这个问题，看ConCurrentHashMap源码中我们发现，他们是这样使用UNSAFE的：

```
    // Unsafe mechanics
    private static final sun.misc.Unsafe UNSAFE;
    
        static {
        int ss, ts;
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
    			.......
    }
```

我们也在Person中使用试试：

```java
public class Person {
    private Integer i = 0;
    private static  sun.misc.Unsafe UNSAFE;

    static {
        UNSAFE = sun.misc.Unsafe.getUnsafe();
    }
```

我们 运行会发现报错：

```java
Exception in thread "main" java.lang.ExceptionInInitializerError
Caused by: java.lang.SecurityException: Unsafe
	at sun.misc.Unsafe.getUnsafe(Unsafe.java:90)
```

看源码我们发现是在UNsafe类的这里抛出的：

```java
    @CallerSensitive
    public static Unsafe getUnsafe() {
        Class var0 = Reflection.getCallerClass();  //获取当前是那个类使用的unsafe这个类，我们本次是person
        if (var0.getClassLoader())  //这里会判断person的类加载器是啥，是application，ConCurrentHashMap的类加载器是boot，boot在这里会返回null
            throw new SecurityException("Unsafe");
        return theUnsafe;
    }
```



所以我们不能直接调用，那怎么用呢？使用反射:

```java
import sun.misc.Unsafe;

import java.lang.reflect.Field;

public class Person {
    private int i = 0;
    private static sun.misc.Unsafe UNSAFE;
    private static long I_OFFSET;

    static {
        Field field = null;
        try {
            field = Unsafe.class.getDeclaredField("theUnsafe");
            field.setAccessible(true);
            UNSAFE = (Unsafe) field.get(null);
            I_OFFSET = UNSAFE.objectFieldOffset(Person.class.getDeclaredField("i"));  //计算对象的偏移量  获取person对象的i属性的偏移量

        } catch (NoSuchFieldException | IllegalAccessException e) {
            e.printStackTrace();
        }

    }

    public static void main(String[] args) {
        final Person person = new Person();
        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
//                    person.i++;
                    boolean b = UNSAFE.compareAndSwapInt(person, I_OFFSET, person.i, person.i + 1);  //如果当前值正好等于person.i，那么+1
                    if (b) {
                        System.out.println(UNSAFE.getIntVolatile(person, I_OFFSET));
                    }
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();

        new Thread(new Runnable() {
            @Override
            public void run() {
                while (true) {
                    boolean b = UNSAFE.compareAndSwapInt(person, I_OFFSET, person.i, person.i + 1);  //如果当前值正好等于person.i，那么+1
                    if (b) {
                        System.out.println(UNSAFE.getIntVolatile(person, I_OFFSET));  //相当于直接从内存中拿person.i的值
                    }
                    try {
                        Thread.sleep(500);
                    } catch (InterruptedException e) {
                        e.printStackTrace();
                    }
                }
            }
        }).start();
    }
}


//就会从1开始正常打印

```



#### ConCurrentHashMap的put



```java
    /**
     * Maps the specified key to the specified value in this table.
     * Neither the key nor the value can be null.
     *
     * <p> The value can be retrieved by calling the <tt>get</tt> method
     * with a key that is equal to the original key.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>
     * @throws NullPointerException if the specified key or value is null
     */
    @SuppressWarnings("unchecked")
    public V put(K key, V value) {
        Segment<K,V> s;
        if (value == null)
            throw new NullPointerException();
        int hash = hash(key);   //计算出一个hash值
        int j = (hash >>> segmentShift) & segmentMask;  //确定放在数组的什么位置   segmentMask=segment大小-1 
        if ((s = (Segment<K,V>)UNSAFE.getObject          // nonvolatile; recheck
             (segments, (j << SSHIFT) + SBASE)) == null) //  查看segment的第J个位置的值是不是为null
          
            s = ensureSegment(j);  //生成segment对象
        return s.put(key, hash, value, false);
    }

    /**
     * Applies a supplemental hash function to a given hashCode, which
     * defends against poor quality hash functions.  This is critical
     * because ConcurrentHashMap uses power-of-two length hash tables,
     * that otherwise encounter collisions for hashCodes that do not
     * differ in lower or upper bits.
     */
    private int hash(Object k) {
        int h = hashSeed;

        if ((0 != h) && (k instanceof String)) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();

        // Spread bits to regularize both segment and index locations,
        // using variant of single-word Wang/Jenkins hash.
        h += (h <<  15) ^ 0xffffcd7d;
        h ^= (h >>> 10);
        h += (h <<   3);
        h ^= (h >>>  6);
        h += (h <<   2) + (h << 14);
        return h ^ (h >>> 16);
    }
```

查看segment的第```j```（数组下标）个位置的元素

```java
UNSAFE.getObject(segments, (j << SSHIFT) + SBASE))
  
  
查看源码发现：SSHIFT = 31 - Integer.numberOfLeadingZeros(ss);
所以上面的代码变成更：
UNSAFE.getObject(segments, (j << (31 - Integer.numberOfLeadingZeros(ss)) + SBASE))

Integer.numberOfLeadingZeros(ss);  //该方法返回二进制中最高位的1前面的0 的个数  ，即高位的0 的个数
```

这里主要是为了保证在new Segment的时候是安全的，使用的是cas，就是一种乐观锁的思维

```
悲观锁

总是假设最坏的情况，每次去拿数据的时候都认为别人会修改，所以每次在拿数据的时候都会上锁，这样别人想拿这个数据就会阻塞直到它拿到锁（共享资源每次只给一个线程使用，其它线程阻塞，用完后再把资源转让给其它线程）。传统的关系型数据库里边就用到了很多这种锁机制，比如行锁，表锁等，读锁，写锁等，都是在做操作之前先上锁。Java中synchronized和ReentrantLock等独占锁就是悲观锁思想的实现。

乐观锁

总是假设最好的情况，每次去拿数据的时候都认为别人不会修改，所以不会上锁，但是在更新的时候会判断一下在此期间别人有没有去更新这个数据，可以使用版本号机制和CAS算法实现。乐观锁适用于多读的应用类型，这样可以提高吞吐量，像数据库提供的类似于write_condition机制，其实都是提供的乐观锁。在Java中java.util.concurrent.atomic包下面的原子变量类就是使用了乐观锁的一种实现方式CAS实现的。

两种锁的使用场景

从上面对两种锁的介绍，我们知道两种锁各有优缺点，不可认为一种好于另一种，像乐观锁适用于写比较少的情况下（多读场景），即冲突真的很少发生的时候，这样可以省去了锁的开销，加大了系统的整个吞吐量。但如果是多写的情况，一般会经常产生冲突，这就会导致上层应用会不断的进行retry，这样反倒是降低了性能，所以一般多写的场景下用悲观锁就比较合适。
————————————————
原文链接：https://blog.csdn.net/qq_34337272/java/article/details/81072874
```





```java
    /**
     * Returns the segment for the given index, creating it and
     * recording in segment table (via CAS) if not already present.
     *
     * @param k the index
     * @return the segment
     */
    @SuppressWarnings("unchecked")
    private Segment<K,V> ensureSegment(int k) {
        final Segment<K,V>[] ss = this.segments;
        long u = (k << SSHIFT) + SBASE; // raw offset
        Segment<K,V> seg;
        if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u)) == null) {  //取当前位置的元素，如果有的话，说明另外的线程已经生成出来了，直接返回
            Segment<K,V> proto = ss[0]; // use segment 0 as prototype  //直接拿concurrenthashMap在初始化的时候在第0个位置生成的segment对象
            int cap = proto.table.length;
            float lf = proto.loadFactor;
            int threshold = (int)(cap * lf);
            HashEntry<K,V>[] tab = (HashEntry<K,V>[])new HashEntry[cap];
            if ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))  //再次判断当前位置还是不是空，如果还是空，真正的new Segment对象
                == null) { // recheck
                Segment<K,V> s = new Segment<K,V>(lf, threshold, tab);
                while ((seg = (Segment<K,V>)UNSAFE.getObjectVolatile(ss, u))  //失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次发起尝试
                       == null) {
                    if (UNSAFE.compareAndSwapObject(ss, u, null, seg = s))  //cas  对数组的第U个位置进行赋值，如果该位置的值不为null,则赋值
                        break;
                }
            }
        }
        return seg;
    }
```

对CAS的理解，CAS是一种无锁算法，CAS有3个操作数，内存值V，旧的预期值A，要修改的新值B。当且仅当预期值A和内存值V相同时，将内存值V修改为B，否则什么都不做。

```
注：t1，t2线程是同时更新同一变量56的值

因为t1和t2线程都同时去访问同一变量56，所以他们会把主内存的值完全拷贝一份到自己的工作内存空间，所以t1和t2线程的预期值都为56。

假设t1在与t2线程竞争中线程t1能去更新变量的值，而其他线程都失败。（失败的线程并不会被挂起，而是被告知这次竞争中失败，并可以再次发起尝试）。t1线程去更新变量值改为57，然后写到内存中。此时对于t2来说，内存值变为了57，与预期值56不一致，就操作失败了（想改的值不再是原来的值）。

（通俗的解释是：CPU去更新一个值，但如果想改的值不再是原来的值，操作就失败，因为很明显，有其它操作先改变了这个值。）

就是指当两者进行比较时，如果相等，则证明共享数据没有被修改，替换成新值，然后继续往下运行；如果不相等，说明共享数据已经被修改，放弃已经所做的操作，然后重新执行刚才的操作。容易看出 CAS 操作是基于共享数据不会被修改的假设，采用了类似于数据库的commit-retry 的模式。当同步冲突出现的机会很少时，这种假设能带来较大的性能提升。
```

以上对象创建好之后，就会执行put方法

```java
final V put(K key, int hash, V value, boolean onlyIfAbsent) {
  		   //第一步：加锁 ，可重入锁（尝试去加锁 ）获取到立马返回true，获取不到，立马返回fase，不会阻塞； 区别于lock():获取不到，就会阻塞，直到获取到
            HashEnputtry<K,V> node = tryLock() ? null :
                scanAndLockForPut(key, hash, value);
            V oldValue;
            try {
                HashEntry<K,V>[] tab = table;
                int index = (tab.length - 1) & hash;
                HashEntry<K,V> first = entryAt(tab, index); //查询该位置的元素
                for (HashEntry<K,V> e = first;;) {
                    if (e != null) {
                        K k;
                        if ((k = e.key) == key ||
                            (e.hash == hash && key.equals(k))) {
                            oldValue = e.value;
                            if (!onlyIfAbsent) {
                                e.value = value;
                                ++modCount;
                            }
                            break;
                        }
                        e = e.next;
                    }
                    else {  //不存在
                        if (node != null)
                            node.setNext(first);
                        else
                            node = new HashEntry<K,V>(hash, key, value, first);  //头插法
                        int c = count + 1;
                        if (c > threshold && tab.length < MAXIMUM_CAPACITY)
                            rehash(node);
                        else
                            setEntryAt(tab, index, node);
                        ++modCount;
                        count = c;
                        oldValue = null;
                        break;
                    }
                }
            } finally {
                unlock();  //释放锁
            }
            return oldValue;
        }
```

我们最终的目的是要把key-value 存到segment中的hashEntry中，可以这样理解，segment的内部也是个小的hashmap，也是由链表+数组组成的

在进行数据插入的时候，就会先加锁处理

```java
 /**
         * Scans for a node containing given key while trying to
         * acquire lock, creating and returning one if not found. Upon
         * return, guarantees that lock is held. UNlike in most
         * methods, calls to method equals are not screened: Since
         * traversal speed doesn't matter, we might as well help warm
         * up the associated code and accesses as well.
         *
         * @return a new node if key not found, else null
         */
        private HashEntry<K,V> scanAndLockForPut(K key, int hash, V value) {
            HashEntry<K,V> first = entryForHash(this, hash);
            HashEntry<K,V> e = first;
            HashEntry<K,V> node = null;
            int retries = -1; // negative while locating node
            while (!tryLock()) {
                HashEntry<K,V> f; // to recheck first below
                if (retries < 0) {
                    if (e == null) {
                        if  (node == null) // speculatively create node   
                            node = new HashEntry<K,V>(hash, key, value, null);  //尝试获取锁，没有获取到的时候，先实例化HashEntry
                        retries = 0;
                    }
                    else if (key.equals(e.key))
                        retries = 0;
                    else
                        e = e.next;
                }
                else if (++retries > MAX_SCAN_RETRIES) {
                    lock();
                    break;
                }
              //这里表示在尝试获取锁的时候，需要判断遍历到该链表的当前节点数据有没有发生改变
                else if ((retries & 1) == 0 &&
                         (f = entryForHash(this, hash)) != first) {
                    e = first = f; // re-traverse if entry changed
                    retries = -1;
                }
            }
            return node;
        }

```

下面的代码演示lock的时候，线程2在尝试获取锁的时候就会阻塞住，知道获取到锁（等待5s）

```java
  public static void main(String[] args) {
        ReentrantLock reentrantLock = new ReentrantLock();

        new Thread(new Runnable() {
            @Override
            public void run() {
                reentrantLock.lock();
                System.out.printf("线程1获取到了锁");
                try {
                    Thread.sleep(5000);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                reentrantLock.unlock();
            }
        }).start();
		
        
        //这里保证线程1先获得锁
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    
    
        new Thread(new Runnable() {
            @Override
            public void run() {
                reentrantLock.lock();  //这里使用lock的时候，由于没有获取到锁，就会阻塞，那这个时候什么也做不了
                System.out.printf("线程2获取到了锁");
                reentrantLock.unlock();
            }
        }).start();

    }
```

如果我们把上面的第二个线程修改一下：

```java
         new Thread(new Runnable() {
            @Override
            public void run() {
//                reentrantLock.lock();  //lock的阻塞状态下，是不消耗cpu的
               while (!reentrantLock.tryLock())  //使用while是消耗cpu
               {
                    //但是这个方法的好处是，在阻塞状态下，可以做其他的事情
               }
                System.out.printf("线程2获取到了锁");
                reentrantLock.unlock();
            }
        }).start();
```

在ConcurrentHashMap中，遍历链表的每个节点的时候，都会尝试获取锁，而这里的锁并不是一直阻塞的，如果没有key相等的，他会先去new HashEntry()

