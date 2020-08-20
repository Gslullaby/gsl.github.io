---
title: Task详述
date: 2020-08-16 13:48:29
tags:
- gradle
keywords:
- gradle
category:
- Gradle
description: Task生命周期概述，Task的创建、配置及执行，Task依赖与执行顺序，简单了解自定义任务

---
## Task的生命周期
首先每个task都归属于某个project。编写build脚本（即build.gradle文件）时，可以在脚本文件中定义task。之后运行构建，在构建的配置阶段，会执行project的build.gradle脚本，此时便会依照task创建语句创建Task对象，并对Task对象进行配置（依赖关系等）。配置阶段结束之后，便会输出一个待执行的task图（DAG）。然后构建进入执行阶段，即按照task图中task的依赖关系，执行每个task，并且每个task仅执行一遍。因此有以下结论
1. task的创建与配置发生在构建的配置阶段
2. task的运行发生在构建的运行阶段

## Task详解
#### **Task的创建**
创建task很简单，只需在build.gradle文件中使用task相关api即可。创建task的方式有以下几种
1. 调用Project对象的task方法并传入string类型的参数作为task的name
   > build.gradle
   >```groovy
      task('hello') {
         doLast {
            println "hello"
         }
      }

      task('copy', type: Copy){
         from(file('srcDir'))
         into(buildDir)
      }
   ```
2. 使用Project对象中的tasks（TaskContainer对象）
   > build.gradle
   >```groovy
      tasks.create('hello') {
         doLast {
            println "hello"
         }
      }

      tasks.create('copy', Copy) {
         from(file('srcDir'))
         into(buildDir)
      }
   ```

3. 使用特定的DSL语句，与第一种方式很像
   > build.gradle
   >```groovy
      task(hello) {
         doLast {
            println "hello"
         }
      }

      task(copy, type: Copy) {
         from(file('srcDir'))
         into(buildDir)
      }
   ```

以上三种方式是等效的，本质上无区别。值得注意的是接受多个参数的那个方法，以第一种方式为例，完整的方法声明是   
`Task task(Map<String, ?> args, String name, Closure configureClosure);`   
其中第一个参数`Map<String, ?> args`为任务的创建选项，用以控制如何创建该task。可用的选项有：
{% asset_img task_creation_options.jpg 任务创建的可选项 %}

#### **Task的配置**
再来看下创建Task的完整的方法声明   
`Task task(Map<String, ?> args, String name, Closure configureClosure);`   
第三个参数是一个闭包，参数名为configureClosure，从名字就能看出该闭包用于配置Task。Project中绝大多数创建task的方法都接受一个闭包或Action用于配置Task。
在闭包中可以调用`doLast()`、`doFirst()`方法向Task中添加Action。还是以copy为例来看下    
```groovy
task(copy, type: Copy) {
   doFirst {
      println "doFirst"
   }
   from(file('srcDir'))
   into(buildDir)
   println "configure task copy"
   doLast {
      println "doLast"
   }
}
```

执行`gradlew copy`命令的结果时
```
> Configure project :
before evaluate "app" project
configure task copy
after evaluate "app" project

> Task :copy
doFirst
doLast

```

上例中在构建的配置阶段，创建了一个名为copy的Copy类型的Task，同时用最后传入的闭包对copy task进行了配置。配置即为Task添加Action的过程。上例中Task的配置过程
1. 调用`doFirst {}`方法，添加Action，该Action会输出"doFirst"
2. 调用`from()`方法，为copy方法指定复制的源文件
3. 调用`into()`方法，指定copy方法的目的路径
4. 打印"configure task copy"
5. 调用`doLast {}`方法，添加Action，该Action输出"doLast"

配置的过程可以理解为初始化Task的过程。上例中比较具有迷惑性的地方在于，闭包中调用from以及into方法，从书写来看，像是在配置阶段就进行了文件的复制，但实际情况是，这两个方法仅指定了待复制文件的路径，以及文件复制的目的路径。而真正的文件复制是在任务执行的时候才进行的，发生在doFirst与doLast之间

除了通过创建任务时传入闭包配置task之外，也可以将task的定义与配置分开进行，先定义task，然后在其他时机获取task然后直接调用相应方法进行配置。如下面这种方式
```groovy
task copy(type:Copy)

------------------------------------------

Copy myCopy = tasks.getByName("myCopy")
myCopy.from 'resources'
myCopy.into 'target'
myCopy.include('**/*.txt', '**/*.xml', '**/*.properties')

```

