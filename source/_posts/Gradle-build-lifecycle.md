---
title: Gradle构建的生命周期
date: 2020-07-20 10:33:30
tags:
- gradle
keywords:
- gradle
description:
---
我们之前说过，Gradle的核心是一种基于依赖的编程语言。按照Gradle的术语来说，这意味着你可以定义一系列任务以及它们之间的依赖关系。Gradle会保证这些任务按照他们的依赖关系顺序执行，并且每个任务只会执行一次。这些任务一起构成了一个有向无环图（DAG）。某些构建工具会在构建任务执行阶段，组建依赖图。但是Gradle是在任务执行之前组建出完整的依赖图的。这正是Gradle的核心所在，使得许多不可能的事情成为了可能。    

由于构建脚本是用于配置依赖图的。因此严格意义上说，构建脚本应该称为构建配置脚本

## 构建阶段
Gradle构建有三个不同的阶段

* 初始换阶段（Initialization）
     
    Gradle支持单/多项目构建。在初始化阶段，Gradle会确定哪些项目要参与到构建中，并且为每个参与构建的项目创建一个Project实例

* 配置阶段（Configuration）   
  
    该阶段对Project实例进行配置。即每个参与构建的项目的构建脚本都会被执行

* 执行阶段（Execution）   
  
    配置阶段中Gradle创建并配置了一系列任务，最终输出一个任务图（DAG），执行阶段中Gradle会先确定一个DAG的待执行子任务集。子任务集是由gradle命令参数传入的任务名以及当前所在目录确定的。确定了子任务集后，Gradle会按顺序执行其中的每一个任务

## Settings.gradle文件
除了构建脚本（build.gradle）文件，Gradle还定义了一个settings脚本文件。settings文件由Gradle通过命名约定确定。该文件的默认名为settings.gradle。在后续章节中会解释Gradle是如何查找settings文件的。    
settings脚本在初始化阶段执行。多项目构建必须定义一个settings.gradle文件，文件路径为多项目的根项目目录。之所以该脚本文件是必需的，是因为该脚本定义了那些任务会参与到构建中。对于单项目构建，可以不定义该脚本文件。该文件中，除了定义参与项目外，可能还需要添加一些依赖库到构建脚本的classpath中。下面先来看一个单项目构建的例子    

*Example 1. 单项目构建* 
> **settings.gradle**

 ```groovy 
 print 'This is executed during the initialization phase.'
 ```
> **build.gradle**

 ```groovy
 println 'This is executed during the configuration phase.'
 
 task configured {
    println 'This is also executed during the configuration phase.'
 }
 
 task test {
    doLast {
        println 'This is executed during the execution phase.'
    }
 }

 task testBoth {
    doFirst {
        println 'This is executed first during the execution phase.'
    }
    doLast {
        println 'This is executed last during the execution phase.'
    }
    println 'This is executed during the configuration phase as well.'
 }
 ```

#### 命令 `gradle test testboth` 的输出
>
```groovy
> gradle test testBoth   
This is executed during the initialization phase.
 
Configure project :    
This is executed during the configuration phase.    
This is also executed during the configuration phase.    
This is executed during the configuration phase as well.

Task :test   
This is executed during the execution phase.
 
Task :testBoth      
This is executed first during the execution phase.   
This is executed last during the execution phase.

BUILD SUCCESSFUL in 0s   
2 actionable tasks: 2 executed    
```

对于build.gradle来说，其中属性的访问以及方法的调用都是委托给project对象完成的。同样的，对于settings.gradle来说是委托给Settings对象完成的。查看API文档中Settings类以了解更多信息

## 多项目构建
所谓多项目构建，即为一次Gradle执行中会构建多个项目的构建。我们必须在settings脚本中声明参与构建的项目。在专门介绍此主题的一章中，有更多关于多项目生成的内容要说(请参见多项目构建)

### **项目位置**
多项目构建总是可以由一个单根树表示。树中的每个元素代表一个项目。每个项目都有一个路径，用于表示该项目在构建树中的位置。大多数情况下，项目的路径即为该项目在文件系统中的真实物理位置。然而，该路径是可配置的。项目树在settings.gradle脚本文件中创建。默认情况下会假定settings.gradle文件目录在根项目目录中。可以在settigns脚本文件中重新定义根项目的目录

