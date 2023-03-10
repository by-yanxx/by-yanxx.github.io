---
title: 测试「过一遍」JVM
date: 2023-02-26 21:14:56
tags: 测试标签
categories: 测试分类
toc: true
sticky:
---

{% note blue 'fas fa-bullhorn' %}
文章开篇内容简介
{% endnote %}

{% note red 'fas fa-warning' %}
特别注意
{% endnote %}

{% note green 'fas fa-lightbulb' %}
内容补充
{% endnote %}

{% note orange 'fas fa-flag' %}
其他标记
{% endnote %}

{% note info simple %} 
info 提示块标签
{% endnote %}

{% label 你好 blue %}
`niha`
*hiaia*
**bnfidf**


# 一 类加载

## （一）类文件结构

>[!总览]

```java
ClassFile {
    u4             magic; //Class 文件的标志
    u2             minor_version;//Class 的小版本号
    u2             major_version;//Class 的大版本号
    u2             constant_pool_count;//常量池的数量
    cp_info        constant_pool[constant_pool_count-1];//常量池
    u2             access_flags;//Class 的访问标记
    u2             this_class;//当前类
    u2             super_class;//父类
    u2             interfaces_count;//接口
    u2             interfaces[interfaces_count];//一个类可以实现多个接口
    u2             fields_count;//Class 文件的字段属性
    field_info     fields[fields_count];//一个类可以有多个字段
    u2             methods_count;//Class 文件的方法数量
    method_info    methods[methods_count];//一个类可以有个多个方法
    u2             attributes_count;//此类的属性表中的属性数
    attribute_info attributes[attributes_count];//属性表集合
}
```

### 1.1 魔数

```java
u4             magic; //Class 文件的标志
```

Class 文件的头 4 个字节为魔数（Majic Number）。

作用：用来判断该文件是否可以被 JVM 接收。

*一般为 0xCAFEBABE (咖啡宝贝🤔)，如果不是这个那就无法被 JVM 接收。*

### 1.2 Class 文件版本号

```java
u2             minor_version;//Class 的小版本号
u2             major_version;//Class 的大版本号
```

接着魔数的 4 个字节是版本号，第 5、6 位是次版本号，第 7、8 位是主版本号。

比如：Java8 -> Java9 主版本号+1

### 1.3 常量池

```java
u2             constant_pool_count;//常量池的数量
cp_info        constant_pool[constant_pool_count-1];//常量池
```

常量池计数器从 1 开始，为 0 时表示：不引用任何一个常量池项。

**常量池内容**
- `字面量`：指字母、数字等构成的字符串和数值常量，如：
	String str = "abc"; // --> 对应常量池里记录的字面量为 str 和 "abc"
	static String str2 = "def"; // --> 对应常量池里记录的字面量为 str2 和 "def"
	int a = 1; // --> 对应常量池里记录的字面量为 a
	final int b = 2; // --> 对应常量池里记录的字面量为 b 和 2
	static final c = 3; // -> 对应常量池里记录的字面量为 c 和 3
	或由 final 修饰的常量值
- `符号引用`包括：
	1. 类、接口的全限定名
	2. 字段名和描述符
	3. 方法名和描述符
	如：Ljava/lang/Integer;

对于符号引用，在常量池里面，它们都是静态的信息，只有到运行时被加载到内存后，这些符号才转为内存地址信息。
这些常量池被加载到内存时，就成为了“运行时常量池”。

常量池里的每一个常量都是一张表，共有 14 个，它们的第一位是一个 u1 类型的标志位 -tag，用来表示该常量的类型。

![](../pictures/Pasted%20image%2020230226221329.png)

*javap -v Student：查看 Student 类的常量池信息。*

### 1.4 访问标志

```java
u2             access_flags;//Class 的访问标记
```

作用：用于标志这个 Class 是类还是接口？是 public 的还是 abstract 的？是否被 final 修饰？

### 1.5 当前类、父类、接口

```java
u2             this_class;//当前类
u2             super_class;//父类
u2             interfaces_count;//接口
u2             interfaces[interfaces_count];//一个类可以实现多个接口
```

