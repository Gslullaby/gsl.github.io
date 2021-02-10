---
title: 依赖配置的相关知识
date: 2020-09-03 21:22:14
tags:
- gradle
keywords:
- gradle
category:
- Gradle
description: 要了解依赖，首先必须得了解依赖配置。因此这篇blog介绍了依赖配置的基本概念、用法等
---
## 依赖配置是什么？
Gradle项目中声明的依赖都有其适用范围。如某些依赖编译代码时需要用到，而另一些依赖仅用于运行时。Gradle中使用依赖配置（Configuration）来表示这些依赖的作用范围（作用域）。

许多Gradle插件会为项目添加一些预先定义好的配置（如`android-application`插件会添加`release`和`debug`配置等，Java插件添加了一些来表示源代码编译、执行测试等所需的各种类路径的插件）。下图使用Java插件作为例子
{% asset_img dependency-management-configurations.png 声明用于特定目的的依赖 %}

**配置的继承与组合**
配置可以继承其他配置，从而形成一个父子层级关系。子配置会继承其父配置声明的所有依赖。

Gradle核心插件（如Java插件）中大量使用了配置继承。如`testImplementation`继承了`implementation`。这种配置层级关系有很实用的目的。编译测试需要在编写测试代码所需依赖之上增加对待测试源码的依赖。如果项目的生产环境的源码引用了Guava，则其在使用JUnit编写并执行测试代码时也需要引用Guava。下图展示了Java插件中的配置依赖所示

{% asset_img dependency-management-configuration-inheritance.png Java插件中的配置依赖 %}

在背后，通过调用`Configuration.extendsFrom(org.gradle.api.artifacts.Configuration[])`方法`testImplementation`与`implementation`配置形成了一种父子层级关系。configuration可以继承其他任何configuration，无论它是定义在构建脚本中还是插件中

以下用冒烟测试作为例子。假如我们要编写一套冒烟测试用例。每个测试用例都会调用一个Http请求用以验证web后端服务。由于底层的测试框架已经默认使用了JUnit。所以我们一个定义一个名为`smokeTest`的配置，继承自`testImplementation`以重用已有的测试框架依赖

代码如下：
> build.gradle
```groovy
configurations {
    smokeTest.extendsFrom testImplementation
}

dependencies {
    testImplementation 'junit:junit:4.13'
    smokeTest 'org.apache.httpcomponents:httpclient:4.5.5'
}
```

## 可解析和可消费的配置
配置是Gradle依赖解析中至关重要的部分。在依赖的解析的上下文中，区分生产者与消费者的是很重要的。如此说来，配置至少有三条规则：
1. 声明依赖
2. 作为消费者，把一系列依赖解析为文件
3. 作为生产者，对外暴漏artifact和依赖项以供其他项目使用（可消费的配置项通常代表生产者给消费者提供的变体）

例如，为了表达应用`app`依赖`lib`，则至少需要一个配置

> build.gradle
```groovy
configurations {
    //声明了一个名为"someConfiguration"的配置
    someConfiguration
}
dependencies {
    // 向`someConfiguration`配置添加一个项目依赖
    someConfiguration project(":lib")
}
```

配置可以通过扩展其他配置以继承其所有依赖。上例中的代码并没有告诉我们任何关于该配置消费者的信息。更为重要的是，他没有告诉我们该如何使用此配置项。假如`lib`为一个Java库：他可能会对外暴露不同的东西，如它的API、实现或者测试工具。根据要执行的task的不同（根据lib的API进行编译，执行该应用程序，编译测试等），可能要对`app`的依赖进行的不同的解析。为了解决这种需求，我们通常可以发现一些伴生配置，旨在明确配置的用法：
> build.gradle
```groovy
configurations {
    // 声明用于解析应用程序的编译类路径的配置
    compileClasspath.extendsFrom(someConfiguration)

    // 声明用于解析应用程序的运行时类路径的配置
    runtimeClasspath.extendsFrom(someConfiguration)
}
```
上面代码中，展示了三种不同的配置，这三种配置分别担任了不同的角色：   
* `someConfiguration`声明了应用程序的依赖。它就像一个篮子用来装该应用的依赖
* `compileClasspath`和`runtimeClasspath`是要解析的配置：分别表示应用的编译类路径以及运行时类路径

这种区别由配置中的canBeResolved标志表示。可以解析的配置即为可以计算依赖图的配置，因为它包含进行解析所需的所有信息。也就是说，我们会出计算一个依赖图，解析依赖图中的组件，最终输出artifact。如果某个配置的`canBeResolved`设置为`false`，则意味着该配置不会被解析，而仅仅用与声明依赖。原因是根据使用情况（编译类路径、运行时类路径），它可以解析为不同的图。尝试解析一个`canBeResolved`设置为`false`的配置是不对的。某种程度上类似于不该被实例化的抽象类(canBeResolved=false)，以及扩展抽象类(canBeResolved=true)的实现类。一个可解析的配置至少会扩展一个不可解析的配置。

另一方面，对于库项目来说（生产者），我们也使用配置来表示可以消费的内容。例如，`lib`可能对外暴露了API或运行时（即运行时用到该lib），我们可能会把`artifact`添加到其一或者两者之中。通常在按照`lib`进行编译时，仅需要其API，而无需依赖其运行时。因此`lib`项目会暴露一个`apiElements`配置，用于给消费者寻找其API。像这样一种配置即为可消费但不可解析的配置。该特性通过配置的`canBeResumed`标志表示
> build.gradle
```groovy
configurations {
    // 供需要使用其API的消费者使用的配置
    exposedApi {
        // 表示该配置不可被解析
        canBeResolved = false
        // 表示该配置可以被消费
        canBeConsumed = true
    }
    // 供需要使用其代码的消费者使用的配置
    exposedRuntime {
        canBeResolved = false
        canBeConsumed = true
    }
}
```

总的来说，配置扮演的角色是由`canBeResolved`和`canBeConsumed`标志决定的，如下图：
{% asset_img configuration_roles.jpg configuration roles %}

为了向后的兼容性，这两个标志为的默认值为`true`。但是对于插件作者来说，应该根据需要设置值，否则可能会出现解析错误。

## 自定义配置
自定义配置可用于分离用于不同目的的依赖，以形成不同的依赖域。

假设你想声明对Jasper Ant任务的依赖关系，以便预编译JSP文件，但这些文件不应该最终出现在用于编译源代码的类路径中。通过自定义配置可以轻松实现此类需求，如下面例子
> build.gradle
```groovy
configurations {
    jasper
}

repositories {
    mavenCentral()
}

dependencies {
    jasper 'org.apache.tomcat.embed:tomcat-embed-jasper:9.0.2'
}

task preCompileJsps {
    doLast {
        ant.taskdef(classname: 'org.apache.jasper.JspC',
                    name: 'jasper',
                    classpath: configurations.jasper.asPath)
        ant.jasper(validateXml: false,
                   uriroot: file('src/main/webapp'),
                   outputDir: file("$buildDir/compiled-jsps"))
    }
}
```

项目的配置由`configurations`对象管理。



关于依赖配置就先到这里吧，下一篇介绍不同类型的依赖，如文件依赖，module依赖等