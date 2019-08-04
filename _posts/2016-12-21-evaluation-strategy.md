---
layout:     post
title:      "Evaluation Strategy"
subtitle:   "理解值传递和引用传递"
header-img: img/pic/2015/10/kanasi2.jpg
tags: [程序设计, javascript, ruby, go, java]

---

### 求值策略和内容类型

求值策略 [Evaluation_strategy](https://en.wikipedia.org/wiki/Evaluation_strategy), 如值传递/引用传递, 属于函数调用时参数的求值策略, 这是对调用函数时, 求值和传值的方式的描述, 和传递的内容的类型没有直接关系.

传递的内容类型(Value Content), 如值类型/引用类型，是用于区分两种内存分配方式，值类型在调用栈上分配，引用类型在堆上分配。

一个描述参数求值策略, 一个描述内存分配方式, 两者之间无任何依赖或约束关系。

---

### 求值策略的区别

求值策略(值传递和引用传递)的关注的点在于:

* 求值的时机:  传入的参数(表达式)在调用函数的过程中，求值的时机、值的形式的选取等问题

  求值时机可能是在调用前, 调用后(用到才求值, 类似Lazy Loading)

* 传递形式: 主要关注有无副本

| 求值策略                    | 求值时机 | 传递方式         | 根本区别     | 影响                     |
|-----------------------------|----------|------------------|--------------|--------------------------|
| 值传递(Pass by value)       | 调用前   | 原值的副本       | 会创建副本   | 函数无法改变(change)原值 |
| 引用传递(Pass by reference) | 调用前   | 原值(无副本)     | 不会创建副本 | 函数可以改变(change)原值 |
| 名传递(Pass by name)        | 调用后   | 与值无关的一个名 | -            | -                        |


---

### 值传递+mutate

Go 是值传递:

```go
package main
import "fmt"

type Person struct {
  name string
}

func change(person Person) {
  person.name = "zhong"
}

func main() {
  me := Person{"fox"}
  change(me)
  fmt.Println(me.name) //fox; 没被改变
}
```

可以看到me.name经过change后没有变化, 看起来证明了「值传递无法改变原值」, 其实不然, 见下文.

Ruby 是也是值传递, 我们来看一个容易误解的例子:

```ruby
def change(person)
  person[:name] = 'zhong'
end

me = { name: 'fox' }

change(me)
puts(me[:name]) # zhong; 被改变了
```

同样是值传递, Ruby代码和go代码基本一致, 但是为什么Ruby对象me被改变了呢?

---

### 理解值传递

「改变」一词在函数中通常有2种不同的含义:

* mutate: 表示内容属性的修改, 但是内容引用不变, 如在ruby中:
  > `person[:name] = 'zhong'`
* change: 表示直接改变量指向, 表现通常是进行重新赋值:
  >  `person = { name: 'zhong' }`

值传递无法改变原值, 说的「改变」是指的「change」, 而不是「mutate」, 上面Go 和 Ruby 代码中me的name改变属于内容变化(mutate), 函数中的mutate能否改变原值指向的原始对象还和传递的内容类型(Content Value)有关:

| 求值策略 | Content Value    | 函数内mutate(如果对象可变) | 函数内change   | 代表                                                                         |
|----------|------------------|----------------------------|----------------|------------------------------------------------------------------------------|
| 值传递   | 值类型(被复制)   | 不影响原始对象             | 不影响原始对象 | Go 传递struct,array等<br>Javascript传递基础类型<br>Ruby立即值Symbol,Fixnum   |
| 值传递   | 引用类型(被复制) | 影响原始对象               | 不影响原始对象 | Go 传递map,slice,channel<br>Javascript传递Object<br>Ruby传递非立即值<br>Java |
{: class="minitable"}

<style>
.minitable {font-size: 13px;}
</style>

上面的例子中, Go传递的struct是Content Value是值类型, 而Ruby传递的hash是引用类型.

因为Ruby 是一门非常纯粹的面向对象的语言, 当我们在Ruby中使用对象时, 其实是在使用对象的引用(这个引用和引用传递没有关系, 文章最开始进行了澄清), Ruby的求值策略是值传递, 但是传递的值是对象引用. 以上Ruby例子中, 从实参me到形参person, 存在副本复制, 只是这个复制的内容是对象引用, 在change函数中, 无法改变这个对象引用, 改变的只是这个对象引用对应的对象属性, 因此上面的例子还是值传递.

其实在Go中, 也存在「值传递」时, 传递的值是类似「引用」的情况, 那就是传递map, slice, channel时, 可以认为是引用(其实还是值, 只是数据结构有很多指针), 我们用map重写上面Golang的例子:

```go
package main

import "fmt"

type Person map[string]string

func change(person Person) {
  person["name"] = "zhong"
}

func main() {
  me := Person{"name": "fox"}

  change(me)
  fmt.Println(me["name"]) //zhong, 被改变了
}
```

值传递(Evaluation)一定会复制副本, 不会影响原值, 这个副本是引用类型还是值类型是由传递的对象决定(Value Content), Value Content如果是引用类型, 在函数mutate时会影响原始对象(因为副本本身就是一个对象引用)

由于这个描述太绕, 于是对于Java, Ruby, JavaScript, Python等语言使用的这种求值策略(值传递+引用类型)，起了一个更贴切名字，叫[Call by sharing](https://en.wikipedia.org/wiki/Evaluation_strategy#Call_by_sharing)

---

### 值传递+change

值传递无法改变原值, 说的「改变」是指的「change」, 来看看change的实例:

```ruby
def change(person)
  person = { name: 'zhong' }
end

me = { name: 'fox' }

change(me)
puts(me[:name]) # fox; 没被改变
```

对于值传递的Golang, Javascript等也有同上的例子, 可以自行试试.

---

### 对比引用传递

C# 里既支持值传递, 又支持引用传递, 比对起来看看:

通过值传递调用:

```cs
using System;
public class Person
{
  public Person(string name)
  {
    Name = name;
  }
  public string Name { get; set; }
}

public class testEvaluation
{
   public static void Change(Person person) //值传递方式, 存在复制副本
   {
      person = new Person("zhong"); //仅副本变化, 原始实参没有变化
   }
  
   public static void Main(string[] args)
   {
      Person me = new Person("fox");
      testEvaluation.Change(me);
      Console.WriteLine(me.Name); //fox, 值传递, 原始实参没有变化
   }
}
```

通过引用传递调用:

```cs
using System;
public class Person
{
  public Person(string name)
  {
    Name = name;
  }
  public string Name { get; set; }
}

public class testEvaluation
{
   public static void Change(ref Person person) //引用传递方式, 没有复制副本
   {
      person = new Person("zhong"); //改变了原始值
   }
  
   public static void Main(string[] args)
   {
      Person me = new Person("fox");
      testEvaluation.Change(ref me);
      Console.WriteLine(me.Name); //zhong, 原始值被改变
   }
}
```

---

### 总结

Evaluation 影响的是传入的「原值」, 而对原值指向的原始对象的影响, 还需要结合Value Content和改变方式:

<table><tbody>
  <tr>
    <th>Evaluation</th>
    <th >对原值的影响</th>
    <th >Value Content</th>
    <th >实例</th>
    <th >改变方式</th>
    <th >对原值指向的原始对象的影响</th>
  </tr>
  <tr>
    <td  rowspan="4">值传递</td>
    <td  rowspan="4">对原值复制<br>无法改变原值</td>
    <td  rowspan="2">值类型</td>
    <td >Go<br>JavaScript_primitive<br>Ruby_Symbol_Fixnum</td>
    <td >change</td>
    <td >无法改变</td>
  </tr>
  <tr>
    <td >Go</td>
    <td >mutate</td>
    <td >无法改变</td>
  </tr>
  <tr>
    <td  rowspan="2">引用类型<br>Call by sharing</td>
    <td  rowspan="2">Ruby<br>Javascript_Object<br>Go_map_channel_slice</td>
    <td >change</td>
    <td >无法改变</td>
  </tr>
  <tr>
    <td >mutate</td>
    <td >可以改变</td>
  </tr>
  <tr>
    <td colspan="1">引用传递</td>
    <td colspan="1">没有复制<br>可以改变原值</td>
    <td colspan="1">值类型/引用</td>
    <td colspan="1">C#</td>
    <td colspan="1">change/mutate</td>
    <td colspan="1">可以改变</td>
  </tr>
</tbody></table>



### 参考资料

* [Evaluation_strategy](https://en.wikipedia.org/wiki/Evaluation_strategy)
* [为什么 Java 只有值传递，但 C# 既有值传递，又有引用传递，这种语言设计有哪些好处？](https://www.zhihu.com/question/20628016/answer/28970414)