作用：
- `类索引`：用于确定类的全限定名
- `父类索引`：用于确定父类的全限定名，因为单继承，所以父类索引只有一个，且除了 Object 类外，其他类的父类索引都不为 0
- `接口索引`：是一个集合，用来描述这个类实现了哪些接口，按 implements 顺序存入

### 1.6 字段表集合

```java
u2             fields_count;//Class 文件的字段属性
field_info     fields[fields_count];//一个类可以有多个字段
```

字段表包括接口或类中声明的变量，包括类变量和实例变量，不包括局部变量。

字段表的 access_flag 一般是用来修饰字段的关键字。

*字段表里的内容都是 boolean 的，这样很适合作为标志位使用。至于字段名、字段数据类型，只能引用常量池中的常量来描述。*

### 1.7 方法表集合

```java
u2             methods_count;//Class 文件的方法数量
method_info    methods[methods_count];//一个类可以有个多个方法
```

methods_count 表示方法的数量，而 method_info 表示方法表。

方法表的 access_flag 和字段表非常相似，一般是用来修饰方法的关键字。

### 1.8 属性表

```java
u2             attributes_count;//此类的属性表中的属性数
attribute_info attributes[attributes_count];//属性表集合
```

*属性表无需了解。*

## （二）类生命周期

>[!总览]

![](../pictures/Pasted%20image%2020230226221306.png)

### 2.1 加载

类的加载：
- 就是将 .class 文件读入内存，并为之创建一个 java.lang.Class 对象
- 任何类被使用时，系统都会为之建立一个 java.lang.Class 对象

类的加载过程：加载 -> 连接(验证 -> 准备 -> 解析) -> 初始化。

类的加载过程，虚拟机主要完成三件事：
- 通过类的全限定名获取其定义的二进制字节流
- 将这个字节流所代表的静态存储结构转化为方法区的运行时数据结构
- 在 Java 堆中生成一个代表这个类的 java.lang.Class 对象，作为方法区这些数据的访问入口

加载 .class 文件的方式：
- 从本地系统中加载 .class 文件
- 通过网络下载 .class 文件
- 从 zip、jar 等归档文件中加载 .class 文件
- 从专有数据库中提取 .class 文件
- 将 Java 源文件动态编译为 .class 文件

### 2.2 连接

#### 2.2.1 验证

作用：确保被加载的类的正确性，验证 .class 文件的字节流中包含的信息符合当前虚拟机的要求，且安全。

验证过程：
- `文件格式验证`：验证字节流文件是否符合 .class 文件格式规范，比如检查文件头、版本号、常量池
- `元数据验证`：对字节码进行语义分析，确保符合 Java 语言规范
- `字节码验证`：通过数据流和控制流分析，确定语义合法、符合逻辑
- `符号引用验证`：确保解析动作能正确地执行

*验证阶段重要，但不是必须的。验证阶段对程序的运行没有影响，如果引用的类经过反复的验证，那么可以通过 -Xverifynone 参数来关闭大部分的类验证措施，从而缩短类加载时间。*

*另外，加载阶段和连接阶段是交叉进行的，加载尚未结束，连接阶段可能就已经开始了。这样的好处是，如果 文件格式之类出现问题，可以及时发现，不必等到加载结束后。*

#### 2.2.2 准备

作用：为类的静态变量分配内存，并将其初始化为默认值。

几点注意：
- 此时内存分配的只有类变量，没有实例变量，实例变量会在对象实例化的时候一起分配在堆中
- 这里赋予的是默认值（0、0.0、null、false 等），而不是代码中显式赋予的初始值，给变量赋予我们自己定义的初始值是在初始化阶段进行的

