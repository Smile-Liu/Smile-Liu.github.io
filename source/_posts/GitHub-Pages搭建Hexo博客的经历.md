---
title: GitHub Pages搭建Hexo博客的经历
description: 记录在GitHub上使用Hexo搭建个人博客的一次经历
date: 2020-02-20 16:06:25
category:
    - Technology
    - Methodology
tags: 
    - ['git', 'hexo']
---
### 前言
&emsp;&emsp;2020年一开始就有种想要看看世界上的程序员们都在干什么事情的想法，于是就去`Stackoverflow`上注册了个账号。结果是使用GitHub账号可以关联，并且看到有填写个人博客的地方。想着这么国际化的地方当然需要一个国际化的博客才能装X呀，于是就想着在`GitHub`上搭建一个个人博客。百度了好久，发现大家经常使用`Hexo`或者`jekyll`框架。但是`jekyll`使用的是`Ruby`语言，咱也不熟啊，`Hexo`使用的是`NodeJs`，这个熟啊，就选这个吧。说干就干。
<!-- more -->
### 建立放博客的GitHub Repository
&emsp;&emsp;当然首先需要一个`GitHub`的账户，没有的同学请自行百度。<br>
&emsp;&emsp;然后新建一个仓库，不会的同学也自行百度一下。本文主要讲解如何把默认仓库改造成`GitHub Pages`，以适应博客所需。
1. 在新建的仓库上选择`Setting`，如下图。

2. 修改仓库名称为`yourname.github.io`，如图。
**&emsp;&emsp;这个是很重要的，一定起名为这种格式，否则最后提交文章时会出现新的分支，而不是master。yourname是指你的登录账户名称，也就是user.name，不是user.email**

3. 修改`GitHub Pages`的设置项`source`为`master`，如图。
**&emsp;&emsp;对于新建的仓库，需要添加一个文件之后，才可以选到这一项，否则是灰色的不可选**

4. 网上很多教程都说要开启SSH，其实没必要，所以这里不再写。
5. 关于自定义域名（`Custom domain`），有以下几点要求：
    - 自定义域名保存后会在`master`自动生成一个`CNAME`的文件
    - 其实这个文件是可以自己添加的，但是一定是叫`CNAME`
    - 其内容只能有一个域名
    - 其内容必须是一个裸域，类似`www.example.com`、`blog.example.com`、`example.com`，`com`可以换成其他的后缀
    - 其内容只能用于一个`GitHub`仓库，如果绑定了其他的仓库，则不能再使用了

### 本地安装Hexo
&emsp;&emsp;首先得安装`Node`，如果本机有，则要升级到最新版。
&emsp;&emsp;下面来安装`Hexo`。
1. 打开cmd，跳转到你要的目录。我这里是`D:\work\git`
2. 依次输入以下命令。
``` js
npm install -g hexo-cli

hexo init myBlog

cd myBlog

npm install
```
3. 执行完成之后的目录结构如下。
``` 
·
|-- _config.yml # 网站的配置文件
|-- package.json
|-- scaffolds # 模板文件夹
|-- source # 资源文件夹，除_posts文件外，其他以下划线开头的文件或文件夹不会被编译到public中
|   |-- _drafts # 草稿文件
|   |-- _posts # 文章
|-- themes # 主题
```
4. 根目录命令行下执行启动命令（s是server的意思，就是启动服务），然后在浏览器访问`http://localhost:4000`。
``` js
hexo s
```
5. 预览页面打开后就表示安装成功了。

### 写作
1. 具体的写作格式就是`MarkDown`语法。请自行百度。
2. `Hexo`的详细文档，请参考[官网](https://hexo.io/zh-cn/docs/index.html "Hexo IO")

### 发布GitHub
&emsp;&emsp;发布就是把本地的文章和线上的GitHub仓库关联起来。首次发布和日常发布是不一样的。
#### 首次发布
&emsp;&emsp;首次发布需要修改网站的配置文件，并且安装一个部署插件
1. 修改网站根目录下的`_config.yml`。在尾部找到`deploy`配置项，进行如下修改。
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repo:
    github: https://github.com/yourusername/yourusername.github.io.git
  branch: master
```
`repo`部分还可以这样写：
```
# Deployment
## Docs: https://hexo.io/docs/deployment.html
deploy:
  type: git
  repository: git@github.com:yourusername/yourusername.github.io.git
  branch: master
```
2. 安装部署插件。在网站根目录命令行下执行以下命令：
``` js
npm install hexo-developer-git --save
```
3. 部署并上传。在根目录命令行下执行以下命令：
``` js
hexo g -d
```
其中，`g`是`generate`，`d`是`develop`
4. 看到 提交`master`完成，就表示正常完成了。

#### 日常发布
&emsp;&emsp;在日常写完文章之后，都可以在本地实时预览，不需要每次都重启`hexo s`，但是想要部署到线上，则需要如下操作。
1. 在网站根目录的命令行中执行命令：
``` js
hexo g -d
```

### 更换主题
&emsp;&emsp;如果觉得默认主题不好看，可以去[Hexo主题官网](https://hexo.io/themes/)挑选一款喜欢的。具体更换步骤如下：
1. 找到主题的Git地址，在网站根目录下执行以下命令。或者手动下载主题的源码，复制到`\themes\`下。
``` git
git clone git@github.com:theme_url themes\name
```
其中，`git@github.com:theme_url`指主题的Git地址，`themes\name`指主题文件放置到`themes`文件夹下对应的目录中

2. 这时，主题里面有一份`_config.yml`，咱称之为*主题配置文件*；在网站根目录下也有一份`_config.yml`，称之为*网站配置文件*

3. 修改*网站配置文件*中的`theme`为对应主题的名字，保存，然后重启`hexo s`，即可生效了。

4. 博主试了几种主题，感觉对于技术类的文章来说，`NexT`还是比较适合的，网上的说明也够齐全。下面给出一份比较好的具体配置`NexT`主题的文章，请大家自行研究。[---> 这是传送门](https://blog.csdn.net/weixin_39345384/article/details/80785373 "作者：Hunter1023")

### 公司和家里同时操作的解决方案
&emsp;&emsp;命令`hexo g -d`只会把生成的文章上传到GitHub，整个网站的源码其实并没有上传。所以要实现公司和家里多台电脑同时写文章的还需要一些手段。<br/>
&emsp;&emsp;该手段的根本思想就是再建一个分支，把源码上传到该分支上，多地电脑同时操作这个分支就可以了。
1. 首先，创建`hexo`的分支。
2. 将该分支设置为**默认分支**。
3. 在创建博客的电脑上，把`hexo`分支克隆到本地。
``` git
git clone -b branch.git
```
4. 将网站的全部文件都复制到分支所在目录下。
5. 删除`node_modules`目录和主题目录下所有的`.git`目录。因为一个git仓库中不能包含另一个git仓库，否则提交主题目录会失败。
6. 因为`hexo g -d`已经绑定`master`分支，而源码则保存在`hexo`分支，互不干扰。

### 结语
&emsp;&emsp;本文简要的介绍了如何搭建个人博客的过程。博主亲测，方法可用。如有疑问，给我留言；如果觉得文章还可以，可以打赏一二。