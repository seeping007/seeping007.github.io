---
layout: post
title: Java代码性能优化建议
description: 分享若干条Java代码性能优化小Tip
category: blog
---

### 前言

> 编程是一门严谨、科学的学问，为自己写的每一行代码负责是程序员的基本素养。在开发过程中，我们经常会因为项目上线的压力，或者自己思维上的懒惰，不自觉地敲下一些性能底下、结构邋遢，甚至有bug隐患的代码。久而久之，一个又一个看似细小的“不注意”积少成多，终究会对程序的性能和维护带来不好的影响。因此，平时开发过程中，应该有意识地养成良好的编码习惯，凡事多想一想，有没有更高效、更优雅的写法。
> 
> 也许有人会说，为什么老是要去抠这些所谓的效率，区别有那么大么，应该更关心业务才对（无视代码规范也属于此类）。我觉得，从不去思考如何写出高效、优雅代码的程序员，也不会真正理解业务的背后有多少资源和逻辑在支持，更不会倾其所能与他人一起探索更优质的解决方案，你相信这样能做好业务？反正我不信。"码如其人"，且码且珍惜。
> 
> 话不多说，下面就分享一些针对Java代码的性能优化建议。



### Java代码性能优化建议

#### 1. 尽量指定类、方法的final修饰符（TODO：内联、java运行期优化）

带final的方法不可被重写，带final的类不可被继承（final类所有的方法都是final的）。Java编译器会寻找机会内联所有的final方法，内联对于提升Java运行效率作用重大，具体参见Java运行期优化。

#### 2. 尽量重用对象
特别是String对象的重用，字符串连接应该使用StringBuilder / StringBuffer。首先，JVM生成对象、回收对象都是需要花时间的；其次，生成对象亦需要分配内存空间，过多的对象创建也会加快新生代的膨胀速度，导致高频的Minor GC。

#### 3. 尽量使用局部变量
调用方法传递的参数以及方法中的临时变量都保存在栈中，速度较快，其他变量（如静态变量、成员变量等）都在堆中创建，速度较慢。另外，局部变量随着方法的运行结束而销毁，不参与GC。

#### 4. 及时关闭流
数据库连接、I/O流使用完毕后务必及时关闭以释放资源，因为对这些大对象的操作会造成系统大的开销，稍有不慎，将会导致严重的后果。

#### 5. 尽量减少对变量的重复计算
明确一个概念，对方法的调用，即使方法中只有一条语句，也是有消耗的，包括创建栈帧、调用方法时保护现场、调用方法完毕时恢复现场等。所以例如以下操作：

```java
for(int i = 0; i < list.size(); i++) {...}
```
建议替换为：

```java
for(int i = 0, int length = list.size(); i < length; i++) {...}
```
这样，当size很大的时候，就减少了很多的消耗。

#### 6. 尽量采用懒加载的策略，即在需要的时候才创建
例如：

```java
String str = "aaa";
if(i == 1) {
    list.add(str);
}
```

建议替换为：

```java
if(i == 1) {
	String str = "aaa";
	list.add(str);
}
```

#### 7. 慎用异常（TODO：异常的正确打开方式）
异常的抛出对性能不利。抛出异常首先要创建一个新的对象，Throwable接口的构造函数调用名为fillInStackTrace()的本地同步方法，fillInStackTrace()方法检查堆栈，收集调用跟踪信息。只要有异常被抛出，JVM就必须调整调用堆栈，因为在处理过程中创建了一个新的对象。异常只能用于错误处理，不应该用来控制程序流程。

#### 8. 除非有特殊的理由，不要在循环中使用try...catch...，应该将其放在最外层
很多网络文章都强调了这一点，不过我个人觉得要视情况而定。当没有异常抛出时，try...catch...放在循环里面或者外面实际上并没有性能上的区别。当有异常抛出时，而且此时try...catch...放在循环里面，则异常有可能会被频繁抛出，这是会极大消耗性能的（如果恰好在一个死循环里面那就见鬼了）。不过，业务中也不排除需要把try...catch...放在循环里的情况，比如你希望不管发生什么情况这个循环都能一直不停地执行下去。所以，除非情况特殊，try...catch...放在循环外面是一个较为合理的选择。

