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
            ++sshift;
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



所以我们不能直接调用，那怎么用呢？使用反射



