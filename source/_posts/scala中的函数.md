---
title: scala中的函数
date: 2018-07-23 14:31:46
tags:
- scala
categories:
- Scala
keywords:
- scala
- 函数
description: 在各类blog及scala书籍中看到的关于函数最多的一句话是“函数是一等公民”，可以看出函数在scala中举足轻重的地位，因此很有必要去全面的掌握它
---
在各类blog及scala书籍中看到的关于函数最多的一句话是
> 函数是一等公民

可以看出函数在scala中举足轻重的地位，因此很有必要去全面的掌握它

## scala函数基础
#### 定义函数
* 匿名函数
``` shell
scala> (x: Int) => x + 1
res0: Int => Int = $$Lambda$1026/1792711692@7412ed6b
```
  固定写法：左边是参数列表，中间符号=>，右边是函数体    
  匿名函数无函数名，也无需给出函数的结果类型

* 函数字面量
``` shell
scala> val f = (x: Int) => x + 1
scala> f(10)
res0: Int = 11    
```
  其中表达式有半部分为一个匿名函数，函数作为一个变量(或不变量)，用变量名+参数表方式调用，与普通方法的调用几乎相同。    
  ** 需要注意的是函数字面量声明函数时不需要写出函数的返回值类型，函数体的最后一条表达式的结果会作为返回值返回 **

* 函数值
函数字面量与函数值的关系就像类与对象的关系。函数字面量其实就是一个实现了trait Function*的函数类，在运行时实例化了一个函数类，这个实例就是函数值

#### 函数的本质
scala不像java有基本类型与引用类型之分，scala中一切皆对象，包括函数也是对象。scala中定义了一系列trait，Function0 - Function22，其中0-22表示参数的个数(至于为什么是到22，理论上讲程序中不会出现多于22个参数的函数)。而函数正是Function*的实例。如`val f = (x: Int) => x + 1`其实就是Function1的实例。下面看下Function*的源码，以Function3为例
``` scala
/** A function of 3 parameters.
 *
 */
trait Function3[-T1, -T2, -T3, +R] extends AnyRef { self =>
  /** Apply the body of this function to the arguments.
   *  @return   the result of function application.
   */
  def apply(v1: T1, v2: T2, v3: T3): R

  /** Creates a curried version of this function.
   *
   *  @return   a function `f` such that `f(x1)(x2)(x3) == apply(x1, x2, x3)`
   */
  @annotation.unspecialized def curried: T1 => T2 => T3 => R = {
    (x1: T1) => (x2: T2) => (x3: T3) => apply(x1, x2, x3)
  }

  /** Creates a tupled version of this function: instead of 3 arguments,
   *  it accepts a single [[scala.Tuple3]] argument.
   *
   *  @return   a function `f` such that `f((x1, x2, x3)) == f(Tuple3(x1, x2, x3)) == apply(x1, x2, x3)`
   */
  @annotation.unspecialized def tupled: Tuple3[T1, T2, T3] => R = {
    case Tuple3(x1, x2, x3) => apply(x1, x2, x3)
  }
  override def toString() = "<function3>"
}
```
可以看到其中有一个apply方法(先忽略其他方法)，接受三个泛型参数，所以当我们调用`f(1)`时，其实是调用了Function1实例的apply方法。

#### 函数与方法
在学习过程中，还有一点很困扰，就是我们在类中用def定义的是不是函数，如果不是函数，它又是什么，跟函数又有什么区别呢？
首先呢使用def 关键字定义的是方法，不是函数，虽然在实际应用中几乎没有差别，但还是需要了解两者的不同的    

** 方法 **  
定义在类中，作为某个对象的成员方法，使用def关键字定义
``` shell
scala> def a(x: Int):Int = x + 1
a: (x: Int)Int
```

** 两者的区别 **
* 方法使用def关键字定义，而函数不用(函数使用val/var或干脆匿名)
* 方法是类的一部分，函数是对象可以赋值给一个val/var
* 函数作为对象可以像任何其他数据类型一样被传递和操作，而方法不行，如果想要传递方法，则需要把方法转换为函数
* 定义方法时，如果没有参数，则参数表可以省略不写(即方法名后不写())，而函数不行
* 定义函数时参数列表后不能声明结果类型，如下面的写法会编译错误
``` shell
scala> f(x: Int):Int => x + 1
<console>:1: error: ';' expected but '=>' found.
       f(x: Int):Int => x + 1
                     ^
```

** 两者的转换 **
* 第一种情况，方法直接转为函数    
不能将方法直接声明为一个val/var，如下面的写法会编译错误
``` shell
scala> def m(x: Int):Int = x + 1
scala> val f = m
<console>:12: error: missing argument list for method m
Unapplied methods are only converted to functions when a function type is expected.
You can make this conversion explicit by writing `m _` or `m(_)` instead of `m`.
       val f = m
```
  正确写法
  ``` shell
  scala> def m(x: Int):Int = x + 1
  scala> val f = m _
  f: Int => Int = $$Lambda$1069/727861082@4990b335
  ```
  这个转换的其实是使用了部分应用(Partial Applied Function)，下面将会讲到

