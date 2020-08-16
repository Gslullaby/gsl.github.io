---
title: 构建的基础知识
date: 2020-08-08 23:54:39
tags: 
- gradle
keywords:
- gradle
category:
- Gradle
description: 本文主要介绍有关构建的一些基本概念，如构建脚本的结构，Project、Task，以及它们之间的关系等等。
---

## 一些基础概念
在Gradle生命周期中，我们曾提及构建脚本其实是构建配置脚本。脚本执行，其实就是对一个特定类型对象的配置。例如，build.gradle脚本执行，就是对`Project`对象的配置。所配置的对象称之为构建脚本的代理对象。下图展示了脚本与代理对象的对应关系

{% asset_img script_delegate.jpg 脚本与代理对象对应关系 %}

在构建脚本中可以访问到代理对象的属性以及方法

每个构建脚本的代理对象都实现了`Script`接口。该接口中定义的属性和方法在构建脚本中均可访问

#### **构建脚本的结构**
构建脚本由0到多个语句及脚本代码块组成。语句可以包含方法调用，属性赋值，成员变量的定义等。脚本块是对一个接收一个闭包作为参数的方法调用。闭包可以看作是配置闭包，因为闭包的调用即为对相应代理对象的配置。

下面是一些常用的顶层脚本块

* `allprojects {}` 
     > 配置当前project及其所有子project

* `buildscript {}`
     > 为当前project配置构建脚本的classpath

* `dependencies {}`
     > 为当前project配置依赖（第三方lib等）

* `repositories {}`
     > 为当前project配置资源库（下载依赖库的仓库，如jcenter等）

* `sourceSets {}`
     > 为当前project配置源码及资源等（如Android中的java代码，资源文件等）
* `subprojects {}`
     > 配置当前project的所有子project

#### **几个核心类型**
* `Project`
     > 该接口是从构建脚本中与Gradle交互的核心API。

* `Task`
     > Task代表一个原子操作，是构建的最小单位
* `Gradle`
     > 表示对Gradle的调用
* `Settings`
     > 声明了实例化和配置要参与构建的Project实例所需的配置

#### **Prject与Task的关系**
Gradle中的一切都建立在两个基本概念之上：project和task

每个Gradle构建都由一个或多个Project组成。Project取决于要构建什么类型的项目。例如，project可能是JAR或web应用。也可能是其他项目输出的JAR打包成的ZIP等。项目也可以不是待构建的事物，也可以是待做任务，如部署应用至测试或生产环境。

每个Project由一个或多个Task组成。任务即构建执行的原子操作。如编译指定的类，打一个JAR包，生成Javadoc或发布库至仓库等

下面是一个单项目的示例，该但项目中定义了一些简单的任务

## Project
该接口中包含了与Gradle交互的主要API。在Project对象中，可以访问Gradle的所有特性

#### **Lifecycle**
`build.gradle`文件与`Project`对象是一对一的关系。在构建初始化阶段，Gradle会为每个参与构建的项目创建一个Project对象。创建步骤如下：   
* 首先为构建创建一个`Settings`对象
* 分析`settings.gradle`脚本，并根据该脚本配置`Settings`对象
* 使用配置好的`Settings`对象，创建参与构建的项目对应的`Project`对象
* 最后，通过执行项目的`build.gradle`脚本来配置与之对应的`Project`对象。Project按广度优先顺序进行配置，即父Project先于子Project配置。不过可以通过调用`Project.evaluationDependsOnChildren()`或`Project.evaluationDependsOn(java.lang.String)`改变初始化顺序

#### **Tasks**
`Project`本质上是`Task`对象的集合。通过调用`TaskContainer`的`create()`方法为`Project`添加任务，如`TaskContainer.create(java.lang.String)`。`TaskContainer`中提供了一些方法，用以查询已定义的`Task`，如`TaskCollection.getByName(java.lang.String)`

#### **Dependencies**
project为了完成工作，通常需要某些依赖。同样的，project通常也会输出结果供其他project使用。依赖按配置分组，从仓库中上传下载。可以使用`Project.getConfigurations()`方法返回的`ConfigurationContainer`对象来管理配置。使用`Project.getDependencies()`方法返回的`DependencyHandler`对象管理依赖。使用`Project.getRepositories()`方法返回的`RepositoryHandler`对象管理仓库

#### **Plugins**
插件可用于构建的模块化以及重用project配置。使用`PluginAware.apply(java.util.Map)`方法或使用`PluginDependenciesSpec`来应用插件

#### **Dynamic Project Properties**
Gradle执行`build.gradle`脚本来配置与之对应的`Project`对象。脚本中使用的任何属性及方法都是对`Project`代理对象的访问。也就是说，可以直接在脚本中使用`Project`接口的任一方法和属性。

例如：
```groovy
defaultTasks('some-task')  // Delegates to Project.defaultTasks()
reportsDir = file('reports') // Delegates to Project.file() and the Java Plugin
```

也可以通过`project`属性访问`Project`实例。某些情况下，以这种方式访问会使脚本更加清晰。如，可以使用`project.name`访问project的name，而不是使用直接用`name`属性

project有5个属性域，用以搜索属性。在build文件中可以通过属性名或调用`Project.property(java.lang.string)`方法访问属性。下面是5个属性域：
* `Project`对象本身。该域中包含了`Project`接口实现类中顶一个任何属性（拥有get&set方法）。例如，`Project.getRootProject()`方法用以访问`rootProject`属性。属性的可读可写性取决于get/set方法的声明
* project的*额外(extra)*属性。每个project都维护了一个额外属性的map，属性名（任意）->属性值。额外属性均可读可写
* 由插件引入的扩展。每个扩展相当于一个只读的属性，扩展名即属性名
* 由插件引入的convention属性。插件可以通过Porject中的`Convention`对象向其中加入方法和属性。该域中属性的可读可写性取决于`Convention`对象
* project中的task。可以将task的name看作属性名对其进行访问。该域中的属性是只读的。例如，名为`compile`的task可以作为`compile`属性进行访问
* extra属性和convention属性是从项目的父级继承的，直到根项目。 该域中的的属性是只读的。

