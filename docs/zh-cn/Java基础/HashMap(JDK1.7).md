## JDK1.7 HashMap

由数组+链表实现

1、通过hashCode找到数组中的元素，

2、然后再通过key的equals方法在链表中找打key对应的value

![](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/jdk1.7数组+链表图解.png)



> 我们一般在HashMap中存储数据，是以key-value的形式存储，看源码知道实际上会把key、value封装成了一个Entry对象。实际上HashMap中的数组和链表上存储的就是Entry对象

我们知道存储数组的时候，都是需要指定下标的。比如ArrayList：

```java
ArrayList arrayList = new ArrayList()；
arrayList.add(1, new Object());
arrayList.add(new Object());
//这里的add方法提供了两个add方法，一个是带下标的，一个是不带的，其实看了源码就知道，不带下标的，实际上是在add的时候，是通过arrayList的size++l来确定下标的
```
我们听过说Hashmap的新增操作比较块，而查询（get）操作比较慢，而ArrayList则相反，查询效率比较高，就是因为HashMap是没有类似于数组的下标的，而是put时候设置的key

```java
    public static void main(String[] args) {
        HashMap<String, String> hashMap = new HashMap<>();
        hashMap.put("1", "1");
        String oldVaue = hashMap.put("1", "2");
        String newValue = hashMap.get("1");

        System.out.printf("旧值为" + oldVaue + "\n");
        System.out.printf("新值为" + newValue);
    }
```


#### 如何存放

Key的存放，转化成hashCode，但是这个数字就会很大，所以这里会对这个数字进行取余操作 hashCode % 数组的length，在保证计算出来的这个key在0到数组的length-1之间，也要保证最后的值是平均的落在这些值，而不是某个值永远都不出现的情况
源码中是这样实现的

```java
    static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```
但是无法保证取余之后的所有的key都是不一样的，所以当hashCode重复时，就用链表存储

**所以，存储hashMap即要么存储到数组上，要么存储到链表上**

#### 链表 所谓的链表，其实也很简单 ，下面实现的就是一个链表

```Java

public class Node {
    private Object content;

    private Node next;

    public Node(Object content, Node next) {
        this.content = content;
        this.next = next;
    }

    public static void main(String[] args) {
        Node header = new Node(new Object(), null);
        header.next = new Node(new Object(), null);  //尾节点的next属性为null
    }
}
```

如果在存放的时候，经过计算后的key发现是一样的，因此要存放在链表上，那这个时候存放在链表的头部块还是尾部块呢？，依然是上面的例子
```Java
    public static void main(String[] args) {
        Node header = new Node(new Object(), null);
        header.next = new Node(new Object(), null);  //保存在尾部 
        new Node(new Object(), header);  //插入在头部
    }
```
因为如果想插在链表的尾部，首先要先找到链表的尾部，智能从header节点循环遍历取查找，所以会比较慢


#### get

在获取我们需要的数据时，使用了头插法的话，我们在get值的时候就会发现取不到，因为HashMap中的链表只有next,没有pre，这个时候呢，如果我们把链表的整体元素全部向下移动一位，就可以了

```java
//put大概的的实现思路
put(key,value)
{
int hashCode=key.hashCode();
int index=hashCode%table.lengtj;
//Entr（key,value,next）  //这里说得这些都是指对象的应用  就是把两个index一样的存在同一个地址
table[index]=new Entry(key,value,table[index]);  //将table[index]插入到头节点

}

```

JDK1.7下 HashMap的put源码

