---
layout: post
title: Java异常处理实践
description: Java异常处理的基本认知，以及重要的异常处理规约。
category: blog
---

## 一、基本认知

### 1. 何为异常

- Exception是程序正常运行中，可以预料的意外情况，可能并且应该被捕获和进行相应处理。
- Error是指在正常情况下，不大可能出现的情况，绝大部分的Error都会导致程序（比如JVM自身）处于非正常的、不可恢复状态。
- Exception的分类：checked异常在代码里必须显示地进行捕获处理，如IOException等；unchecked异常就是所谓的运行时异常（RuntimeException），比如NullPointerException、ArrayIndexOutOfBoundsException等，通常是可以编码避免的逻辑错误，不会在编译期强制要求捕获。

### 2. 异常编写原则

开发者每编写一个异常，脑中需要自动浮现以下三个问题：

- 出了什么错：明确异常类型
- 在哪里出错：明确异常产生范围
- 为什么出错：明确异常处理方式



## 二、异常处理规约

#### 1. Use exceptions only for exceptional conditions.

>仅在异常场景中使用异常，勿将异常用于程序流程控制。

```java
// Horrible abuse of exceptions. Don't ever do this!
try {
     int i = 0;
     while(true)
         range[i++].climb();
} catch(ArrayIndexOutOfBoundsException e) {
    // ...
}
```

- 混淆代码用意
- 降低代码执行效率：现代JVM的实现中，基于异常的语句执行效率低于控制语句（if/else、switch）
- 容易掩盖实际的程序错误，增加调试难度

#### 2. 仅捕获有必要的代码段，尽量不要一个大的try包住整段的代码。

- try-catch代码段会产生额外的性能开销，或者换个角度说，它往往会影响JVM对代码进行优化。
- Java每实例化一个Exception，都会对当时的栈进行快照，如果发生的非常频繁，这个开销可就不能被忽略了。
- 当服务出现反应变慢、吞吐量下降的时候，检查发生最频繁的Exception也是一种思路。

#### 3. 不要忽略或延迟处理异常，必须就地以合理的方式解决之。

- 捕获异常后，需要怎么处理？根据上下文进行恢复操作、保留异常的cause信息、直接抛出或者构建新的异常抛出。

#### 4. 尽量不要捕获类似Exception这样的通用异常，而是应该捕获特定异常。

- 软件工程是门协作的艺术，所以我们有义务让自己的代码能够直观地体现出尽量多的信息，而泛泛的Exception之类，恰恰隐藏了我们的目的。
- 我们也要保证程序不会捕获到我们不希望捕获的异常。比如，你可能更希望RuntimeException被扩散出来，而不是被捕获。

#### 5. Prefer unchecked exceptions for all programmatic errors.

- Checked Exception的假设是我们捕获了异常，然后恢复程序。但是，其实大多数情况下根本就不可能恢复。
- 除非保证异常可恢复（比如和环境相关的IO、网络等），否则尽量不使用Checked Exception。

#### 6. 自定义异常在保证诊断信息足够的同时，还需考虑其他问题。

- 避免包含敏感信息，比如机器名、IP、端口、用户数据等。
- 最好使用产品日志（如log4j、logback等），详细地将诊断信息输出到日志系统里，而非标准出错（如printStackTrace）。

#### 7. 对于分布式系统，要把异常处理当作正常功能来看待和处理。



## 相关资料

* [《Effective Java, 3nd Edition》][1]
* [极客时间：Java核心技术36讲][2]

[1]: https://www.amazon.cn/dp/0134685997/ref=sr_1_5?ie=UTF8&qid=1548813422&sr=8-5&keywords=effective+java
[2]: https://time.geekbang.org/column/intro/82