* 第二种情况，需要函数的地方使用方法
这种情况下不需要使用下划线，方法会被自动转换为函数，称之为eta转换。关于eta-expansion与eta-conversion的解释可以参考[王宏江-scala中的eta-conversion](http://hongjiang.info/scala-eta-conversion/)与[王宏江-再谈eta-conversion与eta-expansion](https://hongjiang.info/eta-conversion-and-eta-expansion/)

## scala函数进阶
#### 高阶函数
引用[维基百科](https://zh.wikipedia.org/wiki/%E9%AB%98%E9%98%B6%E5%87%BD%E6%95%B0)中高阶函数的定义
> 在数学和计算机科学中，高阶函数是至少满足下列一个条件的函数：    
* 接受一个或多个函数作为输入    
* 输出一个函数    

定义应该很容易理解，也就是函数可以接受另一个函数作为参数，或者函数的结果类型为函数类型。    
* 第一种：如数学中的函数`f(x) = x + 1`， `g(x) = x * 2`，则`f(g(x)) = (x * 2) + 1`, 即为高阶函数。用代码实现即为
``` scala
def g(x: Int) = x * 2
def f(x: Int, g: Int => Int) = g(x) + 1
```

* 第二种：结果类型为函数类型
``` scala
def g(x: Int):(Int => Int) = {
  val k = x * 2
  (y: Int) => y + k + 1
}
```

#### 函数的部分应用
先看下代码如何实现函数的部分应用吧
``` shell
scala> def sum(x: Int, y: Int, z: Int) = x + y + z
sum: (x: Int, y: Int, z: Int)Int

scala> val a = sum _
a: (Int, Int, Int) => Int = $$Lambda$1055/565627330@309cedb6

scala> a(1,2,3)
res0: Int = 6

scala> val b = sum(1, _: Int, _: Int)
b: (Int, Int) => Int = $$Lambda$1060/605472344@6d5f4900

scala> b(2,3)
res1: Int = 6
```
上面代码中出现了两种写法
* 写法一   
  `val a = sum _`    
  这行代码表示sum方法的三个参数都为给出，整个参数表都用占位符`_`代替，所以这行代码的效果其实就是讲方法sum转换为了函数a。与之相同的写法还有    
  `val a = sum(_)`
* 写法二    
  `val b = sum(1, _: Int, _: Int)`
  这行代码表示，已知三个参数中的x，y和z未给出，用占位符`_`代替，最后返回一个包含了两个参数的函数b    

部分应用从数学角度理解比较容易，如上面例子中的sum，其实对应于数学中的`sum(x,y,z) = x + y + z`，这是一个三元函数(既有三个未知数)，而写法二对sum的部分应用，相当于我们现在得知`x = 1`，则带入函数得`sum(y,z) = 1 + y + z`，也就是函数f消去了未知元x。** 整体来看函数的部分应用其实就是数学中的消元。 ** 而写法是对未消去任何参数的一种简写

#### 函数的柯里化
在scala中函数函数可以有多个参数列表，而函数柯里化就是将函数的多参数列表转换为多个多参数列表。写法如下
``` shell
scala> def sum(x: Int, y: Int) = x + y
sum: (x: Int, y: Int)Int

scala> val a = sum _
a: (Int, Int) => Int = $$Lambda$1075/571435580@5eb041b5

scala> val b = a.curried
b: Int => (Int => Int) = scala.Function2$$Lambda$1076/1263872787@524dd373
```
柯里化是通过调用函数对象的curried方法实现的。也就是说调用的是Function*中的curried方法(Function2-Function22拥有curried方法，Function0及Function1没有，因为无参函数和但参函数不需要也不能柯里化)，以Function2为例，我们看下源码的实现
``` scala
trait Function2[-T1, -T2, +R] extends AnyRef { self =>
  ...
  @annotation.unspecialized def curried: T1 => T2 => R = {
    (x1: T1) => (x2: T2) => apply(x1, x2)
  }
  ...
}
```
curried的实现是将一个多参函数转换为一个单参函数的函数链，对于上面的例子来说就是把`sum(x,y)`转换成了`sum(x)(y)`，`sum(1)(2)`也就是对两个单参函数的依次调用，而`sum(1)`或者`sum(2)`其实相当于对sum函数的部分应用，返回值是一个单参函数

#### 闭包
引用别人blog中对闭包的定义
>闭包是一个函数，返回值依赖于声明在函数外部的一个或多个变量。

示例
``` shell
scala> var more = 1
more: Int = 1

scala> val add = (i: Int) => i + more
add: Int => Int = $$Lambda$1045/640294829@6be6931f
```
上面例子中add函数的计算依赖于自由变量more，也就是闭包了。闭包很像函数字面量，不同的是，闭包运算的过程不只依赖于输入参数，还需要依赖函数之外的一个或多个自由变量。

关于闭包的详细解释可以参考[知乎-什么是闭包](https://zhuanlan.zhihu.com/p/21346046)，引用博文中的总结
> 最简洁、直击要害的回答，我能想到的分别有这么三句（版权属于 ：
* 闭包是一个有状态（不消失的私有数据）的函数。
* 闭包是一个有记忆的函数。
* 闭包相当于一个只有一个方法的紧凑对象（a compact object）。

上面这三句话是等价的

#### 嵌套函数
scala允许在函数内部定义函数，称之为局部函数。它的作用域仅限于外部函数，其他位置无法访问到内部函数，可以达到控制访问的效果
示例
``` scala
val f = (x:Int) => {
  val innerAdd = (i: Int) => i + 1
  innerAdd(x) + 1
}
```

## 写在最后
关于scala中的函数，就先写这么多吧，等到后面开发应用到其他的知识，在作补充。关于函数的部分应用和柯里化的实际意义，在[scala函数式编程](../../21/scala函数式编程/)那篇里写出来
，这里就不写了
