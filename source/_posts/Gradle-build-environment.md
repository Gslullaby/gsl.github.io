---
title: Gradle构建环境
date: 2020-08-04 12:38:27
tags:
- gradle
keywords:
- gradle
category:
- Gradle
description: 本文介绍了Gradle构建环境的配置。包括对Gradle properties，System properties，Environment variables，Project properties等的配置
---
Gradle提供多种方式对Gradle本身以及Project的行为进行配置。下面列出了四种配置方式，按优先级从高到低排列（第一个方法优先级最高）：
* `Common-line flags` 如:`--build-cache`。优先级高于各种属性和环境变量
* `System properties` 如: 存储在`gradle.properties`中的`systemProp.http.proxyHost=somehost.org`
* `Gradle properties` 如: 存储在项目根目录或`GRADLE_USER_HOME`目录下`gradle.properties`中的`org.gradle.caching=true`
* `Environment variables` 如: Gradle运行环境中的`GRADLE_OPTS`变量

除了配置构建环境外，还可以通过`Project properties`来配置特定项目构建，如`-PreleaseType=final`

## Gradle properties
Gradle提供了一些选项，通过这些选项可以很方便的对用于执行构建的Java进程进行配置。虽然可以在本地环境中通过`GRADLE_OPTS`或`JAVA_OPTS`对该进程进行配置，但是把诸如JVM内存大小、Java Home位置等配置信息加入版本控制（如git/svn等），可以保证项目组构建环境的一致性。

配置一致的构建环境非常简单，只需在`gradle.properties`文件中添加配置即可。所有的`gradle.properties`合并在一起构成了构建环境的配置。如果多个`gradle.properties`出现了同一个配置项，则会第一个会生效。

四种设置方式：
* system properties，例如，当在命令行中设置`-Dgradle.user.home`
* `GRADLE_USER_HOME`目录中的 `gradle.properties`文件
* 项目根目录中的`gradle.properties`文件
* Gradle安装目录下的`gradle.properties`文件

下面列出了一些Gradle构建环境的配置项

`org.gradle.caching=(true,false)`   
当设置为true时，在可能的情况下，Gradle会复用之前构建的输出以加快构建。

`org.gradle.caching.debug=(true,false)`   
当设置为true时，所有任务的输入属性的hash以及构建缓存的key都会输出到console中

`org.gradle.configureondemand=(true,false)`   
启用按需配置，Gradle将尝试仅配置必要的项目

`org.gradle.console=(auto,plain,rich,verbose)`   
自定义console中log的颜色或log的级别。 默认值取决于调用Gradle的方式

`org.gradle.daemon=(true,false)`   
当设置为true时，将使用Gradle Daemon进程执行构建。默认值为true

`org.gradle.daemon.idletimeout=(# of idle millis)`   
设置Gradle Daemon进程自动关闭的时间，单位为毫秒。默认值为`10800000`(3小时)

`org.gradle.debug=(true,false)`   
当设置为true时，Gradle将在启用远程调试的情况下运行构建，并监听5005端口。该设置项等同于在JVM 命令行中添加`-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=5005`。该设置会挂起JVM直至有可绑定的debugger。默认为false

`org.gradle.java.home=(path to JDK home)`   
为Gradle构建进程指定Java home。可设置为`jdk`或`jre`路径，但是最好使用`jdk`。该设置项最合理默认值应该是从PC环境变量（JAVA_HOME或path中的java路径）中获取。该设置不会影响到运行Gradle客户端的JVM所使用的Java版本

`org.gradle.jvmargs=(JVM arguments)`   
为Gradle Daemon进程指定JVM参数。可以使用该配置修改JVM内存，以提升构建的执行效率。该设置不会影响Gradle客户端VM的JVM设置

`org.gradle.logging.level=(quiet,warn,lifecycle,info,debug)`   
用于设置Gradle的日志级别，可设置为quiet、warn，lifecycle，info，debug。不区分大小写。默认为`lifecycle`

