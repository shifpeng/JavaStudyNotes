### 简介
在JDK1.8之前，HashMap采用数组+链表实现，即使用链表处理冲突，同一hash值的节点都存储在一个链表里。但是当位于一个桶中的元素较多，即hash值相等的元素较多时，通过key值依次查找的效率较低。而JDK1.8中，为了解决hash碰撞过于频繁的问题，HashMap采用数组+链表+红黑树实现，当链表长度超过阈值（8）时，将链表(查询时间复杂度为O(n))转换为红黑树(时间复杂度为O(lg n))，极大的提高了查询效率。以下没有特别说明的均为JDK1.8中的HashMap。

### 特点

HashMap 可以说是我们使用最多的 Map 集合，它有以下特点：

- 键不可重复，值可以重复
- 底层哈希表
- 线程不安全
- 允许key为null，value也可以为null

### 数据结构
在Java中，保存数据有两种比较简单的数据结构：数组和链表。数组的特点是：寻址容易，插入和删除困难；链表的特点是：寻址困难，但插入和删除容易；所以我们将数组和链表结合在一起，发挥两者各自的优势，使用一种叫做拉链法的方式可以解决哈希冲突。

#### JDK1.8之前

JDK1.8之前采用的是拉链法。拉链法：将链表和数组相结合。也就是说创建一个链表数组，数组中每一格就是一个链表。若遇到哈希冲突，则将冲突的值加到链表中即可。

![拉链法](../../assets/拉链法.png)

#### JDK1.8之后

相比于之前的版本，jdk1.8在解决哈希冲突时有了较大的变化，当链表长度大于阈值（默认为8）时，将链表转化为红黑树，以减少搜索时间。

![数组+红黑树](../../assets/数组+红黑树.png)

#### JDK1.7 VS JDK1.8 比较

JDK1.8主要解决或优化了一下问题：

- resize 扩容优化
- 引入了红黑树，目的是避免单条链表过长而影响查询效率，红黑树算法请参考
- 解决了多线程死循环问题，但仍是非线程安全的，多线程时可能会造成数据丢失问题。

|不同|JDK 1.7|JDK 1.8|
---|---|---|
|存储结构|	数组 + 链表	数组 + 链表 + 红黑树
|初始化方式	|单独函数：inflateTable()	|直接集成到了扩容函数resize()中
|hash值计算方式|	扰动处理 = 9次扰动 = 4次位运算 + 5次异或运算|	扰动处理 = 2次扰动 = 1次位运算 + 1次异或运算
|存放数据的规则|	无冲突时，存放数组；冲突时，存放链表|	无冲突时，存放数组；冲突 & 链表长度 < 8：存放单链表；冲突 & 链表长度 > 8：树化并存放红黑树
|插入数据方式|	头插法（先讲原位置的数据移到后1位，再插入数据到该位置）|	尾插法（直接插入到链表尾部/红黑树）
|扩容后存储位置的计算方式|	全部按照原来方法进行计算（即hashCode ->> 扰动函数 ->> (h&length-1)）	|按照扩容后的规律计算（即扩容后的位置=原位置 or 原位置 + 旧容量）

扰动处理：高位和低位进行异或操作

整数值的hash值是他本身

//Computes key.hashCode() and spreads (XORs) higher bits of hash to lower.
```
static final int hash(Object key) {
        int h;
        return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    }
```

### 继承关系图
![hashmap继承关系图](../../assets/hashmap继承关系图.jpeg)

HashMap继承抽象类AbstractMap，实现Map接口。除此之外，它还实现了两个标识型接口，这两个接口都没有任何方法，仅作为标识表示实现类具备某项功能。Cloneable表示实现类支持克隆，java.io.Serializable则表示支持序列化。

#### 成员变量

```
//默认初始化Node数组容量16   初始化数组容量大小 16
static final int DEFAULT_INITIAL_CAPACITY = 1 << 4;
//最大的数组容量
static final int MAXIMUM_CAPACITY = 1 << 30;
//默认负载因子0.75   
//**例如给一个桶加水的时候，肯定不是i个加满之后再加桶（扩容需要时间），而是在第一个桶快满的时候就增加（扩容），而这个值是对空间和时间效率的一个平衡选择，一般不做修改
static final float DEFAULT_LOAD_FACTOR = 0.75f;
//由链表转红黑树的临界值 树化
static final int TREEIFY_THRESHOLD = 8;
//由红黑树转链表的临界值
static final int UNTREEIFY_THRESHOLD = 6;
//桶转化为树形结构的最小容量  
//**单个桶的节点超过8 ，整体的节点个数超过64，才会转化为树形结构
static final int MIN_TREEIFY_CAPACITY = 64;
//HashMap结构修改的次数，结构修改是指更改HashMap中的映射数或以其他方式修改其内部结构(例如，rehash的修改)。该字段用于在Collection-views上快速生成迭代器。
transient int modCount;  
//Node数组下一次扩容的临界值，第一次为16*0.75=12（容量*负载因子）
int threshold;
//负载因子
final float loadFactor;
//map中包含的键值对的数量
transient int size;
//表数据，即Node键值对数组，Node是单向链表，它实现了Map.Entry接口，总是2的幂次倍
//Node<K,V>是HashMap的内部类，实现Map.Entry<K,V>接口，HashMap的哈希桶数组中存放的键值对对象就是Node<K,V>。类中维护了一个next指针指向链表中的下一个元素。值得注意的是，当链表中的元素数量超过TREEIFY_THRESHOLD后会HashMap会将链表转换为红黑树，此时该下标的元素将成为TreeNode<K,V>,继承于LinkedHashMap.Entry<K,V>，而LinkedHashMap.Entry<K,V>是Node<K,V>的子类，因此HashMap的底层数组数据类型即为Node<K,V>。
transient Node<K,V>[] table;
//存放具体元素的集,可用于遍历map集合
transient Set<Map.Entry<K,V>> entrySet;

```