### **构建项目树**
在settings文件中，可以使用一系列方法来构建项目树。层次化和扁平化的物理布局都得到了支持。

#### 层次化布局
*Example 2. 层次化布局*
> **settings.gradle**

 ```groovy
 include 'project1', 'project2:child', 'project3:child1'
 ```


`include` 方法以project路径作为参数。假定project路径即为物理文件系统的相对路径。例如，路径'services:api'默认情况下会映射到'services/api'文件夹（相当于项目根目录）。路径中我们仅需要给出项目树的叶子节点即可。例如路径'service:hotels:api'将会创建三个项目：'services'，'service:hotels'以及'service:hotels:api'。更多关于如何使用project路径，请参阅DSL文档中的`Settings.include(java.lang.String[])`方法    

#### 扁平化布局
*Example 3. 扁平化布局*
> **settings.gradle**

 ```groovy
 includeFlat 'project3', 'project4'
 ```

`includeFlat`方法以目录名作为参数。这些目录必须为根项目目录的同级目录。这些目录的位置被视为多项目树中根项目的子项目。

### **修改项目树中的元素**
在设置文件中创建的多项目树由所谓的项目描述符组成。你可以随时在settings文件中修改这些描述符。可以通过下面的方式访问描述符：   
*Example 4.访问项目树的元素*
> **settings.gradle**

 ```groovy
 println rootProject.name
 println project(':projectA').name
 ```

使用project描述符可以修改project的名字、目录以及项目对应的build文件

*Example 5.修改项目树中的元素*
> **settings.gradle**

 ```groovy
 rootProject.name = 'main'
 project(':projectA').projectDir = new File(settingsDir, '../my-project-a')
 project(':projectA').buildFileName = 'projectA.gradle'
 ```

查看API文档中的`ProjectDescriptor`类以获取更多信息

## 初始化
Gradle是如何知道当前进行单项目构建还是多项目构建的呢？如果从一个包含settings文件的目录触发多项目构建，那么一切都很清楚。但是Gradle允许从任何一个参与构建的子项目触发构建。如果从一个不包含settings.gradle文件的子项目中触发构建，则Gradle会按照以下方式查找settings.gradle文件    
* 从master目录下查找，master目录与当前目录在同层级
* 如果没找到，则从其父目录中查找
* 如果还没找到，则进行单项目构建
* 如果找到了一个settings.gradle文件，Gradle首先会检查当前项目是否存在于settings.gradle文件中定义的多项目层级中。如果不是，则将构建作为单项目构建执行。否则，将执行多项目构建

那么此种行为的目的是什么呢？这是因为Gradle需要确定当前所在的project是否是多项目构建的子项目。如果是，则构建当前子项目以及其所依赖的项目，但是Gradle会为整个多项目构建创建构建配置。如果当前项目有settings.gradle文件，则构建会以如下方式执行
* 如果settings.gradle文件未定义多项目层级，则进行单项目构建
* 否则进行多项目构建

对于settings.gradle文件的自动搜索仅适用于物理上层次化或扁平化布局的多项目构建。对于扁平化布局，则必须遵从上述的（"master"）命名约定。Gradle支持多项目构建的任意物理布局，但是对于这种任意布局，需要从settings.gradle所属目录执行构建。关于如何从根目录执行部分构建，请参阅按任务的绝对路径运行任务

Gradle为每个参与构建的项目创建一个Project对象。对于多项目构建，即为Settings对象中指定的Project以及一个root Project。Project对象的name即为project顶层目录的name，每个Project都有一个父Project（root Project除外）。每个Project都可能拥有子Project

## 单项目构建的配置与执行
对于单项目构建，其初始化阶段之后的工作流是十分简单的。构建脚本按照初始化阶段创建的Project对象执行。然后Gradle按照命令行传入的参数查找需要执行的任务。如果任务存在，则会按照所传入的顺序分别独立执行。多项目构建的配置与执行会在*Authoring Multi-Project Builds*中讨论


## 响应构建脚本中的生命周期
构建过程中，构建脚本会收到构建的生命周期通知。剋以以两种方式接受通知：
1. 实现一个特殊的监听器接口
2. 提供一个通知触发时执行的闭包

