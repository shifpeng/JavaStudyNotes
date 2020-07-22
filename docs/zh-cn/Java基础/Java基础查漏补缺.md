### Java查漏补缺

Java的两种核心机制

- Java虚拟机
- 垃圾回收机制

![image-20200610173208034](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200610173208034.png)

---

**知识点1:**

![image-20200629193205696](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200629193205696.png)

---

**知识点2：**

容量小的类型自动转化为容量大的数据类型：byte,short,char之间不会相互转化，他们三只直接在运算时首先会转化为int类型

```java
    byte b1 = 1;
    byte b2 = 2;
    int a = b1 + b2;
```

---

**知识点3:**

如何在内存中区分类和堆

- 类是静态的概念，是放在代码区里面的

- 对象是new出来的，位于堆内存（动态分配内存的），类的每个成员变量在不同的对象中都有不同的值（除了静态变量），而方法只有一份，执行的时候才占内存

---

**知识点4:**

```
public class Test{
	public static void main(String args[])
		Test test=new Test();  //这个时候应该在栈内存中有个test的局部变量，同时，凡是new，都要在堆内存中做分配，所以堆内存中有个test对象
		
}
```

![image-20200630175848232](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200630175848232.png)

---

**知识点5:**super：在Java类中使用super来引用基类的成分

this是当前对象的引用，super是当前对象的父类对象的引用

---

**知识点6:**

- 子类的构造的过程中必须调用其基类的构造方法
- 子类可以在自己的构造方法中使用super(args) 调用基类的构造方法
  - 使用this(args)调用自己的类的其他的构造方法
  - 如果调用了super，必须写在子类构造方法的第一行

- 如果子类的构造方法中没有显示的调用基类构造方法，系统默认调用基类无参数的构造方法，
- 如果子类构造方法中既没有显式调用基类构造方法，而基类中又没有无参的构造方法，则编译会出错

```java
public class SuperTest {
    protected void print(String args) {
        System.out.println(args);
    }

    SuperTest() {
        print("SuperTest()");
    }
}

public class Test extends SuperTest {
    Test() {
        print("Test()");
    }

    public void f() {
        print("Test：f()");
    }

    public static void main(String[] args) {
        Test test = new Test();
        test.f();
    }
}
//执行结果
SuperTest()
Test()
Test：f()
```

---

**知识点7：**

Object类是所有的Java类的根基类，如果在类的声明中未使用extends关键字制定其基类，则默认基类为Object类

---

**知识点8:** 对象转型（casting）

- 一个基类的引用类型变量可以“指向”其子类的对象（即需要的是动物对象，你可以传递一只狗进来）

- 一个基类的引用不可以访问其子类对象增加的成员（属性和方法）（需要的动物，传入一只狗的对象进来，那么这里是把这只狗当作一只动物传进来的，狗新增加的成员就不能使用）

-  可以使用 引用 变量 instanceof 类名 来判断该引用型变量所指向的对象是否属于该类或者该类的子类（即一个对象是否为一个类的实例）
  **注意：编译器会检查 obj 是否能转换成右边的class类型，如果不能转换则直接报错，如果不能确定类型，则通过编译，具体看运行时定。**

  ![](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200709174026972.png)

- 子类对象可以当作基类对象来使用称作为向上转型，反之称为向下转型

---

**知识点9：** 动态绑定和多态

new的什么对象，调用的时候就会调用该对象的方法

> 多态是同一个行为具有多个不同表现形式或形态的能力。
>
> 多态就是同一个接口，使用不同的实例而执行不同操作。
>
> ## 多态存在的三个必要条件
>
> - 继承
> - 重写
> - 基类引用指向派生类对象（引用还是指向基类）

![image-20200709180649163](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200709180649163.png)

![image-20200709181013045](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200709181013045.png)

---



**知识点10:** Final关键字

- final的变量不能被改变
- final的方法不能够被重写
- final的类不能够被继承

---

**知识点11:**接口特性

- 接口中声明的属性**默认**为public static final的，也只能是public static final的；

- 接口中只能定义抽象方法，而且这些方法默认为public的，也只能是public的



**知识点12：**Java中的异常

![image-20200717095343234](/Users/Steven/GitRepositories/JavaStudyNotes/docs/assets/image-20200717095343234.png)

Throwable：根异常：可被抛出的

Error：系统的错误，虚拟机出错了，我们处理不了的错误

Exception:我们可以catch的错误