##### capacity、threshold和loadFactor之间的关系：

- capacity table的容量，默认容量是16
-threshold table扩容的临界值
- loadFactor 负载因子，一般 threshold = capacity * loadFactor，默认的负载因子0.75是对空间和时间效率的一个平衡选择，建议大家不要修改。

```
初始容量为2的n次幂
为什么是2的n次幂
1、进行&运算 
2、扩容之后元素的移动

```

#### 构造方法
```
//初始化容量以及负载因子
public HashMap(int initialCapacity, float loadFactor) {
    //判断初始化数组的容量大小
    if (initialCapacity < 0)
        throw new IllegalArgumentException("Illegal initial capacity: " +
                                           initialCapacity);
    if (initialCapacity > MAXIMUM_CAPACITY)
        initialCapacity = MAXIMUM_CAPACITY;
    //判断初始化的负载因子大小和是否为浮点型
    if (loadFactor <= 0 || Float.isNaN(loadFactor))
        throw new IllegalArgumentException("Illegal load factor: " +
                                           loadFactor);
    //初始化负载因子
    this.loadFactor = loadFactor;
    this.threshold = tableSizeFor(initialCapacity);
}  

//初始化容量
public HashMap(int initialCapacity) {  
    this(initialCapacity, DEFAULT_LOAD_FACTOR);  
}  

//默认构造方法
public HashMap() {  
    this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted  
}  
 
//把另一个Map的值映射到当前新的Map中
public HashMap(Map<? extends K, ? extends V> m) {  
    this.loadFactor = DEFAULT_LOAD_FACTOR;  
    putMapEntries(m, false);  
}  

```
其中主要有两种方式
- 定义初始容量大小（table数组的大小，缺省值为16），定义负载因子（缺省值为0.75）的形式

- 直接拷贝别的HashMap的形式，在此不作讨论

> 值得注意的是，当我们自定义HashMap初始容量大小时，构造函数并非直接把我们定义的数值当做HashMap容量大小，而是把该数值当做参数调用方法tableSizeFor，然后把返回值作为HashMap的初始容量大小

##### tableSizeFor()方法说明

```
//HashMap 中 table 角标计算及table.length 始终为2的幂，即 2 ^ n
//返回大于initialCapacity的最小的二次幂数值
static final int tableSizeFor(int cap) {
    int n = cap - 1;
    n |= n >>> 1;
    n |= n >>> 2;
    n |= n >>> 4;
    n |= n >>> 8;
    n |= n >>> 16;
    return (n < 0) ? 1 : (n >= MAXIMUM_CAPACITY) ? MAXIMUM_CAPACITY : n + 1;
}

```

### 静态内部类

#### Node

HashMap将hash，key，value，next已经封装到一个静态内部类Node上。它实现了Map.Entry<K,V>接口。

```
static class Node<K,V> implements Map.Entry<K,V> {
    // 哈希值，HashMap根据该值确定记录的位置
    final int hash;
    // node的key
    final K key;
    // node的value
    V value;
    // 链表下一个节点
    Node<K,V> next;

    // 构造方法
    Node(int hash, K key, V value, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.value = value;
        this.next = next;
    }

    // 返回 node 对应的键
    public final K getKey()        { return key; }
    // 返回 node 对应的值
    public final V getValue()      { return value; }
    public final String toString() { return key + "=" + value; }

    public final int hashCode() {
        return Objects.hashCode(key) ^ Objects.hashCode(value);
    }

    public final V setValue(V newValue) {
        V oldValue = value;
        value = newValue;
        return oldValue;
    }

    //作用：判断2个Entry是否相等，必须key和value都相等，才返回true
    public final boolean equals(Object o) {
        if (o == this)
            return true;
        if (o instanceof Map.Entry) {
            Map.Entry<?,?> e = (Map.Entry<?,?>)o;
            if (Objects.equals(key, e.getKey()) &&
                Objects.equals(value, e.getValue()))
                return true;
        }
        return false;
    }
}

```   