其他注意：
- 局部变量没有默认值，必须手动定义初始值
- 同时被 static 和 final 修饰的变量，必须在声明时显式地赋值；只被 final 修饰的常量既可以在声明时显式地赋值，也可以在类的初始化时显式地赋值。总之，系统不会为其赋予默认值，必须显式赋值
- 对于引用数据类型，可以直接使用，系统都会给它赋予默认值 null
- 如果数组初始化时没有对数组中的各元素赋值，那么数组中的元素会被赋予对应数据类型的默认值
- 如果变量在属性表中存在 ConstantValue 属性（即，被 static 和 final 同时修饰），那么在准备阶段，其变量值 value 会被初始化为 ConstantValue 属性所指定的值。可以理解为 static final 常量在编译期就将结果放入了调用它的类的常量池中

#### 2.2.3 解析

作用：把虚拟机内的常量池中的符号引用替换为直接引用的过程

`直接引用`：直接指向目标的指针、相对偏移量或一个能够间接定位到目标的句柄

### 2.3 初始化

作用：为类变量赋予正确的初始值，JVM 负责对类进行初始化。

给类变量定义初始值的方式：
- 声明类变量时直接定义初始值
- 使用静态代码块为类变量定义初始值

JVM 初始化步骤：
- 若该类尚未被加载和连接，那么程序会先加载、连接该类
- 若该类的父类尚未被初始化，那么先初始化其父类
- 若该类中有初始化语句，那么系统依次执行这些语句

类初始化的时机：只有主动调用这个类时，这个类才会初始化。所谓主动调用就是如下几种：
- 创建类的实例，也就是 new
- 访问某个类或接口的静态变量，或者对该静态变量赋值
- 调用类的静态方法
- 使用反射方式来强制创建某个类或接口对应的 java.lang.Class 对象
- 初始化某个类的子类，则其父类也会被初始化
- JVM 启动时被标明为启动类的类，直接使用 java.exe 命令运行某个类

### 2.4 使用

类访问方法区内的数据结构的接口，对象是 Heap (堆)区的数据。

### 2.5 卸载

JVM 结束生命周期的几种情况：
- 执行了 System.exit() 方法
- 程序正常执行结束
- 程序遇到了异常或错误而终止
- 操作系统出现错误，导致 JVM 进程终止

## （三）类加载机制

>[!前言]
>《深入理解java虚拟机》提到 “通过一个类的全限定名(packageName.ClassName)来获取描述此类的二进制字节(class文件字节)这个动作的代码模块就叫做类加载器(ClassLoader)”。
>
>类加载器作用：负责将 .class 文件加载到内存（运行时数据区）中，并为之生成一个 java.lang.Class 对象。

### 3.1 类加载器的层次

![](../pictures/Pasted%20image%2020230226221247.png)

*这里父类加载器不是继承关系，而是组合实现的，它们的父子关系只是职责上的。*

从虚拟机的角度，只存在两种类加载器：
- `启动类加载器`：Hotspot 虚拟机中由 C++ 实现（其它很多都是由 Java 实现），它是虚拟机的一部分
- `其他所有类加载器`：全都由 Java 实现，独立于虚拟机之外，全部继承自抽象类 java.lang.ClassLoader，这些类加载器需要由启动类加载器加载到内存中之后，才能去加载其他的类

从程序员的角度，存在三种类加载器：
- `启动类加载器(BootStrapClassLoader)`：负责加载 JDK/jre/lib 目录下的，或被 —Xbootclasspath 参数指定的路径中的，且能被虚拟机识别的类库，比如：所有 java.* 开头的类均由启动类加载器加载。启动类加载器无法被 Java 程序直接引用
- `扩展类加载器(ExtClassLoader)`：由 sun.misc.Launcher$ExtClassLoader 实现，负责加载 JDK/jre/lib/ext 目录中的，或被 java.ext.dirs 系统变量指定的路径中的所有类库，比如 javax.* 开头的类。程序员可以直接使用扩展类加载器
- `应用程序类加载器(AppClassLoader)`：由 sun.misc.Launcher$AppClassLoader 实现，负责加载用户类路径(ClassPath)下的所有 jar 包和类，开发者可以直接使用该类加载器，如果应用程序中没有自定义过自己的类加载器，一般情况下这个就是程序中默认的类加载器
- `自定义类加载器`：如有必要，还可以自定义一个类加载器，使其更加强大，比如：
	- 在执行非置信代码之前，自动验证数字签名
	- 动态地创建符合用户特定需要的定制化构建类
	- 从数据库或网络等特定场所获取 java class

