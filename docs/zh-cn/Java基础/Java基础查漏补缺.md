### Java查漏补缺

Java的两种核心机制

- Java虚拟机
- 垃圾回收机制

![image-20200610173208034](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200610173208034.png)

**知识点1:**

![image-20200629193205696](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200629193205696.png)



**知识点2：**

容量小的类型自动转化为容量大的数据类型：byte,short,char之间不会相互转化，他们三只直接在运算时首先会转化为int类型

```java
    byte b1 = 1;
    byte b2 = 2;
    int a = b1 + b2;
```

**知识点3:**

如何在内存中区分类和堆

- 类是静态的概念，是放在代码区里面的

- 对象是new出来的，位于堆内存（动态分配内存的），类的每个成员变量在不同的对象中都有不同的值（除了静态变量），而方法只有一份，执行的时候才占内存

**知识点4:**

```
public class Test{
	public static void main(String args[])
		Test test=new Test();  //这个时候应该在栈内存中有个test的局部变量，同时，凡是new，都要在堆内存中做分配，所以堆内存中有个test对象
		
}
```

![image-20200630175848232](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200630175848232.png)

**知识点5:**super：在Java类中使用super来引用基类的成分

this是当前对象的引用，super是当前对象的父类对象的引用



**知识点6:**

- 子类的构造的过程中必须调用其基类的构造方法
- 子类可以在自己的构造方法中使用super(args) 调用基类的构造方法
  - 使用this(args)调用自己的类的其他的构造方法
  - 如果调用了super，必须写在子类构造方法的第一行

- 如果子类的构造方法中没有显示的调用基类构造方法，系统默认调用基类无参数的构造方法，
- 如果子类构造方法中既没有显式调用基类构造方法，而基类中又没有无参的构造方法，则编译会出错