下面的例子中使用了闭包的方式。关于如何使用监听器接口，请参阅API文档

### **Project的评估**
在Project评估的前后，我们均会立马收到一个通知。在构建脚本中所有的定义均已应用后，执行可以使用此特性做一些额外操作，如执行附加配置、添加自定义日志或性能分析等

下面例子中，为每个拥有hasTests属性且值为true的Project添加了一个test任务

*Example 6.为拥有特定属性的project添加一个test任务*
> **build.gradle**

 ```groovy
 allprojects {
     afterEvaluate {
         if (project.hasTests) {
             println "Adding test task to $project"
             project.task('test') {
                 doLast {
                     println "Running tests for $project"
                 }
             }
         }
     }
 }
 ```
> 
> **projectA.gradle**

```groovy
 hasTest = true
```

#### 命令`gradle -q test`的输出
>
```groovy
> gradle -q test    
Adding test task to project ':projectA'    
Running tests for project ':projectA'
```

本例调用了Project.afterEvalute()方法并传入一个闭包，该闭包在相应Project评估完成后执行

当然在任一Project评估完成后接受通知也是可以的。下面的例子增加了一些Project评估的日志。需要注意的是无论Project评估成功与否，均会收到afterProject通知

*Example 7.在每个项目评估完成后都会收到的通知*
> **build.gradle**

 ```groovy
 gradle.afterProject {
    if (project.state.failure) {
        println "Evaluation of $project FAILED"
    } else {
        println "Evaluation of $project succeeded"
    }
 }
 ```

#### 命令`gradle -q test`的输出
```groovy
> gradle -q test
Evaluation of root project 'buildProjectEvaluateEvents' succeeded    
Evaluation of project ':projectA' succeeded    
Evaluation of project ':projectB' FAILED

FAILURE: Build failed with an exception.

* Where:
Build file '/home/user/gradle/samples/projectB.gradle' line: 1

* What went wrong:
A problem occurred evaluating project ':projectB'.
broken

* Try:
Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.

* Get more help at https://help.gradle.org

 BUILD FAILED in 0s
```

也可以通过给Gradle添加一个ProjectEvaluationListener已接受通知事件

### **任务的创建**
在一个task被添加到project后，也会立马收到一个通知。在task可用之前，可以为其设置一些默认值或其他操作

下面例子在每个task创建之后，为其设置了`srcDir`属性

*Example 8.为所有任务设置特性属性*
> **build.gradle**

 ```groovy
 tasks.whenTaskAdded { task ->
    task.ext.srcDir = 'src/main/java'
 }

 task a

 println "source dir is $a.srcDir"
 ```

#### 命令`gradle -q a`的输出
```groovy
 > gradle -q a   
 source dir is src/main/java
```

也可以通过向TaskContainer添加一个Action来接受这些事件

### **任务图的组建**
任务图组建完成后，也会立即收到一个通知   

通过向TaskExecutionGraph添加一个TaskExecutionGraphListener以接受这些事件

### **任务的执行**
任一任务执行的前后我们都会立即收到通知

下面例子在每个任务执行的前后输出了日志。注意无论任务执行成功与否均会收到`afterTask`通知

*Example 9.记录每个任务执行的开始与结束*
> **build.gradle**

 ```groovy
 task ok
 
 task broken(dependsOn: ok) {
     doLast {
         throw new RuntimeException('broken')
     }
 }
 
 gradle.taskGraph.beforeTask { Task task ->
    println "executing $task ..."
 }
 
 gradle.taskGraph.afterTask { Task task, TaskState state ->
     if (state.failure) {
         println "FAILED"
     }
     else {
         println "done"
     }
 }
 ```

#### 命令`gradle -q broken`的输出
```groovy
 > gradle -q broken    
 executing task ':ok' ...    
 done   
 executing task ':broken' ...   
 FAILED

 FAILURE: Build failed with an exception.
 
 * Where:   
 Build file '/home/user/gradle/samples/build.gradle' line: 5
 
 * What went wrong:   
 Execution failed for task ':broken'.   
 > broken   

 * Try:    
 Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output. Run with --scan to get full insights.
 
 * Get more help at https://help.gradle.org
 
 BUILD FAILED in 0s
```
也可以通过向TaskExecutionGraph添加TaskExecutionListener来接受这些事件