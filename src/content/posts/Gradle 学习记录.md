---
title: Gradle 学习记录
published: 2024-07-28
category: 文章
tags:
  - Java
---

<!-- @import "[TOC]" {cmd="toc" depthFrom=1 depthTo=6 orderedList=false} -->

<!-- code_chunk_output -->

- [前言](#前言)
- [Groovy 常见特性](#groovy-常见特性)
- [Groovy 不常见特性](#groovy-不常见特性)
- [Gradle 不同于 Groovy 的语法](#gradle-不同于-groovy-的语法)

<!-- /code_chunk_output -->

## 前言

因为实习公司用的 Java 项目构建系统是 Gradle，不得不临时学习一下，记录一下重点。

对于 Gradle 的第一印象就是和 SCons 很像，都是在开始构建前要先用脚本语言构建“构建目标”。只是 SCons 用的是 python，Gradle 用的是 Groovy。

在学 Gradle 之前我打算先了解了一下 Groovy，Groovy 的设计目标就是成为一门让 Java 程序员能快速上手的脚本语言，因此语法基本和 Java 一致，但还是有一些区别。

主要是通过阅读[Groovy 官方文档](https://docs.groovy-lang.org/docs/groovy-4.0.22/html/documentation)来学习的 Groovy。

## Groovy 常见特性

个人认为 Groovy 和 Java 的区别主要区别有以下几点

1.  可选的变量类型，可以直接用 def 声明变量。
2.  由于变量类型可选，方法的返回值也是可选的，同样用 def 表示。
3.  Map 成为了基础类型，可以表示为 `['TopicName'：'Lists'，'TopicName'：'Maps']` 的形式，用`map[xxx]` 或 `map.xxx` 访问，`[:]` 表示一个空 Map。有趣的是，当 key 为字符串时，支持以任意形式的字符串进行访问，例如 `map.'ab cd'`, `map."$key"`。
4.  函数可以有默认参数了。
5.  函数调用可以省略圆括号，直接用空格隔开函数名和参数（可以有多个），如 `println 'Hello'`。若接受单个参数，且参数为 Map 类型，还可省略 Map 的方括号，例如
    ```
    def func(nums) {
        println "a + b = ${nums.a + nums.b}"
    }
    func a: 1, b: 2
    ```
6.  函数只有一行时，可以不写 return，会将该行直接视为返回值。
    ```groovy
    def getHello() { 'Hello' }
    println getHello()
    ```
7.  引入了闭包的概念，行为和 lambda 表达式很相似但不完全相同，和，默认按引用捕获所有用到的变量。

    ```groovy
    def val = 0
    def plus = {val++ }
    plus()
    println val // 输出 1
    ```

    如果要接受参数，则写作

    ```groovy
    def add = {a, b -> a + b}
    println add(1, 2)   // 输出 3
    ```

    如果只接受一个参数，还可写作

    ```groovy
    def myprint = { print it }  // it 是关键字，表示传入的唯一参数
    myprint 'Hello'
    ```

    闭包中还有一些关键字，如 `this`, `owner`, `delegate `等，其中 `delegate` 比较重要。

    在 Gradle 中常见到类似如下代码的配置块

    ```groovy
    copy {
        into layout.buildDirectory.dir("tmp")
        from 'custom-resources'
    }
    ```

    此处 copy 实际上是一个接受一系列参数的函数，最后一个参数必须是闭包，在函数中会为该闭包指定 delegate 对象然后调用。这里的 into 和 from 调用的就是 delegate 对象的方法，关于 delegate 的细节可以参考[这篇文档](https://docs.groovy-lang.org/docs/groovy-4.0.22/html/documentation/#_delegate_of_a_closure)。
    再举个常见的例子，要设置 maven 仓库，常常会用到这样的代码

    ```groovy
    repositories {
        mavenCentral()
    }
    ```

    这里的 repositories 会将闭包的 delegate 设置为 [RepositoryHandler](https://docs.gradle.org/current/dsl/org.gradle.api.artifacts.dsl.RepositoryHandler.html)，其中就包含了 `mavenCentral` 方法。

8.  范围运算符，默认左闭右闭
    - 正序 `0..1`
    - 倒序 `4..2`
    - 左闭右开 `0..<1`
    - 字符范围 `'a'..'z'`
9.  有多种形式的字符串
    - 单引号表示纯字符串，和 Java 中的字符串相同。
    - 双引号表示插值字符串，可以在字符串中通过 `'name = ${student.name}'` 的形式进行插值。特别的，如果插值表达式为单个变量名，可以省略花括号，如 `'a = $a'`。插值还可以写作 `'name = ${ -> student.name }'` 的闭包形式，这样该字符串每次被使用时都会按 `student.name` 的最新状态进行插值。
    - 三个引号表示多行字符串，其中的换行符都会被保留。如果是单引号则不可插值，是双引号则可插值。有时为了书写美观，会希望忽略首尾行行末的换行符，可以使用反斜杠来转义。
    ```groovy
    def str = '''\
        some words
        other words\
    '''
    ```
    - 斜杠表示原始字符串，和 JS 中的基本一致，常用于书写正则表达式。
    - 美元斜杠字符串，很少使用。
10. 支持一系列快捷的正则表达式操作。
11. 支持 `?.` 运算符进行安全访问。
12. 可以使用 `*.` 运算符访问列表中元素的成员变量，得到一个新的列表，例如

```groovy
class Make {
    String name
    List<Model> models
}

@Canonical  // 类似 Lombok 中的 @Data 注解
class Model {
    String name
}

def cars = [
    new Make(name: 'Peugeot',
             models: [new Model('408'), new Model('508')]),
    new Make(name: 'Renault',
             models: [new Model('Clio'), new Model('Captur')])
]

def makes = cars*.name
assert makes == ['Peugeot', 'Renault']

def models = cars*.models*.name
```

12. `*` 用于传播 List，`*:` 用于传播 Map。

    ```groovy
    def l1 = [1]
    def l2 = [*l1, 2, 3]    // l2 的内容为 [1, 2, 3]

    def m1 = {a: 1}
    def m2 = {*:m1, a: 2, b: 2} // m2 的内容为 {a: 2, b: 2}
    ```

## Groovy 不常见特性

还有一些编写 Gradle 脚本基本用不上的特性，但既然写都写了就贴出来吧

1. 访问类字段时，会优先通过 get/set 方法来访问字段。若就是要直接访问，则需使用 `@` 运算符。
   ```groovy
   class User {
       public final String name
       User(String name) { this.name = name}
       String getName() { "Name: $name" }
   }
   def user = new User('Bob')
   println user.name       // 通过 getName 访问，输出 Name: Bob
   println user.@name  // 直接访问，输出 Bob
   ```
2. 使用 `.&` 运算符引用类成员变量，而不是 Java 中的 `::` 运算符。
3. 引入了 `trait` 的概念，用于为接口提供部分方法的实现。trait 的大部分特性和接口基本相同，主要的区别有以下几点

   - 可以 _堆叠_ trait，当某个类实现了多个 trait，且这些 trait 包含某个相同方法时，可以在这些方法中通过 super 访问前一个 trait ，顺序由 implement 的顺序决定。
     例如下面这段代码中

     ```groovy
     class HandlerWithLogger implements DefaultHandler, LoggingHandler {}
     ```

     LoggingHandler 位于 DefaultHandler 后面。

   - 接口只能拥有 public static 字段，而 trait 可以拥有任意访问权限的字段。当出现字段名或方法名冲突时，后面的 trait 的会覆盖前面的默认值和实现，顺序的定义同前。

## Gradle 不同于 Groovy 的语法

Gradle 脚本的语法和 Groovy 并不完全相同，要编写 Gradle 脚本，还需要了解官方文档的[这一小节](https://docs.gradle.org/current/userguide/groovy_build_script_primer.html)。

主要的区别如下：

1. 在一个 Gradle 脚本中，所有代码可以被视为是在一个闭包中运行的，该闭包的 delegate 被设为了 [Project 对象](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html)。因此，在 Gradle 脚本中访问的没有限定符的变量和函数都属于该 Project 对象，因此脚本中执行的所有常见操作都可以在该文档中找到用法。
2. 如果希望使用 Project 对象不具有的属性，需要通过往 ext 中存入值来实现。这里的 ext 其实就是 Project 对象的一个实现了 Map 接口的属性，ext.xxx 使用的是 Groovy 中访问 Map 的语法（参考常见特性中的第三条）。可能有人会问为什么不直接 def 一个变量，因为正如上条所说，脚本可以视为一个闭包，你在脚本中定义的变量都是处于该闭包中的，别的地方看不到，而 Project 的 ext 是可以在其他地方看到的。
3. 之前已经提到过，Gradle 有一些常见的配置块，这些配置块都对应 Project 中的某个函数，具体来说包含以下几种
   ```
   allprojects { }
   artifacts { }
   buildscript { }
   configurations { }
   dependencies { }
   repositories { }
   sourceSets { }
   subprojects { }
   publishing { }
   ```
   他们都可以以特定的形式进行嵌套配置，以形成结构化数据的形式，基本都是以传入闭包，给闭包设置 delegate，在闭包中配置 delegate 的属性的形式。各个配置块的使用方式可以参考 Project 文档的 [Script block details](https://docs.gradle.org/current/dsl/org.gradle.api.Project.html#N16A7D) 小节。
   plugins 也是一个配置块，但是其属于 Gradle 的预处理阶段，包含一些额外的语法糖，因此可以使用 version 等非 Groovy 标准语法指定版本号。
