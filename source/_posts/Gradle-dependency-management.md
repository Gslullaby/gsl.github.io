---
title: Gradle依赖管理概览
date: 2020-08-22 23:36:28
tags:
- gradle
keywords:
- gradle
category:
- Gradle
description: 三部分内容：什么是依赖管理，Gradle中的依赖管理，以及一些相关术语的解释
---
## 什么是依赖管理
很少有项目是完全独立运行不需要其他任何依赖的。大多数情况下，项目都需要依赖一些可重用的公共库或者组件。从概念上讲，依赖项管理是一种自动化声明、解析和使用项目所需依赖的技术

## Gradle中的依赖管理
Gradle中已经内置了依赖管理，并且对现代软件项目中遇到的典型场景做了很好的支持。下图粗略展示了Gradle中依赖管理的原理
{% asset_img dependency-management-resolution.png Gradle中的依赖管理 %}

在编写构建脚本时，可以按照作用域来声明的依赖（如某项依赖仅用于源码编译或仅用于执行单元测试等）。在Gradle中，依赖的作用域称为`configurtion`，如Android中常见的`implementation`，`api`等

通常情况下，依赖以`module`的形式出现。我们需要告诉Gradle从哪里可以找到这些`module`，存储`module`的地方即为`repository`（仓库）。通过声明`repository`，Gradle就可以找到`module`。有两种形式的`repository`：本地`repository`或远程`repository`

在构建的运行阶段，如果某个任务的执行需要某项依赖，则Gradle会去特定的位置寻找依赖。依赖可能需要从远程仓库下载，或者从本地仓库中寻找，也可能是多项目构建中的其他project等。寻找依赖的过程称为**依赖解析**

依赖解析成功之后，该依赖项就会存储在本地缓存中（也称为依赖缓存）。依赖解析时会先从缓存中查找以减少网络下载

每个依赖都会有一些metadata，用于描述该依赖库的详细信息，如依赖库在仓库中的位置，作者等信息。通过metadata，也可以描述当前依赖所需要的其他依赖，如`JUnit 5`依赖需要平台的一些公共依赖。Gradle会自动解析这些依赖库需要的其他依赖库，称为`transtive dependencies`

## 依赖管理的一些术语
#### **Artifact**
没有一个能与之对应的中文词汇，官方翻译为*构件*，但是不是很好理解，可理解为构建产物。Artifact为构建输出的一个目录或文件。如一个JAR、ZIP或native可执行文件

文件形式的Artifact设计用于其他项目或部署在托管系统中。目录形式的Artifact通常出现在多项目构建中项目间的依赖

#### **Capability**
Capability用于标识一个或多个`component`所能提供的功能。Capability可以使用类似于`module`的版本号坐标那样的坐标进行标识。默认情况下，每个`module`的版本都提供与其坐标匹配的功能，例如`com.google:guava:18.0`。`capability`可用于描述一个`component`提供了多个特性的变体，或两个不同的`component`实现了同一特性（但是两个组件不能同时使用）

#### **Component**
依赖库的任意一个版本称为Component（组件）

对于外部库来说，术语`component`即为改库的一个发行版本

在构建中，`component`由插件（例如Java库插件）定义，并提供了一种简单的方法来定义待发布的。 其中包含了`artifact`以及适当的元数据用以详细描述`component`的变体。 例如，其默认设置中的Java Component由jar task输出的JAR以及Java api和运行时变体的依赖项信息组成。 它还可以使用相应的`variant`定义其他变体，例如`sources`和`Javadoc`。

#### **Configuration**
`Configuration`是一组特定目的的依赖
`configuration`提供了对底层以解析的`module`以及其`artifact`的访问

> “configuration”是一个重载术语，在依赖管理上下文之外的其他地方具有不同的含义。

#### **Denpendency**
`Denpendency（依赖）`是一个指向运行构建、测试或module所需软件的指针

#### **Dependency constraint**
`Dependency constraint`依赖约束定义了module需要满足的要求，以使其成为一个能有效解析的依赖项。 例如，依赖约束可以缩小支持的模块版本。 依赖约束可以用于描述对传递依赖（transitive dependencies）。

#### **Feature Variant**
`Feature Variant`是表示可单独选择或不可选择的组件特征的变量。一个`feature variant`由一个或多个`capability`标识

#### **Module**
`Module`是一组随时间不断迭代的代码库。每个module都有一个名字。每个module的发行版本都有一个版本号。在`repository`中可以定位到一个`module`

#### **Module metadata**
`module`的发行版中会带有`metadata`，用以详细描述该module，如如何定位到其中国的`artifact`，或当前module需要的其他依赖（transitive dependencies）等。Gradle不仅提供了名为Gradle Module Metadata(.module文件)，而且还支持Maven(.pom文件)以及lvy(ivy.xml)

#### **Component metadata rule**
`Component metadata rule`是在从`repository`中获取`component metadata`后对其进行修改的规则，如添加丢失的信息或纠正错误信息。与`resolution rules`不同的是，该规则应用于依赖解析开始之前。该规则是构建逻辑的一部分，并且可以通过插件进行共享。

#### **Module version**
即为module的版本号，随module的迭代更新而不断增加。如，`18.0`标识坐标为`com.google:guava:18.0`的module的版本号。

#### **Platform**
`Platform`是一组旨在一起使用的`module`。有不同种类的`platform`，应用于不同的使用场景：
* module set: 作为一个整体发布的一组`module`。使用其中一个module即意味着我们需要使用其他所有的module
* runtime environment: 一组已知可以良好地协作的库。
* deployment environment: 如Java runtime、应用服务等

#### **Publication**
对作为实体被发布至repository中供用户使用的文件或元数据的描述

`publication`拥有一个名字，它是由一个或多个artifact以及这些artifact的信息（metadata）共同组成的

#### **Repository**
Repository中存储了许许多多module，module通过版本号指明其发行版本。Repository可以基于二进制存储库产品(例如ArtiFactory或Nexus)或文件系统中的目录结构

#### **Resolution rule**
即依赖的解析规则，直接影响如何解析依赖。解析规则会作为构建逻辑的一部分定义在构建脚本中

#### **Transitive dependency**
一个组件可能需要依赖其他的组件才能正常工作，即为依赖传递。发行至仓库中的某个module可以提供metadata用以声明依赖的传递。默认情况下，Gradle会自动解析传递依赖。可通过声明依赖约束来改变传递依赖的版本选择。

#### **Variant (of a componment)**
每个组件由一到多个变体组成。变体由一组artifact组成，并定义了一组依赖。由一组attributes和capabilities标识

Gradle的依赖解析可识别变体，并在确定组件（即模块的一个版本）后为每个组件选择一个或多个变体。如果无法明确选择变体（即Gradle没有足够信息从互斥的变体中选择出一个合适的变体），则构建会失败。在这种情况下，可以通过`variant attribute`提供足够信息

#### **Variant Attribute**
变体属性用于识别和选择变体。一个变体定义了一到多个属性，如`org.gradle.usage=java-api, org.gradle.jvm.version=11`。在解析依赖时，会请求一组属性，Gradle会为依赖图中的每个组件找到最合适的变体。可以为属性实现兼容性和歧义消除规则，以表示版本间的兼容性(例如，Java 8与Java 11兼容，但如果请求的版本是11或更高版本，则应首选Java 11)。这些规则通常由插件提供