### 3.2 寻找类加载器

使用 `getParent()` 方法获取父类加载器。

比如：
```java
package com.pdai.jvm.classloader;
public class ClassLoaderTest { 
	public static void main(String[] args) { 
		ClassLoader loader = Thread.currentThread().getContextClassLoader();
		System.out.println(loader);
		System.out.println(loader.getParent());
		System.out.println(loader.getParent().getParent()); 
	} 
}
```

输出：
```java
sun.misc.Launcher$AppClassLoader@64fef26a
sun.misc.Launcher$ExtClassLoader@1ddd40f3
null
```

### 3.3 类的加载

三种加载方式：
- 命令行启动应用时，由 JVM 初始化加载（静态加载）
- 通过 Class.forName() 方法动态加载
- 通过 ClassLoader.loadClass() 方法动态加载

Class.forName() 和 ClassLoader.loadClass()区别？
- `Class.forName()`: 将类的 .class 文件加载到 JVM 中之外，还会对类进行解释，执行类中的 static 块，即，它会初始化 Class 对象
- `ClassLoader.loadClass()`: 只干一件事情，就是将 .class 文件加载到 JVM 中，不会执行 static 中的内容，只有在 newInstance 才会去执行 static 块
- `Class.forName(name, initialize, loader)`：带参函数也可控制是否加载 static 块，并且只有调
用了 newInstance() 方法采用调用构造函数，创建类的对象

### 3.4 类加载机制

四种类加载机制：
- `全盘委托`：当一个类加载某个 Class 时，这个类所依赖和调用的其他类也都由这个类加载器负责载入，除非显式地使用另一个类加载器
- `父类委托`：先让父类加载器尝试加载，父类加载器无法加载该类时，再尝试从自己的类路径中加载该类
- `缓存机制`：保证所有加载过的 Class 都能被缓存，需要加载某个 Class 时，会先从缓存区中寻找，缓存区里没有的话，系统会读取该类的二进制数据，并转为 Class 对象存入缓存区。这也是修改 Class 后，必须重启 JVM，修改才能生效
- `双亲委派机制`：当类加载器收到加载一个类的请求时，它首先不会自己去加载这个类，而是将加载任务委托给它的父类加载器，依次向上，直到最上方的启动类加载器。只有父类加载器无法完成加载时，子加载器才会尝试自己去加载该类

### 3.5 双亲委派机制

双亲委派机制的过程：
- 当 AppClassLoader 加载一个 class 时，它首先不会自己去尝试加载这个类，而是把类加载请求委派给父类加载器 ExtClassLoader 去完成
- 当 ExtClassLoader 加载一个 class 时，它首先也不会自己去尝试加载这个类，而是把类加载请求委派给 BootStrapClassLoader 去完成
- 如果 BootStrapClassLoader 加载失败(例如在 $JAVA_HOME/jre/lib 里未查找到该 class)，会使用 ExtClassLoader 来尝试加载
- 若 ExtClassLoader 也加载失败，则会使用 AppClassLoader 来加载，如果 AppClassLoader 也加载失败，则会报出异常 ClassNotFoundException

双亲委派的好处：
- 防止内存中出现多份同样的字节码
- 保证 Java 程序安全稳定运行

