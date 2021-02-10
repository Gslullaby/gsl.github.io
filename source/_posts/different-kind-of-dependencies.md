---
title: Gradle中的三种依赖方式
date: 2020-09-06 16:59:05
tags: 
- gradle
keywords:
- gradle
category:
- Gradle
description: 根据源的不同，Gradle有三种不同的依赖方式，分别是Module依赖、 文件依赖、项目依赖
---
## Module依赖
Module依赖方式是最常用的依赖方式。即从仓库中引用一个Module。如下面例子
```groovy
dependencies {
    // map方式声明
    runtimeOnly group: 'org.springframework', name: 'spring-core', version: '2.5'
    // string方式声明
    runtimeOnly 'org.springframework:spring-core:2.5',
            'org.springframework:spring-aop:2.5'
    runtimeOnly(
        [group: 'org.springframework', name: 'spring-core', version: '2.5'],
        [group: 'org.springframework', name: 'spring-aop', version: '2.5']
    )
    runtimeOnly('org.hibernate:hibernate:3.0.5') {
        transitive = true
    }
    runtimeOnly group: 'org.hibernate', name: 'hibernate', version: '3.0.5', transitive: true
    runtimeOnly(group: 'org.hibernate', name: 'hibernate', version: '3.0.5') {
        transitive = true
    }
}
```

如上面例子所示，有两种方式声明Module依赖：
1. string方式
2. map方式

module依赖有一组API对依赖进行配置。API包含了一些属性及配置方法。使用string方式进行声明时，仅能使用API中的部分属性。但是map方式可以使用全部属性。

> module依赖的解析方式
> 当声明了一项module依赖，Gradle会从声明的仓库中查找metadata文件（.module、.pom、ivy.xml）。如果找到了metadata文件，则进行解析并下载该module的artifact。若未找到则从Gradle 6.0开始，需要配置用于查找metadata的metadata源。

> 在Maven中，一个module有且仅有一个artifact
> 但在Gradle和lvy中，一个module可以拥有多个artifact。每个artifact可以拥有一组不同的依赖

## 文件依赖
某些项目可能并不依赖于二进制资源库中的lib。而是将依赖托管到某个共享磁盘，或是把依赖与源码一起加入到vc（git，svn等）中。这种依赖即为文件依赖，文件依赖没有metadata（依赖传递信息，作者等信息）

{% asset_img dependency-management-file-dependencies.png 从本地文件系统或共享磁盘中解析文件依赖 %}

下例展示了分别从`ant`、`libs`和`tools`目录解析依赖
```groovy
configurations {
    antContrib
    externalLibs
    deploymentTools
}

dependencies {
    antContrib files('ant/antcontrib.jar')
    externalLibs files('libs/commons-lang.jar', 'libs/log4j.jar')
    deploymentTools(fileTree('tools') { include '*.exe' })
}
```

如上例所示，每项文件依赖都需要定义文件的在fs中的精确路径。

> 需要注意的是`FileTree`中定义的文件的顺序是不稳定的。即使用`freeTree`声明文件依赖，最终的解析结果可能会不同，进而可能会影响到以此解析结果作为输出的任务的可缓存性。因此建议尽使用`files`

还可以使用某个任务的输出文件作为文件依赖的源，如下例
```groovy
dependencies {
    implementation files("$buildDir/classes") {
        // 声明使用compile任务输出的文件作为源
        builtBy 'compile'
    }
}

// compile任务，实际应用中会输出一系列文件
task compile {
    doLast {
        println 'compiling classes'
    }
}

// 测试task，打印每个配置的编译classpath
task list(dependsOn: configurations.compileClasspath) {
    doLast {
        println "classpath = ${configurations.compileClasspath.collect { File file -> file.name }}"
    }
}
```
list任务的输出为：
```groovy
$ gradle -q list
compiling classes
classpath = [classes]
```

## 项目依赖
一个项目通常会被分解为多个组件module，以提高项目的可维护性并去除了强耦合。项目中的组件module之间可以互相依赖，以重用代码

{% asset_img dependency-management-project-dependencies.png  多项目构建中，项目间的互相依赖%}

Gradle会对module之间的依赖关系进行建模。因为每个module由一个Gradle项目标识，因此这些将这些依赖称为项目依赖

```groovy
dependencies {
    implementation project(':shared')
}
```

在运行时，构建会自动确保以正确的顺序构建项目依赖项，并将其添加到类路径以进行编译。


至此三种依赖方式就介绍完了...