```java
    /**
     * Associates the specified value with the specified key in this map.
     * If the map previously contained a mapping for the key, the old
     * value is replaced.
     *
     * @param key key with which the specified value is to be associated
     * @param value value to be associated with the specified key
     * @return the previous value associated with <tt>key</tt>, or
     *         <tt>null</tt> if there was no mapping for <tt>key</tt>.
     *         (A <tt>null</tt> return can also indicate that the map
     *         previously associated <tt>null</tt> with <tt>key</tt>.)
     */
    public V put(K key, V value) {
        if (table == EMPTY_TABLE) {
            inflateTable(threshold);  //初始化数组
        }
        if (key == null)
            return putForNullKey(value);
        int hash = hash(key);    //这里也不只是简单的取hashCode
        int i = indexFor(hash, table.length);  //计算数据的下标
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {  //从头节点开始便利 
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {  //相同的key的情况下会覆盖
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        modCount++;
        addEntry(hash, key, value, i);
        return null;
    }


    private static int roundUpToPowerOf2(int number) {
        // assert number >= 0 : "number must be non-negative";
        return number >= MAXIMUM_CAPACITY
                ? MAXIMUM_CAPACITY
                : (number > 1) ? Integer.highestOneBit((number - 1) << 1) : 1;
    }

    /**
     * Inflates the table.   初始化数组
     */
    private void inflateTable(int toSize) {
        // Find a power of 2 >= toSize
        int capacity = roundUpToPowerOf2(toSize);

        threshold = (int) Math.min(capacity * loadFactor, MAXIMUM_CAPACITY + 1);
        table = new Entry[capacity];
        initHashSeedAsNeeded(capacity);
    }

    /**
     * Retrieve object hash code and applies a supplemental hash function to the
     * result hash, which defends against poor quality hash functions.  This is
     * critical because HashMap uses power-of-two length hash tables, that
     * otherwise encounter collisions for hashCodes that do not differ
     * in lower bits. Note: Null keys always map to hash 0, thus index 0.
     */
    final int hash(Object k) {
        int h = hashSeed;
        if (0 != h && k instanceof String) {
            return sun.misc.Hashing.stringHash32((String) k);
        }

        h ^= k.hashCode();
				//如果两个hashCode的二进制，只有一位是不同的，那么直接调用indexFor做h & (length-1)运算后，那么计算出来的下标就是一样的
      
        // This function ensures that hashCodes that differ only by
        // constant multiples at each bit position have a bounded
        // number of collisions (approximately 8 at default load factor).
      
      
      //让key的hashCode的二进制中高位参与到运算中来，减少数组下标一致导致的链表过长的问题
      //即让生成的hash值更散列一点
        h ^= (h >>> 20) ^ (h >>> 12);   
        return h ^ (h >>> 7) ^ (h >>> 4);
    }



    /**
     * Offloaded version of put for null keys
     * 如果key为null的话，这个值会被存在hashmap的第0个位置
     */
    private V putForNullKey(V value) {
        for (Entry<K,V> e = table[0]; e != null; e = e.next) {
            if (e.key == null) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }
        modCount++;
        addEntry(0, null, value, 0);
        return null;
    }
```

```java
    /**  Integer 的方法
     * Returns an {@code int} value with at most a single one-bit, in the
     * position of the highest-order ("leftmost") one-bit in the specified
     * {@code int} value.  Returns zero if the specified value has no
     * one-bits in its two's complement binary representation, that is, if it
     * is equal to zero.
     *
     * @return an {@code int} value with a single one-bit, in the position
     *     of the highest-order one-bit in the specified value, or zero if
     *     the specified value is itself equal to zero.
     * @since 1.5
     */
    public static int highestOneBit(int i) {
        // HD, Figure 3-1
        i |= (i >>  1);
        i |= (i >>  2);
        i |= (i >>  4);
        i |= (i >>  8);
        i |= (i >> 16);
        return i - (i >>> 1);
    }
```

```java

    /**
     * 计算数组的下标 h:哈希值；length:数组的length
     * Returns index for hash code h.
     */
    static int indexFor(int h, int length) {
        // assert Integer.bitCount(length) == 1 : "length must be a non-zero power of 2";
        return h & (length-1);
    }

//思路：保证两点：
//1、不能越界，比如，数组的大小是16，那么下标就只能是0～15；
//2、0～15这16个数字出现的频率是平均的（即不能某些下标一直取不到的情况），我们想到取余操作是可以满足这个条件的，但是没有位运算快

//但是上面没用取余操作，我们测试下
比如hashCode是 
h: 0101 0101
length(16): 0001 0000  
--进行计算--- h & (length-1);
h：0101 0101
15: 0000 1111  
&：都为1则为1
0000 0101 
  
我们发现得到的值就是hashCode的1-4位 ，而1-4位的取值范围为0000～1111 即 0～15 满足第一点
又因为hashCode就是随机的，所以满足第二点；
  
  
所以我们说为什么一个数组的容量一定是2的幂次方，就是为了方便后面的计算下标


```

#### 扩容

扩容是针对数组，而不是链表