#### 9. 如果能估计到待添加的内容长度，为底层以数组方式实现的集合、工具类指定初始长度
比如ArrayList、LinkedLlist、StringBuilder、StringBuffer、HashMap、HashSet等等.
以Stringbuilder为例：

```java
StringBuilder()　　　　　　// 默认分配16个字符的空间
StringBuilder(int size)　　// 默认分配size个字符的空间
StringBuilder(String str)　// 默认分配16个字符+str.length()个字符空间
```
为此类结构设定初始化容量可以明显地提升性能。比如StringBuilder（length表示当前能容纳的字符数量），当其达到最大容量的时候，其扩容的大小为length * 2 + 2。无论何时，只要StringBuilder进行扩容，它就不得不创建一个新的字符数组，然后将旧的字符数组内容拷贝到新字符数组中－－－这是十分耗费性能的一个操作。所以，为底层以数组方式实现的集合、工具类设置一个合理的初始长度，这样可以减少此类扩容的发生。

#### 10. 当复制大量数据时，使用System.arraycopy()（TODO：待验证）

```java
public static void arraycopy(Object src,   // 源数组
                             int srcPos,   // 源数组要复制的起始位置
                             Object dest,  // 目的数组
                             int destPos,  // 目的数组要复制的起始位置
                             int length)   // 复制的长度
```
注意：src和dest都必须是同类型或者可以进行类型转换的数组。

#### 11. 乘法和除法使用移位操作
例如：

```java
for (val = 0; val < 100000; val += 5) {
	a = val * 8;
	b = val / 2;
}
```
用移位操作可以极大地提高性能，因为在计算机底层，对位的操作是最方便、最快的，因此建议修改为：

```java
for (val = 0; val < 100000; val += 5) {
    a = val << 3;
	b = val >> 1;
}
```
移位操作虽然快，但是可能会降低代码可读性，因此最好加上相应的注释。

#### 12. 循环内不要不断创建对象引用
例如：

```java
for (int i = 1; i <= count; i++) {
	Object obj = new Object();
}
```
这种做法会导致内存中有count份Object对象引用存在，count很大的话，就耗费内存了，建议修改为：

```java
Object obj = null;
for (int i = 0; i <= count; i++) { 
	obj = new Object(); 
}

```

#### 13. 尽量使用HashMap、ArrayList、StringBuilder
除非线程安全需要，否则不推荐使用Hashtable、Vector、StringBuffer，后三者由于使用同步机制而导致了性能开销。

#### 14. 不要将数组声明为public static final
因为这毫无意义，这样只是定义了引用为static final，数组的内容还是可以随意改变的，将数组声明为public更是一个安全漏洞，这意味着这个数组可以被外部类所改变。

#### 15. 尽量在合适的场合使用单例
使用单例可以减轻加载的负担、缩短加载的时间、提高加载的效率，但并不是所有地方都适用于单例，简单来说，单例主要适用于以下三个方面：

- 控制资源的使用，通过线程同步来控制资源的并发访问
- 控制实例的产生，以达到节约资源的目的
- 控制数据的共享，在不建立直接关联的条件下，让多个不相关的进程或线程之间实现通信

#### 16. 尽量避免随意使用静态变量
要知道，当某个对象被定义为static的变量所引用，那么GC通常是不会回收这个对象所占有的堆内存的。

#### 17. 实现RandomAccess接口的集合比如ArrayList，应当使用最普通的for循环而不是foreach循环来遍历
这是JDK推荐给用户的。JDK API对于RandomAccess接口的解释是：实现RandomAccess接口用来表明其支持快速随机访问，此接口的主要目的是允许一般的算法更改其行为，从而将其应用到随机或连续访问列表时能提供良好的性能。实际经验表明，实现RandomAccess接口的类实例，假如是随机访问的，使用普通for循环效率将高于使用foreach循环；反过来，如果是顺序访问的，则使用Iterator会效率更高。可以使用类似如下的代码作判断：

```java
if (list instanceof RandomAccess) { 
	for (int i = 0; i < list.size(); i++){}
} else {
	Iterator<?> iterator = list.iterable(); 
	while (iterator.hasNext()) {
		iterator.next();
	}
}
```
foreach循环的底层实现原理就是迭代器Iterator，参见Java语法糖1：可变长度参数以及foreach循环原理。