当访问一个属性时，project按以上顺序从域中进行查找，如果未找到，则抛出异常

#### **Extra Properties**
所有的extra属性都必须通过"ext"命名空间来定义。额外属性定义后，即可在所属对象中直接访问（可以是Project、Task或者sub-project），可以读取也可更改。下面是一个例子

```groovy
project.ext.prop1 = "foo"
task doStuff {
    ext.prop2 = "bar"
}
subprojects { ext.${prop3} = false }
```

通过"ext"读取extra属性
```groovy
ext.isSnapshot = version.endsWith("-SNAPSHOT")
if (isSnapshot) {
    // do snapshot stuff
}
```

#### **Dynamic Methods**
project同样由5个方法域，用以查找方法
* `Project`对象本身
* build文件。build文件中可以定义方法，project从其中查找特定的方法。
* 由插件引入的扩展。扩展可以看作一个接受一个闭包或`Action`作为参数的方法
* 由插件引入的convention方法。插件可以通过project的`Convention`对象向其中添加属性和方法
* project中的task。每个Task都会拥有一个与Task同名的方法，该方法接受一个闭包或`Action`作为参数,并且使用传入的闭包关联到Task的`Task.configure(groovy.lang.Closure)`方法。例如，project有一个名为`compile`的方法，则一个`void compile(Closure ConfigureClosure)`方法会添加至project中
* 继承自父Project的方法
* project中值为闭包的属性。闭包可以看作一个方法，直接传入参数调用

## Task
前文中已经给出了Task的定义，以及Task与Project的关系，这里不再赘述。    

下面介绍下Task的属性、创建、执行。以及Task之间的依赖关系的声明等

#### **Task的组成**
Task由一系列的`Action`组成。Task的执行，即为其中`Action`的执行（调用`Action.execute(T)`）。可以通过调用`Task.doFirst(org.gradle.api.Action)`或`Task.doLast(org.gradle.api.Action)`方法向Task中添加`Action`，Task中还有两个接受`groovy.lang.Closure`闭包作为参数的同名方法，同样可以向Task中添加Action。

Task中的两个特殊异常，`StopActionException`和`StopExecutionException`。其中`StopActionException`会导致当前`Action`中止执行，转而执行下一个`Action`。而`StopExecutionException`会中止当前`Task`，转而执行下一个`Task`。可以通过抛出这两个异常来跳过某些执行错误的`Action`或`Task`

#### **Task之间的依赖关系**
一个Task的执行可能依赖于其他Task，也可能必须在某个Task执行完成后才能开始执行。Gradle会保证任务按照它们之间的依赖关系执行。

下面几组方法可以声明Task间的依赖关系，每组中的两个方法是等效的。
* `Task.dependsOn(java.lang.Object[])` 或 `Task.setDependsOn(java.lang.Iterable)`
* ` Task.mustRunAfter(java.lang.Object[])` 或 ` Task.setMustRunAfter(java.lang.Iterable`
*  `Task.shouldRunAfter(java.lang.Object[])` 或 ` Task.setShouldRunAfter(java.lang.Iterable)`

可以使用以下类型作为参数来指定任务间的依赖关系
* `String`、`CharSequence`或`groovy.lang.GString`类型的任务路径或任务名。当千project的task的相对路径。这种方式可以引用其他project中的task
* `Task`类型
* `TaskDependency`类型
* `TaskReference`类型
* `Buildable`类型
* `RegularFileProperty`或`DirectoryPerty`类型
* `Provider<T>`类型。其中泛型为这里列出的任意类型之一
* `Iterable`、`Collection`、`Map`或array类型。其中元素类型为这里列出的任意类型之一。其中元素最终可递归的转换为task
* `Callable`类型。`call()`方法返回类型为这里列出的任意类型之一，并且最终可转换为task，返回值`null`会被视为空集合
* Groovy闭包`Closure`或Kotlin函数。闭包或函数返回类型为这里列出的任意类型之一，并且最终可转换为task，返回值`null`会被视为空集合
* 其他类型会导致error

#### **Task的动态属性**
`Task`有四个属性域。在构建文件中可以通过属性名直接访问属性，或通过调用`Task.property(java.lang.String)`方法访问属性。可以通过调用`Task.setProperty(java.lang.String, java.lang.Object)`方法改变属性值
* `Task`对象本身。`Task`实现类中拥有get和set方法的字段即为属性。属性的读写性取决于get、set方法的声明
* 通过插件添加至任务的扩展。每个扩展即为一个同名的属性
* 通过插件引入的*convention*属性。插件可以通过`Convention`对象向Task中添加属性及方法。属性的读写性取决于`Convention`对象
* 与project一样，task也可以添加额外属性。每个task都维护了一个额外属性的map，属性名（任意）->属性值。额外属性均可读可写

#### **Task并行执行**
默认情况下，多个task是不会并行执行，除非一个task正在等待异步操作完成，并且有其他task在等待执行。在构建初始化时，可以通过`--parallel`标志启用并行执行，不同project的task可以并行执行


本篇只是简单介绍了包括Project、Task在内的基本概念。关于Project、Task以及实战操作会在后续文章中详细介绍