`org.gradle.parallel=(true,false)`
配置后，Gradle将派生到org.gradle.workers.max JVM以并行执行项目

`org.gradle.priority=(low,normal)`
为Gradle daemon以及所有由它启动的进程指定调度优先级。默认为`normal`

`org.gradle.unsafe.watch-fs=(true,false)`
观察文件系统的开关。允许Gradle下次构建时重用fs信息。默认为false

`org.gradle.warning.mode=(all,fail,summary,none)`   
当设置为`all`/`summary`/`none`，Gradle会使用不同的警告类型显示

`org.gradle.workers.max=(max # of worker processes)`   
用于设置worker进程的最大数量。默认为CPU的核心数

下面例子演示如何使用以上属性

*Example 1.在gradle.properties文件中设置属性*
> gradle.properties
```
gradlePropertiesProp=gradlePropertiesValue
sysProp=shouldBeOverWrittenBySysProp
systemProp.system=systemValue
```
> build.gradle
```groovy
task printProps {
    doLast {
        println commandLineProjectProp
        println gradlePropertiesProp
        println systemProjectProp
        println System.properties['system']
    }
}
```
```groovy
$ gradle -q -PcommandLineProjectProp=commandLineProjectPropValue -Dorg.gradle.project.systemProjectProp=systemPropertyValue printProps

commandLineProjectPropValue
gradlePropertiesValue
systemPropertyValue
systemValue
```

## System properties
使用`-D`命令行选项，可以传入一些系统属性给运行Gradle的JVM。Gradle命令的-D选项与java命令的-D选项具有相同的效果。   
也可以在`gradle.properties`中使用`systemProp`前缀来设置system properties
如：
```
systemProp.gradle.wrapperUser=myuser
systemProp.gradle.wrapperPassword=mypassword
```

下面列出了可用的系统属性。注意命令行选项方式具有更高的优先级

`gradle.wrapperUser=(myuser)`   
指定用于从服务器（使用Http基本身份验证）下载Gradle的用户名。

`gradle.wrapperPassword=(mypassword)`
指定上一条属性中设置的用户名所对应的密码

`gradle.user.home=(path to directory)`
指定Gradle的用户主目录

在多项目构建中，在除根项目以外的项目中设置的"`systemProp.`"属性都将被忽略，也就是说，只有根项目的`gradle.properties`文件中以"`systemProp.`"为前缀的属性会被检测

## Environment variables
`gradle`命令有以下几个可用的环境变量。命令行选项以及系统变量具有比环境变量更高的优先级

`GRADLE_OPTS`   
设置Gradle client VM启动时使用的JVM参数。由于Client VM仅处理命令行input/output，因此很少需要更改其VM选项。实际的构建任务是由Gradle daemon进程执行的，因此该环境变量并不会影响构建

`GRADLE_USER_HOME`   
指定Gradle用户主目录（如果未设置，则默认值为`$USER_HOME/.gradle`）

`JAVA_HOME`
为client VM指定JDK的安装路径。此VM也用于daemon进程，除非在带有`org.gradle.java.home`的Gradle属性文件中指定了不同的VM

## Project properties
可以直接使用`-P`命令行选项为`Project`对象添加一些属性

当Gradle遇到一些特定名称的系统属性或环境变量时也会为Project设置属性。如：如果环境变量的名称为`ORG_GRADLE_PROJECT_prop=somevalue`，则Gradle会为Project对象设置一个名为`prop`值为`somevalue`的属性。对于系统属性来说，则使用了不同的命名模式，即为`org.gradle.project.prop`。

下面两个例子都会子啊Project对象中设置一个名为`foo`值为`bar`的属性

**通过系统属性设置Project属性**
`org.gradle.project.foo=bar`

**通过环境变量设置Project属性**
`ORG_GRADLE_PROJECT_foo=bar`

