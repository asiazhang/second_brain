# 为什么使用kotlin替代java8

知乎上有一个段子很有意思，提问是[JVM上的各种语言分别解决了Java的什么问题](https://www.zhihu.com/question/48633827/answer/254037101)？其中一个回答我点了赞：

> 综合各位答主的意思，就是：
> Scala：想解决Java表达能力不足的问题
> Groovy：想解决Java语法过于冗长的问题
> Clojure：想解决Java没有函数式编程的问题
> Kotlin：**想解决Java**

只要用过kotlin的同学，此刻肯定会心一笑。

## 廉颇老矣，尚能饭否

Java从诞生到现在，已经有十几年的历史了。从Java8的语法设计来看，目前部分场景已经跟不上时代。

1. 没有自动类型推导
2. 没有属性概念，必须要写`getter`/`setter`
3. 没有好用的函数式编写方法
4. 没有可空类型
5. 没有表达式直接作为返回值处理的方式
6. 没有扩展函数支持

只要到网络上随便搜搜，就能看到一大堆吐槽，说java的代码就像**老太婆的裹脚布——又长又臭**。

提升软件的开发效率是永恒的追求。JVM上也涌现出了一堆新语言，对Java的地位进行挑战。经过分析我选择了用Kotlin替代Java。

## Why Kotlin

为什么不使用JVM上的其他语言呢？我自己选择Kotlin的原因如下：

1. 从Java平滑过度到Kotlin。Kotlin语法差异跟Java相比起来较小，学习很容易，基本半天就搞定了。而其他的JVM语言如`Scala`/`Groovy`/`Clojure`学习成本都较高，跟Java语言有明显的差异。Kotlin设计的时候非常明显就是要解决Java在软件开发中最为影响开发效率的地方，因此才有Kotlin想解决Java这个段子🐶。
2. Kotlin跟Java可以100%互相调用。
3. 提供的特性都是解决Java痛点的地方，没有太标新立异，可以认为是一个`Better Java`。


## Kotlin解决了Java哪些痛点

### 数据类

### 自动类型推导

### 可空类型

### 函数式编程

## 迁移结果

