Diary Theme for Jekyll
======================

[中文说明](https://github.com/soyaine/jekyll-theme-diary/blob/master/README_CN.md)

一个简洁的日历 [Jekyll](http://jekyllrb.com/) 主题。

![Screenshot](http://i2.muimg.com/588926/15b8a32295edd802.jpg)


特性
-------

* **日历** — 通过日历的形式展示你的文章\日记。
* **自适应** — 此主题适应了电脑及移动端的不同屏幕尺寸。

![DEMO](https://cl.ly/323z0m3k1h1J/Screen%20recording%202017-05-15%20at%2009.33.52%20PM.gif)


使用
--------------

使用 `git clone https://github.com/soyaine/jekyll-theme-diary.git` 获取主题的代码，修改 `_config.yml` 后即可使用。

配置
--------------

在 `_config.yml` 文件里面修改配置信息：

```
name: YourName(显示在页面底部)
 
# 修改社交账号链接
# 此处只需修改冒号后的内容即可
link:
  email : yourname@gmail.com
  github : yourname
  twitter : yourname
  jianshu: http://www.jianshu.com/users/yourname

# 导航栏链接信息
# 此处可自定义任意标题及链接
nav:
   Home: http://yoursite.cn
   Diary: http://yoursite.cn/diary/
   ANYTHING: any url

```

示例
-------

你可以查看[主题示例](https://soyaine.github.io/jekyll-theme-diary/)，也可以参照[我使用这个主题的网站](https://soyaine.github.io/diary/)。

问题
-------
此主题会保持更新和不断完善，如果你有什么好的建议，或是发现了什么问题，请[提 issue ](https://github.com/soyaine/jekyll-theme-diary/issues/new)告诉我。

补充
-------
以上为原主题的说明，在此基础上我又做了些修改：
1. 社交账号配置中去除了 twitter 和 jianshu 的配置，添加了微博，微信，其它博客以及定阅功能的链接。  
2. 导航栏中修改为首页、归档、分类、关于和搜索功能，并在界面上以图标形式显示。  
3. 添加了博文中对 LaTeX 数学符号的支持。  
4. 添加了 gittalk 评论功能。  
5. 添加了 leanCloud 博文计数的功能。  

License
---------
MIT