> 用户主目录中的属性文件优先级要高于项目目录中属性文件

当对一个持续集成服务没有管理员权限并且需要设置一些不容易被看到的属性值时，该特性非常有用。在此场景下，因为你无法使用`-P`选项，也无法更改系统层级的配置文件，所以正确的策略是修改持续集成构建任务的配置，添加一个与于其模式匹配的环境变量。这对系统上的普通用户是不可见的。

在构建脚本中可以像使用普通变量那样，通过属性名称即可访问Project的属性

> 如果应用了一个不存在的Project属性，则会抛出异常，进而构建失败
>
> 在使用`Project.hasProperty(java.lang.String)`方法访问某个属性之前，应该先检查该Project属性是否存在

## 配置JVM内存
可以使用以下方式调整JVM

Gradle属性`org.gradle.jvmargs`用以控制运行构建的VM。默认值为`-Xmx512m "-XX:MaxMetaspaceSize=256m"`

**修改构建VM的JVM参数**
`org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8`

环境变量`JAVA_OPTS`用以控制命令行终端，该终端仅用于显示控制台输出。默认值为`-Xmx64m`

**修改命令行终端VM的JVM参数**
`JAVA_OPTS="-Xmx64m -XX:MaxPermSize=64m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8"`

> 有一种情况终端VM也会作为构建VM使用，即：Gradle Daemon进程处于禁用状态，并且终端VM与构建VM的参数设置相同，则终端VM会直接执行构建。其他情况下终端VM会fork出一个新的VM用与执行构建

某些task，如`test` task，也会fork出额外的JVM进程。可以通过task本身对其进行配置。默认值为`-Xmx512m`

*Example 2. 为`JavaCompile`任务设置Java编译选项*
> build.gradle
```groovy
plugins {
    id 'java'
}

tasks.withType(JavaCompile) {
    options.compilerArgs += ['-Xdoclint:none', '-Xlint:none', '-nowarn']
}
```

## 使用Project属性配置任务
可以根据调用时指定的项目属性更改任务的行为

假设像确保只有CI才能触发发布版本操作，则简单是实现方式是通过使用Project的`isCI`属性

*Example 3. 禁止CI之外发布版本*
> build.gradle
```groovy
task performRelease {
    doLast {
        if (project.hasProperty("isCI")) {
            println("Performing release actions")
        } else {
            throw new InvalidUserDataException("Cannot perform release outside of CI")
        }
    }
}
```

> $ gradle performRelease -PisCI=true --quiet
Performing release actions

## 通过HTTP代理访问Web
通过标准的JVM系统属性可以配置HTTP/HTTPS代理（用于下载依赖等）。可以直接在构建脚本中设置这些属性，如使用`System.setProperty('http.proxyHost', 'www.somehost.org')`来设置HTTP代理。也可以在`gradle.properties`中进行设置

**`在gradle.properties`中配置HTTP代理**
```groovy
systemProp.http.proxyHost=www.somehost.org
systemProp.http.proxyPort=8080
systemProp.http.proxyUser=userid
systemProp.http.proxyPassword=password
systemProp.http.nonProxyHosts=*.nonproxyrepos.com|localhost
```
**`在gradle.properties`中配置HTTPS代理**
```groovy
systemProp.https.proxyHost=www.somehost.org
systemProp.https.proxyPort=8080
systemProp.https.proxyUser=userid
systemProp.https.proxyPassword=password
systemProp.https.nonProxyHosts=*.nonproxyrepos.com|localhost
```

为了访问一些特殊网络可能还需要设置一些其他属性。下面是两个参考链接
* [ProxySetup.java in the Ant codebase](https://git-wip-us.apache.org/repos/asf?p=ant.git;a=blob;f=src/main/org/apache/tools/ant/util/ProxySetup.java;hb=HEAD)
* [JDK 7 Networking Properties](http://download.oracle.com/javase/7/docs/technotes/guides/net/properties.html)

