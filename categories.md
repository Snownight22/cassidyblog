---
layout: default
title: 分类
permalink: /categories
---

<style>
  #closeBtn, #openBtn {
      display: none;
  }
</style>

<script>
  function getParameter(name){
	var url = decodeURI(location.href);
	var paramStr = url.substring(url.search(/\?/) + 1);
	var params = paramStr.split("&");

	for(index in params){
	  var paramPair = params[index];
      var paramPairSeparator = paramPair.search(/=/) ;
	  var key = paramPair.substring(0 , paramPairSeparator );
	  var value = paramPair.substring(paramPairSeparator + 1);
	  if(name == key )
		return value ;
	}
  }
  
  var reqCat = getParameter("category");
  document.title='类别：' + reqCat;
//   console.log(reqCat);
</script>

<div class="category-div" id="categoryId">
  <div class="post post-archive">
    <h3 id="categoryTitle">category</h3>
    <ul class="arc-list">
    </ul>
  </div>
</div>

<script>
  function show_cate_title(title) {
    var element = document.getElementById('categoryTitle');
    element.innerText = title;
  }

  function show_postList(postList) {
    // console.log(postList);
    var html = '';
    for (var post of postList) {
        // console.log(post);
        html += '<li><span class="date">' + post.date + '</span><a href="' + post.url +'">' +  post.title + '</a></li>';
    }

    $('.arc-list').html(html);
    $('.arc-list').show();
  }

  {% for category in site.categories %}
    var cate = '{{ category | first }}';
    if (cate == reqCat) {
        console.log("OK, find\n");
        console.log(cate);
        show_cate_title(cate);
        var postLists = [];
        {% for post in category.last %}
            var p = { 
                title: '{{ post.title }}',
                date: '{{ post.date | date: "%B %e, %Y" }}',
                url: '{{ post.url | prepend:site.baseurl }}'
            };
            postLists.push(p);
        {% endfor %}
        show_postList(postLists);
    }
  {% endfor %}

</script>