---
layout:	post
title:	"第一篇博客"
date:	2020-01-15 15:04:21 +0800
tags:	jekyll github
color:	rgb(255,90,90)
cover:	'../assets/th.jpg'
subtitle:	'搞了一天，记录一下是怎么搞出来的博客'
---

## 0x00 拥有github
- 为什么用github：
(￣▽￣)"方便不要钱，没有别的

- 注册github账号：

这个不用说了，之前就有账号了，直接在仓库里新建就行，会有个域名，比如我的是`sueisok`，域名就是`sueisok.github.io`了。不需要什么dnspod，什么dnsdaddy，什么一年50美元租服务器，贵且不好维护，且第三方平台比如wordpress可能有漏洞之类的，还要时常更新版本

## 0x01 选择的工具

本人电脑为win10系统

- [Ruby][ruby]:一种简单快捷的面向对象（面向对象程序设计）脚本语言，安装Jekyll需要电脑上安装Ruby
- [Jekyll][jekyll]：jekyll生成Git Pages，是一个生成静态网页的工具，不需要数据库支持。但是可以配合第三方服务，比如评论区可以有第三方支持，我这里用的是来必力
- [Typora][typora]:是一款支持实时预览的 Markdown 文本编辑器，很强大
- [来必力][laibili]:添加的第三方评论工具


[ruby]:https://rubyinstaller.org/downloads/
[jekyll]:http://jekyllcn.com/
[typora]:https://typora.io/
[laibili]:https://livere.com/

## 0x02 安装工具

### 安装ruby

- [Ruby官网][ruby]下载安装包，选择带DevKit的

- 安装到某个路径下即可

- 添加到环境变量

- 检查ruby是否安装成功：

```
$ ruby -v
```
- 检查gem版本，gem是一个管理ruby库和程序的标准包，用RubyInstaller安装ruby之后都附带有gem（不是邓紫棋

```
$ gem -v
```
### 安装bundle

```
$ gem install bundle
```

至于这是什么，我不知道，至于会出现什么报错，我没遇见

### 安装jekyll

```
$ gem install jekyll
```

成功的话就成功了。这里碎碎念一下，(ノへ￣、)公司电脑上就可以安装成功，家里自己的电脑就不成功

查看jekyll版本

```
$ jekyll -v
```

如果报错啥的，就按他提示，有可能是

```
$ bundle exec jekyll -v
```

显示了版本就是成功袅 我的是jekyll 3.8.6 

### 注册来必力

注册完将生成的内容添加到仓库文件的`_config.yml`里

## 0x03 发表自己的第一篇博客

我自己之前在github上有过主页，所以我这里只写如何应用一个自己喜欢的主题，然后提交到git上，让所有人都看到

### 把自己原先的仓库clone下来

原先github上有过主页，clone到本地一个路径下

```
$ git clone xxx你的仓库xxx
```

### 选择一个自己喜欢的主题

有好多主题的网站，我这个是在[jekyllthemes][jekyllthemes]找的，也可以在喜欢的博主页脚中找到他们用的主题（真是个小机灵鬼儿

[jekyllthemes]:http://jekyllthemes.org/

下载下来，如果这个主题能用的话，接着往下进行

如果直接把主题中的内容复制到仓库里，会出错，这里建议先在主题的目录下生成页面，再复制到clone的仓库目录下，在下载的主题目录下：

```
$ bundle exec jekyll serve
```

然后将生成的所有内容复制到自己目录下

如果之前仓库有自己的内容，比如`_posts`里的静态页面或者`_config.yml`的内容，需要修改后再替换

在自己的仓库目录下：

```
$ bundle exec jekyll serve
```

显示

```
Configuration file: xxxx/sueisok.github.io/_config.yml
            Source: xxxx/sueisok.github.io
       Destination: xxxx/sueisok.github.io/_site
 Incremental build: disabled. Enable with --incremental
      Generating...
       Jekyll Feed: Generating feed for posts
                    done in 2.183 seconds.
 Auto-regeneration: enabled for 'xxxx/sueisok.github.io'
  JekyllAdmin mode: production
    Server address: http://127.0.0.1:4000
  Server running... press ctrl-c to stop.
...
```

即可访问本地`http://127.0.0.1:4000`查看页面，直到自己满意了之后再提交到github上

### 用Typora写一篇内容

Typora是可以写markdown格式内容的编辑器，简约大方又不失便捷，还可以直接添加emoji表情，下次试一下

写完直接丢到仓库的`_posts`里面

### 将修改内容提交到github

可以下载[GitHub Desktop][githubdesktop]，比较直观

[githubdesktop]:https://desktop.github.com/

也可以手动提交到git，我这里目前只有一个master分支，操作也比较暴力因为只有我一个人维护，如果需要其他分支可自行百度

查看修改内容：

```
$ git status
```

将要提交的文件的信息添加到索引库

```
$ git add .
```

添加此次修改的备注

```
$ git commit -m "xxxx"
```

提交到master分支

```
$ git push origin master
```

网络差的话可能提交失败，再试一下就好了（〃｀ 3′〃）



然后稍等一下就可以看到自己的commit成功，在等一下就可以看到自己的文章更新好了，奥里给！