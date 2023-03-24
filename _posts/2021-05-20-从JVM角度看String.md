---
title: "从JVM角度看Java String"
toc: true
toc_label: "目录"
toc_icon: "cog"
toc_sticky: true
categories:
  - JVM
tags:
  - JVM
---

<!--more-->

# String基本特性
## 不可变性
```java
public class StringExer{
    String str=new String("good");
    char[] ch={'t','e','s','t'};

    public void change(String str,char[]){
        str="test ok";
        ch[0]='b';
    }

    public static void main(String[] args){
        StringExer ex=new StringExer();
        ex.change(ex.str,ex.ch);
        System.out.println(ex.str);//good
        System.out.println(ex.ch);//best
    }
}
```
## 字符串常量池

- 字符串常量池不会存储相同的字符串

通过debug功能查看String常量池中的数量可以发现这一点

```java
    System.out.println("1");//2330
    System.out.println("2");//2332
    System.out.println("3");//2333

    System.out.println("1");//2334
    System.out.println("2");//2334
    System.out.println("3");//2334
```

- String的String Pool是一个固定大小的Hashtable，如果放进pool的string非常多，就会造成hash冲突严重，从而导致链表很长，造成调用intern时性能大幅下降。
- `-XX:StringTableSize`设置StringTable的长度
- `intern`如果字符串常量池中没有对应的data字符串，则在常量池中生成

# String的内存分配

- 在Java语言中有8种基本数据类型和一种比较特殊的类型String。这些类型为了使它们在运行过程中速度更快、更节省内存，都提供了一种常量池的概念。
- String类型的常量池主要有两种使用方式
    - 直接使用双引号的会直接存储在常量池中。`String info="abc"`
    - 如果不是使用双引号声明的，可以使用String提供的`intern`方法

```java
class Memory{
    public static void main(String[] args){
        int i=1;
        Object obj=new Object();
        Memory mem=new Memory();
        mem.foo(obj);
    }
    private void foo(Object param){
        String str=param.toString();
        System.out.println(str);
    }
}
```

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/JVMAndString.jpg)

# 字符串拼接操作

1. 常量与常量的拼接结果在常量池，原理是编译器优化
2. 只要其中有一个是变量，结果就在堆中（不在常量池）。变量拼接的原理是StringBuilder
3. 如果拼接的结果调用intern()方法，则主动将常量池中还没有的字符串对象放入池中，并返回此对象地址。

```java
    String s1 = "javaEE";
    String s2 = "hadoop";
    String s3 = "javaEEhadoop";
    String s4 = "javaEE" + "hadoop";//编译期优化，等同于"javaEEhadoop"（可以看idea反编译结果）
    //如果拼接符号前后出现了一个变量，则相当于在堆空间中new String() 	
    String s5 = s1 + "hadoop";
    String s6 = "javaEE" + s2;
    String s7 = s1 + s2;

    System.out.println(s3 == s4);//true
    System.out.println(s3 == s5);//false
    System.out.println(s3 == s6);//false
    System.out.println(s3 == s7);//false
    System.out.println(s5 == s6);//false
    System.out.println(s5 == s7);//false
    System.out.println(s6 == s7);//false
    //intern()方法是如果常量池中没有就创建并返回地址，如果存在，则直接返回地址
    String s8 = s6.intern();
    System.out.println(s3 == s8);//true

    final String s9 = "javaEE";//被final修饰的视为常量
    final String s11 = "hadoop";
    String s10 = s9 + s11;
    System.out.println(s10 == s3);//true
```

如果用String做字符串拼接，通过查看字节码可以发现，每次都会创建新的StringBuilder和String对象，而且还涉及到GC，所以会非常耗时。（在实际开发中建议不要使用String拼接）

```java
    long startTime = System.currentTimeMillis();
    String s = new String();
    for (int i = 0; i < 100000; i++) {
        s = s + "a";
    }
    long endTime = System.currentTimeMillis();
    System.out.println("String拼接时间;" + (endTime - startTime));//5107
```

可以发现采用StringBuilder的方式，速度远快于String，因为自始至终只有StringBuilder这一个对象。

```java
    long startTime1 = System.currentTimeMillis();
    StringBuilder sb = new StringBuilder();
    for (int i = 0; i < 100000; i++) {
        sb.append("a");
    }
    long endTime1 = System.currentTimeMillis();
    System.out.println("StringBuilder拼接时间;" + (endTime1 - startTime1));//4
```

查看StringBuilder源码，StringBuilder的默认构造器指定默认的char数组大小为16，当大小不足的时候会进行扩容，我们可以手动指定大小做进一步优化。

