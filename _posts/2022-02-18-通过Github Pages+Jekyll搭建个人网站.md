---
layout: post
title:  通过Github Pages+Jekyll搭建个人网站
date:   2022-02-18 18:36:00 +0800
categories: 杂项
tag: 教程
---

* content
{:toc}


GitHub Pages是一个静态的站点托管服务，它直接从GitHub的存储库中获取HTML、CSS和JavaScript文件，通过构建系统选择性地运行这些文件，并发布一个网站。我们都可以从自己的Github上通过pages发布一个个人网站。
## 搭建github pages网站
### 首先 注册一个Github帐户
进入github 主页https://github.com/，点击signup注册一个帐户，有github帐户的略过。
### 添加一个新仓库
登陆github后，在右上角点击+号，New repository, 添加一个新仓库。

![githubCreateRepo.jpg]({{site.baseurl}}/styles/images/githubPages/githubCreateRepo.jpg)  

起一个你喜欢的名字。注意一定要是public的，private的pages要收费的哦！  

![githubCreateRepo1.jpg]({{site.baseurl}}/styles/images/githubPages/githubCreateRepo1.jpg)

### 创建主页
创建成功后将仓库克隆到本地，
`git clone https://username.github.com/test.git`
或按github提示在本地建立一个git文件夹，然后跟github此项目关联。

![gitCommit.jpg]({{site.baseurl}}/styles/images/githubPages/gitCommit.jpg)

在仓库主目录下添加一个主页 index.html，在里面随便写点内容。

![createIndex.jpg]({{site.baseurl}}/styles/images/githubPages/createIndex.jpg)

然后把它push到你的远程仓库。  
(Git的操作可以参考[廖雪峰的git教程](https://www.liaoxuefeng.com/wiki/896043488029600))

### 发布主页
在github远程仓库设置pages:

![githubPagesSetting3.jpg]({{site.baseurl}}/styles/images/githubPages/githubPagesSetting3.jpg)

source中选分支根目录，然后点save。你主页的地址会出现在上方。点击主页url，可以访问你的主页了。

## 使用Jekyll生成博客
Jekyll是一个可以将纯文本转换为静态博客网站的工具，本地可以安装jekyll，然后在博客目录启动jekyll，就可以通过访问本地地址http://127.0.0.1:4000来本地预览网站效果。
### 安装Jekyll
1. Jekyll安装需要Ruby支持，点击<https://rubyinstaller.org/>下载并安装Ruby；
2. 点击<https://rubygems.org/pages/download>下载并安装RubyGems；
3. 安装jekyll，在命令行执行 `gem install jekyll`；
4. 从<http://jekyllthemes.org/>上选择一个jekyll主题，下载下来，将所有文件放到之前克隆到本地的仓库根目录中，执行`jekyll serve`，就可以在本地预览博客了。

![jekyllStart.jpg]({{site.baseurl}}/styles/images/githubPages/jekyllStart.jpg)

### Jekyll目录结构
Jekyll官网（<http://jekyllcn.com/docs/structure/>）　详细介绍了jekyll的目录结构。

![jekyllDirStruction.jpg]({{site.baseurl}}/styles/images/githubPages/jekyllDirStruction.jpg)

其中比较重要的文件或目录：
index.html　网站入口页面  
_config.yml　保存网站配置数据  
_posts　这个目录保存你的文章内容  
_layouts　存放网页布局模板  
_includes　可以包含到布局或文章中的html片段  
_drafts　草稿，未发布的文章  
_site　jekyll启动时转换的静态页面  

### LessOrMore主题配置
我们知道了jekyll的目录结构，就可以根据需要定制所选主题的显示了。
jekyll主题中我选择了lessOrMore的主题。

![lessOrMoreTheme.jpg]({{site.baseurl}}/styles/images/githubPages/lessOrMoreTheme.jpg)

可以通过主题主页<https://github.com/luoyan35714/LessOrMore>　查看主题的主要修改配置：

![lessOrMoreConfig.jpg]({{site.baseurl}}/styles/images/githubPages/lessOrMoreConfig.jpg)

这个主题没有评论功能，这里我添加了gitalk来添加评论功能，可以查看[11111](www.baidu.com)，里面记录了如何给该主题添加gitalk评论模块。

至此，网站搭建完毕，可以在你的博客写文章了。


---
#### 参考：  
[搭建一个免费的，无限流量的Blog----github Pages和Jekyll入门](http://www.ruanyifeng.com/blog/2012/08/blogging_with_jekyll.html)  
[LessOrMore主页](https://github.com/luoyan35714/LessOrMore)  
[Jekyll文档指南](http://jekyllcn.com/docs/home/)  