[其他资料](obsidian://open?vault=Obsidian%20Vault&file=3_%E3%80%8CJava%20%E3%80%8D(%E7%BD%91%E8%AF%BE%E5%90%91)%2F2-%E3%80%8C%E7%A8%8B%E5%AD%A6%E5%A7%90%E6%8E%A8%E8%8D%90%E8%B7%AF%E7%BA%BF%E3%80%8D%2F%E3%80%9001_%E5%9F%BA%E7%A1%80%E7%AF%87%E3%80%91%2F2-Java_JVM%2F01_JVM%20%E4%BD%93%E7%B3%BB%E7%BB%93%E6%9E%84)

### 3.6 自定义类加载器

在通过网络传输字节码时，为保证安全，一般会对字节码进行加密，在解密时系统的类加载器无法完成加载，那就需要自定义类加载器来实现加载任务。

除了启动类加载器，其他类加载器均由 Java 实现且全部继承自 java.lang.ClassLoader。

自定义类加载器一般继承自 ClassLoader 类
- 想要破坏双亲委派机制，可以重写 loadClass() 方法
- 如果不想破坏双亲委派机制，只需重写 findClass() 方法即可

几点注意：
- 传输的文件名需要类的全限定性名，因为 defineClass() 方法将字节码转为 Class 就是以这种格式进行处理的
- 最好不要重写 loadClass() 方法，容易破坏双亲委派机制
- .java 本身可以被 AppClassLoader 类加载，对应的 .class 文件不能放在类路径下，否则，由于双亲委派机制，可能导致该类由 AppClassLoader 加载，而不会通过自定义的类加载器加载


# 二 JVM 内存结构

>[!前言]
>JVM 内存布局规定了 Java 在运行过程中内存申请、分配、管理的策略。
>JVM 的内存结构，也就是运行时数据区的结构，包括：
>- 线程共享的：堆、方法区（包括运行时常量池）、堆外内存（Java7 的永久代或 Java8 的元空间、代码缓存）
>- 线程私有的：PC 寄存器（也叫程序计数器）、虚拟机栈、本地方法栈 

![](../pictures/Pasted%20image%2020230226221229.png)

## （一）程序计数器

### 1.1 作用

用来存储指向下一条指令的地址（也叫偏移地址），即，将要执行的指令代码。

由执行引擎读取下一条指令。

### 1.2 概述

相关要点：
- 它所占的内存空间很小，运行速度最快
- 线程私有，每个线程都有一个程序计数器，生命周期和所在的线程一致
- 当前线程执行的如果是 Java 方法，那么程序计数器记录的是 JVM 字节码指令地址；如果是 native 方法，那记录的就是未指定值
- 它本质上是程序控制流的指示器，分支、循环、跳转、异常处理、线程恢复等都依赖它
- 字节码解释器工作时就是通过改变这个计数器的值来选取下一条需要执行的字节码指令
- 唯一一个没有 `OutOfMemoryError` 情况的区域

## （二）Java 虚拟机栈

>[!前言]
>Java 虚拟机栈，以前也叫 Java 栈。

### 2.1 概述

每个线程被创建的时候，也会创建一个属于该线程虚拟机栈，内部保存一个个栈帧，每个栈帧都对应着一次方法的调用，线程私有，生命周期与线程一致。

作用：主管 Java 程序的运行，保存局部变量、部分结果，参与方法的调用与返回。

特点：
- 对栈的访问速度很快，仅次于程序计数器
- JVM 直接对虚拟机栈的操作有两个，一个是方法执行时，入栈；另一个是方法结束时，出栈
- 不存在垃圾回收

JVM 允许虚拟机栈的大小是固定或者动态的，这样可能出现的问题：
- 如果虚拟机栈大小固定，那么超过虚拟机栈长度时，会报栈溢出异常
- 如果虚拟机大小不固定，是动态的，那么当动态扩展时如果无法申请到足够的内存，或者没有足够的内存满足申请的需求，那么会报内存溢出异常

*可以通过 —Xss 来设置线程的最大栈空间，它直接决定了函数调用的最大可达深度。*

### 2.2 栈的存储单位

栈里存储着什么：
- 一个线程一个栈，栈里存的是一条一条的栈帧
- 一个线程里的每一个方法，都对应着一条栈帧
- 栈帧就是一个内存区块，是一个数据集，维系方法执行过程中的各种数据信息

### 2.3 栈运行原理

- JVM 直接对虚拟机栈的操作就两个，一个入栈，一个出栈
- 一条正在运行的线程，某一时刻，只会有一个活动的栈帧，也就是处于栈顶的栈帧，该栈帧称为“当前栈帧”，“当前栈帧”对应的方法称为“当前方法”，定义这个方法的类称为“当前类”
- 执行引擎运行的所有字节码指令只针对“当前帧”进行操作
- 方法里调用了其他方法，那么会在栈顶继续创建一个新的栈帧，此时这个处于栈顶的新的栈帧就是新的“当前栈帧”
- 不同线程中的栈帧不允许互相引用，即，不能在一个栈帧中引用其他线程的栈帧
- 方法里调用了其他方法，当其他方法执行结束，在返回时，会将返回值传递给前一个栈帧，接着丢弃“当前栈帧”，其前一个栈帧成为新的“当前栈帧”
- Java 有两种返回方式，一种是 return，另一种是抛出异常，这两种都会使栈帧弹出

### 2.4 栈帧的内部结构

栈帧里存储着什么：
- 局部变量表
- 操作数栈（也叫表达式栈）
- 动态链接：指向运行时常量池里的方法引用
- 方法返回地址：方法 return 正常退出和抛出异常退出的地址
- 一些附加信息

![](../pictures/Pasted%20image%2020230226221159.png)
![700](Pasted%20image%2020230225192051.png)

#### 2.4.1 局部变量表

局部变量表：
- 局部变量表，也叫局部变量数组、本地变量表
- 主要用于存储形参、局部变量，包括基本数据类型、对象引用、return 地址的类型（指向了一条字节码指令的地址，已经被异常表取代
- 局部变量表存在于栈帧，栈帧存在于线程，且被线程私有，所以不存在安全问题
- 局部变量表的大小是在编译期间确定的，运行期间表的大小不会变
- 方法嵌套调用的次数由栈的大小决定，栈越大，可调用的次数就越多。一个方法的参数与局部变量越多，其局部变量表就越大，栈帧也随之变大，导致其占用更多的栈空间，进而使得方法的嵌套调用次数减少
- 局部变量表里的变量只在当前方法调用中有效，方法执行时，JVM 通过局部变量表，将参数值传递给形参；方法结束时，对应栈帧出栈，局部变量表随之销毁
- 参数值的存放总是在局部变量表的 index 从 0 开始，到 长度 -1 的索引结束

槽（Slot）
- 局部变量表的基本存储单元是 Slot（槽，也叫变量槽）
- 在局部变量表里，32 位（4 个字节）以内的类型只占用一个 Slot（包括 return Address 类型），64 位（8 个字节）的类型占用连续的两个 Slot
	- byte、short、char 存储之前会转为 int，boolean 也会转为 int（0 表示 false，1 表示 true）
	- long、double 占用连续的两个 Slot
	- 对于占用两个 Slot 的类型来说，访问索引时不能拆开访问，只需访问前一个索引即可
- 若当前帧是由构造方法或实例方法创建的，那么该对象引用的 this 变量会存放于 index 为 0 的 Slot 处，其余参数按参数列表的顺序排列
- 若当前帧是由静态方法创建的，但因为静态方法的局部变量表里并没有 this 变量，所以在静态方法里无法调用 this
- 栈帧的局部变量表里的 Slot 是可以重用的，如果一个局部变量过了其作用域，那么在此之后的新的局部变量很可能会复用已过期的局部变量的 Slot，从而节省资源空间
- 在栈帧中，于性能调优关系最大的就是局部变量表
- 局部变量表中的变量也是重要的垃圾回收根节点，只要被局部变量表中元素直接或间接引用的对象都不会被回收

#### 2.4.2 操作数栈

*【暂时未深入学习，回头再看。】*

#### 2.4.3 动态链接

*【暂时未深入学习，回头再看。】*

#### 2.4.4 方法返回地址

*【暂时未深入学习，回头再看。】*

#### 2.4.5 附加信息

*【暂时未深入学习，回头再看。】*






## （三）本地方法栈

## （四）堆

## （五）方法区

## （六）直接内存

## （七）元数据区内存


# 三 垃圾回收

# 四 Java 内存模型

# 五 调优与排错