```java
	/**
     * Constructs a string builder with no characters in it and an
     * initial capacity of 16 characters.
     */
    public StringBuilder() {
        super(16);
    }

    /**
     * Creates an AbstractStringBuilder of the specified capacity.
     */
    AbstractStringBuilder(int capacity) {
        value = new char[capacity];
    }

    /**
     * Returns a capacity at least as large as the given minimum capacity.
     * Returns the current capacity increased by the same amount + 2 if
     * that suffices.
     * Will not return a capacity greater than {@code MAX_ARRAY_SIZE}
     * unless the given minimum capacity is greater than that.
     *
     * @param  minCapacity the desired minimum capacity
     * @throws OutOfMemoryError if minCapacity is less than zero or
     *         greater than Integer.MAX_VALUE
     */
    private int newCapacity(int minCapacity) {
        // overflow-conscious code
        int newCapacity = (value.length << 1) + 2;
        if (newCapacity - minCapacity < 0) {
            newCapacity = minCapacity;
        }
        return (newCapacity <= 0 || MAX_ARRAY_SIZE - newCapacity < 0)
            ? hugeCapacity(minCapacity)
            : newCapacity;
    }
```

```java
    long startTime2 = System.currentTimeMillis();
   //指定一个初始大小
    StringBuilder sb1 = new StringBuilder(100000);
    for (int i = 0; i < 100000; i++) {
        sb1.append("a");
    }
    long endTime2 = System.currentTimeMillis();
    System.out.println("优化后的StringBuilder拼接时间;" + (endTime2 - startTime2));//4。运行时间没有明显优化是因为数据量不够大，效果不明显
```

# intern()

确保字符串在内存里只有一份拷贝，节约内存空间，加快字符串操作任务的执行速度。注意，这个值会被存放在字符串内部池。如果字符串中已经有了，返回对象的地址。如果没有，则会把对象的引用地址复制一份，放入常量池，并返回常量池中的引用地址。当遇到大量重复字符串的时候，使用intern可以节约空间，加快执行速度。

```java
    String s = new String("1");//在常量池中创建“1”对象，并且s指向“1”
    s.intern();//并没有效果
    String s2 = "1";//常量池“1”的地址
    System.out.println(s == s2);//false

    String s3 = new String("1") + new String("1");	//相当于在堆空间创造了“11”
    s3.intern();//在字符串常量池中生成“11”
    String s4 = "11";
    System.out.println(s3 == s4);//true
```

![alt]({{ site.url }}{{ site.baseurl }}/assets/images/JVMAndString2.jpg)

拓展：

```java
    String s3 = new String("1") + new String("1");    //相当于在堆空间创造了“11”
    String s4 = "11"; //在常量池中创造“11”
    s3.intern();//因为常量池中已经有“11”了，所以直接返回“11”的地址
    String s5 = s3.intern();//intern返回地址给s5
    System.out.println(s3 == s4);//false
    System.out.println(s5 == s4);//true
```

```java
    String s = new String("a") + new String("b");
    String s2 = s.intern();
    System.out.println(s2 == "ab");//true
    System.out.println(s == "ab");//true
```

# 面试题

## `new String("ab")`会创建几个对象？

看字节码，创建了两个对象。一个对象是new关键词在堆空间创建的，另外一个对象是字符串常量池中的对象“ab”。

```xml
 0 new #2 <java/lang/String>	//new 一个对象，放到堆里面
 3 dup
 4 ldc #3 <ab>				//常量池中放入ab
 6 invokespecial #4 <java/lang/String.<init>>
 9 astore_1
10 return
```

## `new String("a")+ new String("b")`会创建几个对象？

6个对象，详见字节码。

```java
 0 new #2 <java/lang/StringBuilder>	//1.字符串拼接，创建StringBuilder对象
 3 dup
 4 invokespecial #3 <java/lang/StringBuilder.<init>>
 7 new #4 <java/lang/String>	//2. 创建String对象
10 dup
11 ldc #5 <a>	//3.常量池中添加“a”对象
13 invokespecial #6 <java/lang/String.<init>>
16 invokevirtual #7 <java/lang/StringBuilder.append>
19 new #4 <java/lang/String>	//4.创建String对象
22 dup
23 ldc #8 <b>			//5.常量池中添加“b"
25 invokespecial #6 <java/lang/String.<init>>
28 invokevirtual #7 <java/lang/StringBuilder.append>
31 invokevirtual #9 <java/lang/StringBuilder.toString>	//6. StringBuilder的toString底层是new一个String，但不放在常量池
34 astore_1
35 return
```