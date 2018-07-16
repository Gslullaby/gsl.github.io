---
title: GitHub Pages + Hexo搭建博客汇总
categories:
- Web前端
tags:
- Hexo
- GitHub Pages
keywords:
- Hexo
- GitHub Pages
- Blog
description: 搭建个人blog
---
网上关于GitHub Pages + Hexo搭建博客的教程较多，因此此文旨在查缺补漏，记录整个流程，汇总一些参考的优质blog，以及梳理我在搭建过程中遇到的问题。

## 如何搭建个人blog
在开始之前，我先想了下如何搭建个人blog。既然blog是个网站，那么首先能想到的是，需要自己开发一个网站出来啊，有了网站，还需要服务器来部署我们的网站，然后还需要一个域名，方便访问我们的网站。    

#### 因此我们要做的工作有
* 开发自己的网站（网站设计，UI开发，评论统计系统集成维护等）
* 搭建服务器（部署网站，维护等）
* 购买域名（绑定域名）

首先开发网赚需要耗费巨大精力，其次购买服务器和域名都需要花钱，而我的初衷只是写blog啊，然而却需要做这么多，那岂不是舍本逐末了。好在已经有现成且免费的框架和工具，GitHub Pages
和Hexo帮我们做了上面几乎所有的事儿

#### GitHub Pages介绍
GitHub Pages是Github提供的一个免费静态网站的托管服务，旨在直接从GitHub仓库托管个人/组织/项目的页面，并且为站点提供了默认域名github.io
。也就是说GitHub Pages帮我们解决了 **服务器和域名** 这两大难题

