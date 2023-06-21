---
layout: post
title:  使用 LeanCloud 为 Github Pages 添加阅读量统计
date:   2023-06-18 22:30:00 +0800
categories: 杂项
tag: 教程
---

<!--
* content
{:toc}
-->

使用 github Pages 写博客这么久，忽然发现一个问题，不蒜子对每篇文章的阅读量统计是不正确的，它的计数是整个网站的计数，并未查找到原因。因此想换一个统计方式。比较流行是是使用 LeanCloud。因此在个人网站上加入 LeanCloud 进行访问量统计。  

### 注册 LeanCloud

去 [LeanCloud 官网](https://www.leancloud.cn/)注册一个用户并登陆，注册过程不再缀述。  

### 创建应用

点击'创建应用'按键，名称随便填一个，下面选开发版。如图：  

![leancloud_createApp.png]({{site.baseurl}}/styles/images/githubPages/leancloud_createApp.png)

### 创建类

左侧选择结构化数据，右侧点击创建Class，类名填 Counter，其它都默认。  

![leancloud_createClass.png]({{site.baseurl}}/styles/images/githubPages/leancloud_createClass.png)  
![leancloud_tocreateClass.png]({{site.baseurl}}/styles/images/githubPages/leancloud_tocreateClass.png)  

创建完类就有了数据库表，表名即为类名 Counter，默认列有 ObjectId, ACL, CreatedAt, UpdatedAt 四列，我们创建用于存储访问量的三个列，Url(String):页面地址，Time(Number):访问次数，Title(String):页面标题。  

![leancloud_addColumnResult.png]({{site.baseurl}}/styles/images/githubPages/leancloud_addColumnResult.png)  

### 安全设置

leanCloud 还有一个安全设置，左侧点击设置 -> 安全中心，在右侧有一个 web安全域名，在里面填上你博客的地址。  

![leancloud_safeDomain.png]({{site.baseurl}}/styles/images/githubPages/leancloud_safeDomain.png)  

到这里，leanCloud 就设置完成了，点击设置里的应用凭证，把右侧的 AppID 和 AppKey 记录下来，后面博客配置中会用到。  

![leancloud_appid.png]({{site.baseurl}}/styles/images/githubPages/leancloud_appid.png)  

### 博客配置及js代码

在网站配置文件 _config.yml 中添加以下配置：  

```
# leanCloud 配置
leanCloud:
  enable: true    #计数开关
  appId: xxxxxx    #你的AppID
  appKey: xxxxxx    #你的AppKey
  counter: counter    #你创建的类名
```

在首页分页显示页面及 post 页分别加入以下显示计数的元素：  

```
<span id={{page.url}} class="leancloud-visitors" data-flag-title={{page.title}}>
    <span class="post-meta-divider">|</span>
    <span class="post-meta-item-text">阅读量 </span>
    <span class="leancloud-visitors-count">-</span>
</span>
```

然后新建一个文件记录 js 代码，我这里文件名为 readerStatistics，写入以下代码：  

```
<script src="https://cdnjs.loli.net/ajax/libs/jquery/3.2.1/jquery.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/leancloud-storage@3.11.1/dist/av-min.js"></script>
<script src="//unpkg.com/valine/dist/Valine.min.js"></script>

<script>
    AV.initialize('{ {site.leanCloud.appId}}', '{ {site.leanCloud.appKey}}');
</script>
<script>
    // 自己创建的Class的名字
    var name='{ {site.leanCloud.counter}}';
    // 创建新记录
    function createRecord(Counter, url){
        // 设置 ACL
        var acl = new AV.ACL();
        acl.setPublicReadAccess(true);
        acl.setPublicWriteAccess(true);
        var $element = $(document.getElementById(url));

        var title = $element.attr('data-flag-title').trim();
        var url = $element.attr('id').trim();
        var newcounter = new Counter();
        newcounter.setACL(acl);
        newcounter.set("Title", title);
        newcounter.set("Url", url);
        newcounter.set("Time", 1);
        // 获得span的所有元素
        var allcounter=[];
        allcounter.push(newcounter);

        AV.Object.saveAll(allcounter).then(function (todo) {
            // 成功保存记录之后
            console.log('创建记录成功！');
        }, function (error) {
            // 异常错误 
             console.error('创建记录失败: ' + error.message);
        });
        $element.find(".leancloud-visitors-count").text("1");
    }

    // 添加计数
    function addCount(Counter) {
        var $visitors = $(".leancloud-visitors");
        var url = $visitors.attr('id').trim();
        var title = $visitors.attr('data-flag-title').trim();
        var query = new AV.Query(Counter);
        query.equalTo("Url", url);
        query.find().then(function(results) {
            if (results.length > 0) {
                var counter = results[0];
                counter.increment("Time");
                counter.save().then(function(todo) {
                    console.log("increse number ok!");
                    var $element = $(document.getElementById(url));
                    var times = counter.get("Time");
                    $element.find(".leancloud-visitors-count").text(times);
                }, function(error) {
                    console.log("Error:" + error.code + ", " + error.message);
                });
            } else {
                console.log("创建计数：" + url);
                createRecord(Counter, url);
            }
        }, function(error) {
            console.log("Error:" + error.code + ", " + error.message);
        });
    }

    // 显示计数
    function showCount(Counter){
        var query = new AV.Query(name);
        var $postInfo = $(".leancloud-visitors");
        var url = $postInfo.attr('id').trim();
        query.equalTo("Url", url);
        query.find().then(function(results) {
            if (results.length > 0) {
                var counter = results[0];
                var $element = $(document.getElementById(url));
                var times = counter.get("Time");
                $element.find(".leancloud-visitors-count").text(times);
            } else {
                console.log("未找到对应计数:"+ url);
            }
        }, function(error) {
            console.log("Error:" + error.code + ", " + error.message);
        });
    }
    
    // 显示所有计数
    function showAllCount(Counter) {
        var query = new AV.Query(name);
        var elements = document.getElementsByClassName("leancloud-visitors");
        var urlList = [];
        for (var i=0; i < elements.length; i++) {
            var url = elements[i].id;
            urlList.push(url);
        }
        query.containedIn("Url", urlList);
        query.find().then(function(results) {
            for (var i = 0; i < results.length; i++) {
                var counter = results[i];
                var url = counter.get("Url");
                var times = counter.get("Time");
                var $element = $(document.getElementById(url));
                $element.find(".leancloud-visitors-count").text(times);
            }
        }, function(error) {
            console.log("Error:" + error.code + ", " + error.message);
        });
    }
</script>
```

其中，showCount 函数用于在 post 页显示计数，showAllCount 函数用于首页分页显示中显示计数，分别在两个页面添加以下执行函数：  

```
// post页面，添加并显示计数
{ % if site.leanCloud.enable %}
{ % include readerStatistics %}
<script>
    $(function() {
        var Counter = AV.Object.extend(name);
        addCount(Counter);
        showCount(Counter);
    });
</script>
{ % endif %}

// 首页页面，只显示计数
{ % if site.leanCloud.enable %}
{ % include readerStatistics %}
<script>
    $(function() {
      var Counter = AV.Object.extend(name);
      showAllCount(Counter);
    });
</script>
{ % endif %}
```

效果如下图：  

![leancloud_homePage.png]({{site.baseurl}}/styles/images/githubPages/leancloud_homePage.png)  

![leancloud_postPage.png]({{site.baseurl}}/styles/images/githubPages/leancloud_postPage.png)  

---
**参考**：  
[使用LeanCloud为Github Pages添加阅读量统计功能](https://zhuanlan.zhihu.com/p/353378112)  
[jekyll使用LeanCloud记录文章的访问次数](https://priesttomb.github.io/%E6%97%A5%E5%B8%B8/2017/11/06/jekyll%E4%BD%BF%E7%94%A8LeanCloud%E8%AE%B0%E5%BD%95%E6%96%87%E7%AB%A0%E7%9A%84%E8%AE%BF%E9%97%AE%E6%AC%A1%E6%95%B0/)  
[hexo史上最全搭建教程](http://www.5ityx.com/cate117/85430.html)  