#### Task的执行
执行某个task，如copy task，直接执行`gradlew copy`命令即可。当然如果某任务的执行依赖与其他任务，则执行该任务也会导致前置任务的执行，关于任务依赖的详解说明，在下文介绍。Task的执行即为Task中每个Action的执行。

## Task之间的依赖、顺序

#### **定义Task的依赖**
使用task的`dependsOn()`方法以添加依赖    
有以下几种方式
1. 在创建task的时候通过creation options传入依赖的task，下例展示了跨project的依赖
   > build.gradle
   > ```groovy
   build.gradle

   project('projectA') {
      task('taskX', dependsOn : ':projectB:taskY') {
         doLast {
            println 'taskX'
         }
      }
   }

   project('projectB') {
      task('taskY') {
         doLast {
            println 'taskY'
         }
      }
   }
   ```

2. 直接调用task的`dependsOn()`方法进行依赖。如下：
   > projectA:build.gradle
   > ```groovy

   task('taskX') {
      doLast {
         println 'taskX'
      }
   }
   ```
   > projectB:build.gradle
   > ```groovy
   task('taskY') {
      doLast {
         println 'taskY'
      }
   }

   taskY.dependsOn(':projectA:taskX')
   ```
   
   > 执行`gradle -q taskY`命令的输出
   > ```groovy
   taskX
   taskY
   ```

从第二个例子的运行结果可看出，执行某个任务时，会导致其前置任务的执行

#### **定义Task的顺序**
除了定义任务之间依赖关系，还可以定义任务的顺序。使用Task的`mustRunAfter()`和`shouldRunAfter()`方法即可定义两个任务的顺序。

任务顺序与依赖的不同之处：    
任务间的依赖是一种强关系，执行某任务，则其前置任务必须先执行。但是任务间的顺序是一种弱关系，仅定义某任务必须在另一个任务之后，如`taskA.mustRunAfter taskB`，在执行taskA时taskB不会被执行，只有两个任务都被执行时，才会保证顺序

`mustRunAfter()`与`shouldRunAfter()`的区别：    
则在多个任务执行时，如果用`mustRunAfter()`定义了它们之间的顺序，则执行顺序必须如此，否则会执行失败，抛出异常。而`shouldRunAfter()`不会强制保证顺序，某些情况下顺序会被忽略，下面例子很直观：
```groovy
task taskX {
    doLast {
        println 'taskX'
    }
}
task taskY {
    doLast {
        println 'taskY'
    }
}
task taskZ {
    doLast {
        println 'taskZ'
    }
}
taskX.dependsOn taskY
taskY.dependsOn taskZ
```
然后分别使用`mustRunAfter()`以及`shouldRunAfter()`定义顺序
```groovy
taskZ.shouldRunAfter taskX
```
执行`gradlew -q taskX`命令，输出为：
```
taskZ
taskY
taskX
```

可以看到`shouldRunAfter()`定义的任务顺序被忽略了，下面再来看`mustRunAfter()`
```groovy
taskZ.mustRunAfter taskX
```
执行`gradlew -q taskX`命令，输出为：
```groovy
FAILURE: Build failed with an exception.

* What went wrong:
Circular dependency between the following tasks:
:libA:taskX
\--- :libA:taskY
     \--- :libA:taskZ
          \--- :libA:taskX (*)
```
提示三个任务出现了循环依赖，执行失败

`shouldRunAfter()`会被忽略的两种情况
1. 任务依赖与顺序冲突，则按照依赖执行任务，忽略顺序
2. 启用了并行执行，并且该任务所有依赖的任务都已执行，则该任务会忽略顺序而执行

## 自定义任务
自定义任务线简单了解下，Task是一个接口，Gradle中提供了一些已有的实现，如`DefaultTask`，`Copy`，`Delete`等。在创建任务是可以通过creation options传入type指定要创建的任务类型，如果没有则默认为`DefaultTask`类型。

自定义任务需要集成自Gradle中已有的任务类型，最常见的时继承自`DefaultTask`。   

自定义任务另一个关键是，使用特定注解，如`@Input`、`@InputFiles`、`@OutputDirectory`等用于指定输入输出。注解有很多。这里不展开写了，后续如果有必要时再写。