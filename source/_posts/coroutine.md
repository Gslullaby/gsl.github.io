---
title: 什么是协程？
date: 2020-05-05 16:33:57
tags:
- coroutine
keywords:
- coroutine
- kotlin
- 协程
description:
categories:
- Coroutine
---

Kotlin中的协程虽然会用，但也一直停留在表层Api。没有仔细探究过其思想以及实现，使用起来也会比较虚，心里没底。所以很有必要对其一探究竟，揭一揭它的面纱

## 协程的定义

协程: Coroutine    
协程的出现，可以追溯到1958年，它由马尔文·康威提出用于构建汇编程序，而且它的出现要早于线程。

定义[Wiki](https://en.wikipedia.org/wiki/Coroutine):  :
> __Coroutines__ are computer program components that generalize subroutines for non-preemptive multitasking, by allowing execution to be suspended and resumed. Coroutines are well-suited for implementing familiar program components such as cooperative tasks, exceptions, event loops, iterators, infinite lists and pipes    

> 译: 协程时计算机程序的一类组件，通过允许挂起和恢复执行概括了非抢占式（即协作式）多任务的子程序。协程非常适合用于实现一些常见的程序组件，如协作式多任务，异常，事件循环，迭代器，无限列表以及管道

Wiki中对于协程的定义还是挺难理解的，其中的关键词
* 子程序（subroutine）  
* 挂起（suspend）和恢复（resume）
* 抢占式（preemptive）和协作式（cooperative or non-preemptive）  

如果不先理解这几个关键词的话，是很难搞清楚协程的

#### 首先是子程序
《The Art of Computer Programming》(计算机程序设计艺术)一书中对subroutine有详细的解释，以下是原文
> When a certain task is to be performed at severial different places in a program, it is usually undesirable to repeat the coding in each place. To avoid this situation, the coding (called a subroutine) can be put into one place only, and a few extra instructions can be added and to restart the outer program properly after the subroutine is finished    

> 译: 当一项确定的任务再程序的不同位置多次执行时，通常重复的代码不是我们想看到的。为了避免这种情况，这段代码（叫做一个子程序）可以被单独放在一个位置，通过少量额外的指令在每个需要执行该此程序的位置添加并执行子程序，并在子程序执行完成后，自动的重启外部程序

这段对于子程序的描述还是很容易理解的，从高级语言的角度来看，子程序就相当于一个函数或方法，对子程序的引用即为对函数或方法的调用。

子程序的用途书中也有描述，这里简单提一下：子程序的主要优势是避免重复代码，节省程序空间。也间接的节省了时间，如：加载程序需要的时间减少等。子程序也会让构思大型程序变得更容易，debug也会更容易。封装出来的子程序也可以作为开放库对外暴露给其他开发者使用。

理解了子程序，那么子程序跟协程又有什么关系呢？或者说两者有什么区别呢？同样引用原文给出答案
> Subroutines are special case of more general program components, called coroutines. In contrast to the unsymmetric relationship between a main routine and a subroutine, there is complete symmetry between coroutines, which on call each other.    
> 
> We may consider the main program and subroutines as a team of programs, each member of the team having a certain job to do. The main program, in the course of doing its job, will activate the subprogram; the subprogram will perform its own function and then activate the main program. We may stretch our imagination to believe that, from the subroutine's point of view, when it exits it is calling the main routine; the main routine continues to performs its duty, then 'exits' to the subroutine. the subroutine acts, then call the main routine again.

译:     
> 子程序是通用程序组件（即协程）的一种特例。与主程序与子程序间非对等关系（调用者与被调用者）对比，协程间是完全对等的，他们可以互相调用 

> 我们可以将主程序和子程序看作是程序组，程序组中的每个成员都要完成特定的任务。主程序在完成其任务的过程中，会激活子程序，然后子程序会执行完成它自己的任务，随后再激活主程序让主程序继续执行。拓展下我们的想象力，从子程序的视角出发，在子程序出口位置调用主程序，主程序继续执行其任务，接着主程序'退出'并在出口位置调用子程序，子程序执行，而后再调用主程序

上面第二段的意思是，子程序只有一个入口和出口，而协程可以有多个入口和出口。即子程序被主程序调用后只能从头开始执行（入口）直到执行完成然后退出（出口），而后主程序继续执行。而协程在被调用后，在中途可能会有一个或多个出口（出口点之后仍然有代码待执行），当执行中途遇到出口时，当前协程交出执行权让其他协程执行，当下次再调用该协程时，会从上次的出口点位置继续开始执行后续的代码（上次的出口点即为下次执行的入口点）。这也就解释了第一段中子程序和主程序关系与协程间关系的差别了

> Wiki中子程序与协程的比较
> * 子例程可以调用其他子例程，调用者等待被调用者结束后继续执行，故而子例程的生命期遵循后进先出，即最后一个被调用的子例程最先结束返回。协程的生命期完全由对它们的使用需要来决定
> * 子例程的起始处是惟一的入口点，每当子例程被调用时，执行都从被调用子例程的起始处开始。协程可以有多个入口点，协程的起始处是第一个入口点，每个yield返回出口点都是再次被调用执行时的入口点。
> * 子例程只在结束时一次性的返回全部结果值。协程可以在yield时不调用其他协程，而是每次返回一部分的结果值，这种协程常称为生成器或迭代器。
> * 现代的指令集架构通常提供对调用栈的指令支持，便于实现可递归调用的子例程。在以Scheme为代表的提供续体的语言环境下，恰好可用此控制状态抽象表示来实现协程。

#### 挂起和恢复
其实在子程序那一部分中已经对挂起和恢复做出了解释。协程拥有多个出口和入口，所谓挂起就是在执行途中遇到出口，当前协程让出执行权，跳转到其他协程继续执行，而恢复就是当再次调用该协程时从上次出口点继续执行后续任务。出口对应挂起，入口对应恢复

#### 抢占式和协作式
首先，无论是抢占式还是协作式，都意味着当前Task都不再继续拥有执行权，而两者区别是，抢占式多任务中，当前任务被动的系统剥夺执行权，协作式多任务中，当前任务在不需要系统资源时主动让出执行权    

抢占式多任务涉及到了中断机制的使用，中断机制可以挂起当前进程并唤起调度器以决定接下来应该调度执行哪个进程。因此，所有进程在一段时间内都会获得一定数量的CPU时间

抢占式多任务依赖于 CPU 的硬件支持。 因为调度器需要“剥夺”进程的执行权，就意味着调度器需要运行在比普通进程高的权限上，否则任何“流氓（rogue）”进程都可以去剥夺其他进程了。只有 CPU 支持了执行权限后，抢占式调度才成为可能

不同于抢占式由系统调度进程切换，协作式进程在空闲（idel）或阻塞（block）时会主动的让出执行权（yield）


## 协程能解决什么问题呢

如果从早先协程的设计理念来看，协程是设计用来处理单内核下多任务并发的。协程最本质的特性是可挂起和恢复的特性，拥有多入口多出口。
相较于子程序的从头运行至尾，协程有更丰富的过程（有一个或多个运行阶段）,多个任务可以交替执行，使得多任务协作提供成为可能，即协程拥有并发的能力。

而对于现代的高级语言来说，协程更像是任务，运行在线程之上，在语言级别进行调度。许多语言的也都是多线程 + 协程的的实现，以达到像写同步代码那样处理异步问题。通常异步需要多线程来实现，但是弊端就是多线程调度时，切换上下文较为耗时，而如果通过多线程 + 协程方式来处理异步的话，相当于将系统级的线程调度替换为语言级别的协程调度，一定程度上减少了多线程调度切换上下文的开销。另一方面，多线程方式实现异步最可能面临异步回调，而最熟悉的应该是JS中的回调地狱问题了，而有了协程之后，就可以完美的消除该问题。

## 简单总结
协程从本质上讲依然是一种程序，运行与线程之上。名称以及它表现出来的特征与线程有相似之处，但与线程不同的是，他的调度并不需要系统参与，而是直接在语言级别完成。这也是许多人称协程为用户级线程的原因