```java
    /**
     * Adds a new entry with the specified key, value and hash code to
     * the specified bucket.  It is the responsibility of this
     * method to resize the table if appropriate.
     *
     * Subclass overrides this to alter the behavior of put method.
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {
      //threshold：预值 =table.length*负载因子
      
        if ((size >= threshold) && (null != table[bucketIndex])) {
          	
            resize(2 * table.length); 
            hash = (null != key) ? hash(key) : 0;
            bucketIndex = indexFor(hash, table.length);
        }

        createEntry(hash, key, value, bucketIndex);
    }


    /**
     * Like addEntry except that this version is used when creating entries
     * as part of Map construction or "pseudo-construction" (cloning,
     * deserialization).  This version needn't worry about resizing the table.
     *
     * Subclass overrides this to alter the behavior of HashMap(Map),
     * clone, and readObject.
     */
    void createEntry(int hash, K key, V value, int bucketIndex) {
        Entry<K,V> e = table[bucketIndex];
        table[bucketIndex] = new Entry<>(hash, key, value, e);
        size++;
    }

    /**
     * Rehashes the contents of this map into a new array with a
     * larger capacity.  This method is called automatically when the
     * number of keys in this map reaches its threshold.
     *
     * If current capacity is MAXIMUM_CAPACITY, this method does not
     * resize the map, but sets threshold to Integer.MAX_VALUE.
     * This has the effect of preventing future calls.
     *
     * @param newCapacity the new capacity, MUST be a power of two;
     *        must be greater than current capacity unless current
     *        capacity is MAXIMUM_CAPACITY (in which case value
     *        is irrelevant).
     */
    void resize(int newCapacity) {
        Entry[] oldTable = table;
        int oldCapacity = oldTable.length;
        if (oldCapacity == MAXIMUM_CAPACITY) {
            threshold = Integer.MAX_VALUE;
            return;
        }

        Entry[] newTable = new Entry[newCapacity];
      	//将老的数组中的数据转移到新的数组
        transfer(newTable, initHashSeedAsNeeded(newCapacity));
        table = newTable;
        threshold = (int)Math.min(newCapacity * loadFactor, MAXIMUM_CAPACITY + 1);
    }

    /**
     * Transfers all entries from current table to newTable.
     */
    void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];  //头插法，这样，转移完之后，链表的顺序变反了
                newTable[i] = e;
                e = next;
            }
        }
    }
```

多线程同时执行，扩容的时候都会去生成新的数组，并且都会去遍历老的数组，那么线程一的变量（key1，key2，key3）已经指向（transfer）到新的数组了，并且链表的数据为key3、key2、key1，即（e2表示线程2）

![image-20200526172440627](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200526172440627.png)



![image-20200526172839612](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200526172839612.png)

![image-20200526173151065](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200526173151065.png)



由于是头插法，所以会导致数据transfer的时候链表的顺序发生变化，所以在多线程的情况下，就有可能出现循环链表

这样就会出现循环链表了，死循环了



问？为什么在扩容的时候直接把数组对象指向到新的数组中？

答：扩容的目的其实是减短链表的长度，所以会在扩容的时候，遍历整个数组以及链表



#### 影响hash值的计算



```java
    /**
    
    
    初始化哈希种子
    如果觉得hashmap的算法不适合你，你可以通过设置 jdk.map.althashing.threshold 来影响生成hash值的散列情况，当数组的长度大于等于这个值的时候，就会起作用
     jdk.map.althashing.threshold：虚拟机的环境变量
    
    
     * Initialize the hashing mask value. We defer initialization until we
     * really need it.
     */
    final boolean initHashSeedAsNeeded(int capacity) {
        boolean currentAltHashing = hashSeed != 0;
      //数组容量如果大于等于** 的时候，switching为true,这个时候就会改变hash种子的值。这个值会在计算hash的时候用到的，就会导致计算出的hash值更散列一些
        boolean useAltHashing = sun.misc.VM.isBooted() &&
                (capacity >= Holder.ALTERNATIVE_HASHING_THRESHOLD);
      //^:异或 只有在不同的时候返回1
        boolean switching = currentAltHashing ^ useAltHashing;
        if (switching) {
            hashSeed = useAltHashing
                ? sun.misc.Hashing.randomHashSeed(this)
                : 0;
        }
        return switching;
    }
```



```java
private static class Holder {

        /**
         * Table capacity above which to switch to use alternative hashing.
         */
        static final int ALTERNATIVE_HASHING_THRESHOLD;

        static {
            String altThreshold = java.security.AccessController.doPrivileged(
                new sun.security.action.GetPropertyAction(
                    "jdk.map.althashing.threshold"));

            int threshold;
            try {
                threshold = (null != altThreshold)
                        ? Integer.parseInt(altThreshold)
                        : ALTERNATIVE_HASHING_THRESHOLD_DEFAULT;

                // disable alternative hashing if -1
                if (threshold == -1) {
                    threshold = Integer.MAX_VALUE;
                }

                if (threshold < 0) {
                    throw new IllegalArgumentException("value must be positive integer.");
                }
            } catch(IllegalArgumentException failed) {
                throw new Error("Illegal value for 'jdk.map.althashing.threshold'", failed);
            }

            ALTERNATIVE_HASHING_THRESHOLD = threshold;
        }
    }

```

