## JDK1.7 HashMap

由数组+链表实现

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
但是无法保证取余之后的所有的key都是不一样的，所以key重复时，就用链表存储

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
        int hash = hash(key);
        int i = indexFor(hash, table.length);
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

    /**
     * Adds a new entry with the specified key, value and hash code to
     * the specified bucket.  It is the responsibility of this
     * method to resize the table if appropriate.
     *
     * Subclass overrides this to alter the behavior of put method.
     */
    void addEntry(int hash, K key, V value, int bucketIndex) {
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
```




