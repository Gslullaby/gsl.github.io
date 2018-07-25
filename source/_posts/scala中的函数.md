---
title: scala中的函数
date: 2018-07-23 14:31:46
tags:
- scala
keywords:
- scala
- 函数
description:
---
在各类blog及scala书籍中看到的关于函数最多的一句话是
> 函数是一等公民

可以看出函数在scala中举足轻重的地位，因此很有必要去全面的掌握它

## scala函数基础
#### 定义函数
* 匿名函数
``` bash
scala> (x: Int) => x + 1
res0: Int => Int = $$Lambda$1026/1792711692@7412ed6b
```
  固定写法：左边是参数列表，中间符号=>，右边是函数体    
  匿名函数无函数名，也无需给出函数的结果类型

* 函数字面量
``` bash
scala> val f = (x: Int) => x + 1
scala> f(10)
res0: Int = 11    
```
  其中表达式有半部分为一个匿名函数，函数作为一个变量(或不变量)，用变量名+参数表方式调用，与普通方法的调用几乎相同。    
  ** 需要注意的是函数字面量声明函数时不需要写出函数的返回值类型，函数体的最后一条表达式的结果会作为返回值返回 **

#### 函数的本质
scala不像java有基本类型与引用类型之分，scala中一切皆对象，包括函数也是对象。scala中定义了一系列trait，Function0 - Function22，表示参数的个数(至于为什么是到22，理论上讲程序中不会出现多于22个参数的函数)。而函数正是Function*的实例。如`val f = (x: Int) => x + 1`其实就是Function1的实例。下面看下Function*的源码，以Function3为例
``` java
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
``` bash
scala> def a(x: Int):Int = x + 1
a: (x: Int)Int
```

** 两者的区别 **
* 方法使用def关键字定义，而函数不用(函数使用val/var或干脆匿名)
* 方法是类的一部分，函数是对象可以赋值给一个val/var
* 函数作为对象可以像任何其他数据类型一样被传递和操作，而方法不行，如果想要传递方法，则需要把方法转换为函数
* 定义方法时，如果没有参数，则参数表可以省略不写(即方法名后不写())，而函数不行
* 定义函数时参数列表后不能声明结果类型，如下面的写法会编译错误
``` bash
scala> f(x: Int):Int => x + 1
<console>:1: error: ';' expected but '=>' found.
       f(x: Int):Int => x + 1
                     ^
```

** 两者的转换 **
* 第一种情况，方法直接转为函数    
不能将方法直接声明为一个val/var，如下面的写法会编译错误
``` bash
scala> def m(x: Int):Int = x + 1
scala> val f = m
<console>:12: error: missing argument list for method m
Unapplied methods are only converted to functions when a function type is expected.
You can make this conversion explicit by writing `m _` or `m(_)` instead of `m`.
       val f = m
```
  正确写法
  ``` bash
  scala> def m(x: Int):Int = x + 1
  scala> val f = m _
  f: Int => Int = $$Lambda$1069/727861082@4990b335
  ```
  这个转换的其实是使用了部分应用(Partial Applied Function)，下面将会讲到

* 第二种情况，需要函数的地方使用方法
这种情况下不需要使用下划线，方法会被自动转换为函数，称之为ETA展开，关于ETA展开，在另一篇里写吧，这里不详述了

## scala函数进阶
#### 高阶函数
引用[维基百科](https://zh.wikipedia.org/wiki/%E9%AB%98%E9%98%B6%E5%87%BD%E6%95%B0)中高阶函数的定义
> 在数学和计算机科学中，高阶函数是至少满足下列一个条件的函数：    
* 接受一个或多个函数作为输入    
* 输出一个函数    

定义应该很容易理解，也就是函数可以接受另一个函数作为参数，或者函数的结果类型为函数类型。    
* 第一种：如数学中的函数`f(x) = x + 1`， `g(x) = x * 2`，则`f(g(x)) = (x * 2) + 1`, 即为高阶函数。用代码实现即为
``` bash
def g(x: Int) = x * 2
def f(x: Int, g: Int => Int) = g(x) + 1
```

* 第二种：结果类型为函数类型
``` bash
def g(x: Int):(Int => Int) = {
  val k = x * 2
  (y: Int) => y + k + 1
}
```

#### 函数的部分应用
