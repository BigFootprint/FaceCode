---
title: Hexo 高级玩法和坑
date: 2016-12-03 18:09:18
tags: [Hexo]
categories: 工具
---

Hexo 虽然已经有很多的主题实现了各种酷炫的功能和排版，但是依然有人不满足。本文就结合自己部署博客时所做改动介绍两个大家玩的比较溜的高级玩法，以及这两个玩法要注意的问题。

> [Hexo 插件官网](https://hexo.io/plugins/)上有很多插件可以选择。Hexo的插件安装非常简单，在博客的根目录下面执行:`npm install -g 插件名`即可。

## 置顶功能

置顶功能相关的帖子最早出处应该是[解决Hexo置顶问题](http://www.netcan666.com/2015/11/22/%E8%A7%A3%E5%86%B3Hexo%E7%BD%AE%E9%A1%B6%E9%97%AE%E9%A2%98/)这篇文章。Hexo 在安装的时候就自带了生成 Index 的插件，也就是`/node_modules/hexo-generator-index`插件，这个插件的具体玩法在[这里](https://github.com/hexojs/hexo-generator-index)。可以有两个配置项，其中一个就表示根据 date 排序，也就是文章的创建日期。<!-- more -->

[解决Hexo置顶问题](http://www.netcan666.com/2015/11/22/%E8%A7%A3%E5%86%B3Hexo%E7%BD%AE%E9%A1%B6%E9%97%AE%E9%A2%98/)通过比较 top 来实现置顶问题，非常完美，但是要注意的是原来的排序代码是：

`var posts = locals.posts.sort(config.index_generator.order_by);` 

而新的代码却是：

`posts.data = posts.data.sort(function(a, b)...`

这里多了一个 data 属性，而不是直接对 posts 数组排序，如果不写这个，会出现乱序。

> 所有文章也同样需要排序，它的修改位置在`/node_modules/hexo-generator-archive/lib/generator.js`中，搜索 sort 进行相同替换就可以了。

另外这个 top 属性也可以拿来做别的事情，比如我们不仅希望博客可以指定，而且有明显的标识，像下面：

<div align="center"><img src="http://7xktd8.com1.z0.glb.clouddn.com/博客置顶样式.png" alt="博客置顶样式" width="320"/></div>

在标题前面加上红色加粗"[置顶]"文案，那可以去修改对应主题`/layout/_partial/post/titls.ejs`文件：

```javascript
<% if (post.link){ %>
  <h1 itemprop="name">
    <a class="<%= class_name %>" href="<%- url_for(post.link) %>" target="_blank" itemprop="url">
      <%if(post.top){%><span style="font-weight:bold;color:red">[置顶]</span><% } %><%= post.title %>
    </a>
  </h1>
<% } else if (post.title){ %>
  <% if (index){ %>
    <h1 itemprop="name">
      <a class="<%= class_name %>" href="<%- url_for(post.path) %>">
        <% if(post.top){ %>
          <% if(class_name !== 'archive-article-title'){ %>
            <span style="font-weight:bold;color:red">[置顶]</span>
          <% } else { %>
            <span style="font-weight:bold;color:red;display:inline">[置顶]</span>
          <% } %>
        <% } %>
        <%= post.title %>
      </a>
    </h1>
  <% } else { %>
    <h1 class="<%= class_name %>" itemprop="name">
      <%if(post.top){%><span style="font-weight:bold;color:red">[置顶]</span><% } %><%= post.title %>
    </h1>
  <% } %>
<% } %>
```

在所有显示标题的地方判断文章是不是top的，如果是，则添加文案。

## 浏览量统计功能

这个功能对于博客写作者来说应该是蛮重要一个功能，我的视线方案使用的是[使用LeanCloud平台为Hexo博客添加文章浏览量统计组件](https://link.zhihu.com/?target=http%3A//crescentmoon.info/2014/12/11/popular-widget/)所介绍的基于 LeadCloud 的方案。详细方案原文已经阐述，但这个方案实际是作者根据自己的需求进行开发的，也就是显示在边栏上。因此建议读者看懂 JS 代码之后再修改一下，下面是我的：

```javascript
<% if(!index){ %>
    <span class="post-meta">总字数: <%= wordcount(post.content) %> | 阅读预计 <%= min2read(post.content) %> 分钟</span>
    <script>
    	addCount();
    </script>
<% } %>
```

我这里还涉及一个字数统计的功能，后面再介绍。然后修改`head.ejs`文件，在最后 header 标签之前添加：

```javascript
<script src="https://cdn1.lncld.net/static/js/av-min-1.5.0.js"></script>
<script>
    var APP_ID = 'XXXXXXXX';
    var APP_KEY = 'XXXXXXXXXXX';
    AV.init({
        appId: APP_ID,
        appKey: APP_KEY
    });
</script>

<!--page counter part-->
<script>
    function addCount() {
        var Counter=AV.Object.extend("Counter");
        url=$('.article-date').attr('href').trim();
        title = $('.article-title').text().trim();
        var query=new AV.Query(Counter);
        //use url as unique idnetfication
        query.equalTo("url",url);

        query.find({
            success: function(results){
                if(results.length>0) {
                    var counter=results[0];
                    var times=counter.get('time')+1;
                    $('.post-meta').append(" | PV: " + times);
                    counter.set('time', times);
                    counter.save();
                } else {
                    var newcounter=new Counter();
                    newcounter.set("title",title);
                    newcounter.set("url",url);
                    newcounter.set("time",1);

                    $('.post-meta').append(" | PV: 1");
                    newcounter.save().then(function (counter) {
                        // 成功
                    },function (error) {
                        // 失败
                    });
                }
            },
            error: function(error){
            }
        });
    }
</script>
```

这里 JS 文件的链接与原文的不同，原文提供的链接已经失效，这点原作者有提醒，另外初始化方法也不一样，这是升级所致，具体可见：[[JavaScript SDK 安装指南](https://leancloud.cn/docs/sdk_setup-js.html#JavaScript_SDK_安装指南)](https://leancloud.cn/docs/sdk_setup-js.html#初始化)，下面是一个简单的存取，方法可见文档：[[数据存储开发指南 · JavaScript](https://leancloud.cn/docs/leanstorage_guide-js.html#数据存储开发指南___JavaScript)]

另外还有一个需要注意的点就是在 LeanCloud 上新建 Class 的时候，权限务必选择无限制，不然在更新数据的时候会被 Forbidden。

## 博客统计功能

这个功能使用的是插件[hexo-wordcount](https://www.npmjs.com/package/hexo-wordcount)，提供以下三个功能：

1. 博客总数量统计；
2. 博客字数统计；
3. 预计阅读时间统计；

具体怎么用在官网上有明确的说明，而具体放置在什么位置，读者可以研究一下主题下面`/layout/_partial/article.ejs`文件，这个文件描述了在文章列表和详情页面文章如何显示，找一个合适的地方插入即可。

> 注：可以通过 index 这个参数判断是不是在列表页面上。

## 最后

粗略感觉 Hexo 是一个结构非常好的项目，即使我这种菜鸟也能在短时间内照葫芦画瓢进行一些个性化定制，配置 Hexo 的过程极大的激发了我学前端的欲望。