详细介绍：[GitHub Pages官网](https://pages.github.com/)

#### Hexo介绍
Hexo是一款基于Node.js的静态博客框架。通过Hexo，用户可以定制自己喜欢的网站主题，专注与使用Markdown编写blog，它会帮用户生成静态网站并托管在GitHub，相当的简便。因此Hexo帮我们解决了** 开发网站 ** 的难题

详细介绍：[Hexo中文文档][]  

## 搭建个人blog的流程
在使用Hexo极力推荐阅读** [Hexo中文文档][] ** ,文档里的说明要比许多blog里清晰的多

#### 搭建前的准备工作
安装流程在上文中的[Hexo中文文档][]里有详细说明
* 安装Git
* 安装Node.js
* 安装Hexo  

#### 搭建流程  
这里只列出简单流程，定制主题，及git分支设置在后面介绍
1. GitHub上创建新仓库，仓库名为 ** GitHub用户名.github.io **
> 这里需要注意，仓库的名字必须为上面的格式，否则部署后将无法访问blog

2. GitHub新建分支hexo，并设置为默认分支
3. 使用Git克隆新建仓库到本地（此时为hexo分支）
``` bash
$ git clone https://github.com/github_user_name/github_user_name.github.io.git
```
4. 在仓库根目录使用hexo init命令初始化（此时为hexo分支）
``` bash
$ hexo init
```
5. 上一步完成后依次使用hexo g 与 hexo s（此时为hexo分支），其中g为generator，s为server
``` bash
$ hexo g
$ hexo s
```
6. 打开浏览器输入localhost:4000查看本地blog效果，如下图
{% asset_img hexo_landscape.jpeg Hexo默认主题 %}
7. 修改仓库根目录下的_config.yml文件，修改deploy中的以下字段并保存，注意branch要设置为master
``` YAML
deploy:    
    type: git
    repo: https://github.com/github_user_name/github_user_name.github.io.git
    branch: master
```
9. 提交本地变更至hexo分支
``` bash
$ git add .
$ git commit -m 'commit msg'
$ git push origin hexo
```
10. 使用hexo d命令完成blog部署
``` bash
$ hexo s
```

#### blog目录结构
在上面流程的第4步中，执行完hexo init命令后，会在文件夹下生成hexo工程，以下简要展示生成的目录结构及各个目录的作用，更为详细的解释见[Hexo目录结构][]
```text
|-- _config.yml      // 网站全局配置文件，详细说明见
|-- package.json     //
|-- scaffolds        //
|-- source           // 网站资源文件夹，存放网站图片，文章等
   |-- _drafts
   |-- _posts        // 我们的Markdown和HTML文件存放在次文件夹
|-- themes
   |-- landscape     // 官方默认的主题
   |-- next          // next主题

```

经过第4步之后，通过修改\_config.yml文件进行网站的配置,如网站title，author等，详情见[Hexo网站配置][]。其中最应该注意的是deploy字段，该字段用于配置网站的部署，如网站部署到git的xxx仓库的xxx分支，或是地址为xxx.xxx.xxx.xxx的服务器，详情见[Hexo deploy]    

#### 设置主题
Hexo允许用户自定义主题，下面展示主题的设置方式

1. 挑选自己喜欢的主题，找到其仓库地址
2. 命令行cd 定位到themes文件夹，将主题clone至该文件夹
``` bash
$ cd themes
$ git clone <theme repo>
```
3. 在clone完成后，新添加的文件夹名称即为主题名称xxx，打开\_config.yml将theme字段的landscape修改为xxx

关于主题的挑选，如果没有web前段经验，最好挑选那些star数计较高，社区比较活跃，经常迭代维护的主题，如我自己使用的next主题，下面是我在挑选主题时参考的资料   

[知乎-有哪些好看的Hexo主题](https://www.zhihu.com/question/24422335/answer/46357100)
>高赞回答用python爬虫爬出来的主题star排行榜 ,推荐参考这个挑选主题     

[官方主题列表](https://hexo.io/themes/index.html)
> 官方没有对主题进行排名，挑选时稍微麻烦些

[推荐NexT主题](https://theme-next.iissnan.com/getting-started.html)
> NexT主题外观比较简洁利落，并且社区活跃，有详尽的中文文档，主题的外观也可进行各种灵活的配置，是前段盲的必备良药。对于该主题的详细用法，其官方文档中有详细的说明，也就不再赘述了

#### 关于git仓库的说明
在搭建流程中的前两步，建立完仓库后，又新建了一个hexo分支，并且设置为默认分支。这是因为我们在维护我们的网站时，要维护以下两部分内容
* Hexo工程
``` text
hexo init之后生成的部分   
```

* 生成的静态网站
``` text
之前在_config.yml中配置了deploy的git仓库及分支，这里我配置的是master分支，在执行了hexo deploy命令之后，Hexo就会生成静态静态网站提交至master分支并自动部署
```
这两部分都需要提交至git仓库的，第一部分提交至git，我们就可以随时随地只要有电脑网络就可以clone工程，写blog并更新网站。第二部分是GitHub Pages部署网站必须的部分。这两部分的内容互不相干，因此建议新建hexo分支，用于存储我们的Hexo工程，master分支用于部署网站

参考的文档：[知乎-使用Hexo，如果换了电脑怎么更新博客](https://www.zhihu.com/question/21193762)

#### 关于NexT主题个人签名不显示的问题
在NexT主题配置文件\_config.yml中，如果设置seo字段为true，则网站配置文件\_config.yml中的description字段会被用于seo，而不会显示为签名，这时如果想要显示个人签名，需要额外添加signature字段
```YAML
signature: 随行、随记
```

## Hexo SEO优化
辛辛苦苦搭好了blog，写了文章，最后发现浏览器里搜不到啊，根本没人看啊，怎么办啊，好没有成就感。这时候就需要SEO出马了，那么什么是SEO呢？
> SEO 英文全称Search Engine Optimization 即搜索引擎优化。是一种利用搜索引擎规则提高网站在有关引擎内的自然排名

既然是针对搜索引擎的优化，那么也就是针对Baidu和Google的优化了。相关blog有很多，并且都是死套路，所以这里就挑选几篇比较新比较好的blog列出来

[Hexo博客Next主题SEO优化方法](https://hoxis.github.io/Hexo+Next%20SEO%E4%BC%98%E5%8C%96.html)
[Hexo个人博客SEO优化系列](https://juejin.im/post/5ae7f8a2f265da0ba266c5c6)




[Hexo中文文档]:https://hexo.io/zh-cn/docs/index.html
[Hexo目录结构]:https://hexo.io/zh-cn/docs/setup.html
[Hexo网站配置]:https://hexo.io/zh-cn/docs/configuration.html
[Hexo deploy]:https://hexo.io/zh-cn/docs/deployment.html