#### 18. 使用同步代码块替代同步方法
除非能确定一整个方法都是需要进行同步的，否则尽量使用同步代码块，避免对那些不需要进行同步的代码也进行了同步，影响了代码执行效率。

#### 19. 不要创建一些不使用的对象，不要导入一些不使用的类
这毫无意义，如果代码中出现"The value of the local variable i is not used"、"The import java.util is never used"，那么请删除这些无用的内容。

#### 20. 程序运行过程中避免使用反射
反射是Java提供给用户一个很强大的功能，功能强大往往意味着效率不高。不建议在程序运行过程中使用尤其是频繁使用反射机制，特别是Method的invoke方法，如果确实有必要，一种建议性的做法是将那些需要通过反射加载的类在项目启动的时候通过反射实例化出一个对象并放入内存－－－用户只关心和对端交互的时候获取最快的响应速度，并不关心对端的项目启动花多久时间。

#### 21. 使用数据库连接池和线程池
这两个池都是用于重用对象的，前者可以避免频繁地打开和关闭连接，后者可以避免频繁地创建和销毁线程。

#### 22. 使用带缓冲的输入输出流进行IO操作
带缓冲的输入输出流，即BufferedReader、BufferedWriter、BufferedInputStream、BufferedOutputStream，这可以极大地提升IO效率。

#### 23. 顺序插入和随机访问比较多的场景使用ArrayList，元素删除和中间插入比较多的场景使用LinkedList
这个，理解ArrayList和LinkedList的原理就知道了。

#### 24. 字符串变量和字符串常量equals的时候将字符串常量写在前面
这么做主要是可以避免空指针异常。总之，务必确保调用equals的变量不为空。

#### 25. 把一个基本数据类型转为字符串，基本数据类型.toString()是最快的方式，String.valueOf(数据)次之，数据+""最慢
把一个基本数据类型转为String一般有三种方式，我有一个Integer型数据i，可以使用i.toString()、String.valueOf(i)、i+“”三种方式，三种方式的效率如何，看一个测试：

```java
public static void main(String[] args) { 
	int loopTime = 50000;
	Integer i = 0; 
	long startTime = System.currentTimeMillis(); 
	for (int j = 0; j < loopTime; j++) {
		String str = String.valueOf(i);
	}
	System.out.println("String.valueOf()：" 
                       + (System.currentTimeMillis() - startTime) + "ms");
	
	startTime = System.currentTimeMillis(); 
	for (int j = 0; j < loopTime; j++) {
		String str = i.toString();
	}
	System.out.println("Integer.toString()：" 
                       + (System.currentTimeMillis() - startTime) + "ms");
	
	startTime = System.currentTimeMillis(); 
	for (int j = 0; j < loopTime; j++) {
		String str = i + "";
	}
	System.out.println("i + \"\"：" 
                       + (System.currentTimeMillis() - startTime) + "ms");
}
```
运行结果为：

```java
String.valueOf()：13ms
Integer.toString()：4ms
i + ""：39ms
```
所以以后遇到把一个基本数据类型转为String的时候，优先考虑使用toString()方法。至于为什么，很简单：
1、String.valueOf()方法底层调用了Integer.toString()方法，但是会在调用前做空判断
2、Integer.toString()方法就不说了，直接调用了
3、i + “”底层使用了StringBuilder实现，先用append方法拼接，再用toString()方法获取字符串
三者对比下来，明显是2最快、1次之、3最慢。

#### 26. 使用最有效率的方式去遍历Map
遍历Map的方式有很多，通常场景下我们需要的是遍历Map中的Key和Value，那么推荐使用的、效率最高的方式是：

```java
public static void main(String[] args) {
	Map<String, String> hm = new HashMap<String, String>();
	hm.put("111", "222");
	Set<Map.Entry<String, String>> entrySet = hm.entrySet();
	Iterator<Map.Entry<String, String>> iter = entrySet.iterator(); 
	while (iter.hasNext()) {
		Map.Entry<String, String> entry = iter.next();
		System.out.println(entry.getKey() + "," + entry.getValue());
	}
}
```
如果你只是想遍历一下这个Map的key值，那用”Set<String> keySet = hm.keySet();”会比较合适一些。

### 参考

* [35 个 Java 代码性能优化总结][1]

[1]: http://www.jianshu.com/p/436943216526