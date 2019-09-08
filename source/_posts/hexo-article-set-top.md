---
title: Hexo 文章置顶功能
catalog: true
date: 2019-09-08 11:47:05
subtitle: Hexo 文章置顶
header-img: "/img/article_header/article_header.png"
tags:
- Hexo
categories:
- Hexo
---


# Hexo 实现文章置顶

### 文章列表置顶
&emsp;Hexo 中文章指定功能无法通过配置实现，需要修改 node_modules/hexo-generator-index/lib/generator.js 文件。
需要添加的代码：
```js
posts.data = posts.data.sort(function(a, b) {
      if(a.top && b.top) {
          if(a.top == b.top) return b.date - a.date;
          else return b.top - a.top;
      }
      else if(a.top && !b.top) {
          return -1;
      }
      else if(!a.top && b.top) {
          return 1;
      }
      else return b.date - a.date;
  });
```
以下是最终的generator.js内容：
```js
'use strict';

var pagination = require('hexo-pagination');

module.exports = function(locals) {
  var config = this.config;
  var posts = locals.posts.sort(config.index_generator.order_by);

  posts.data = posts.data.sort(function(a, b) {
      if(a.top && b.top) {
          if(a.top == b.top) return b.date - a.date;
          else return b.top - a.top;
      }
      else if(a.top && !b.top) {
          return -1;
      }
      else if(!a.top && b.top) {
          return 1;
      }
      else return b.date - a.date;
  });

  var paginationDir = config.pagination_dir || 'page';
  var path = config.index_generator.path || '';

  return pagination(path, posts, {
    perPage: config.index_generator.per_page,
    layout: ['index', 'archive'],
    format: paginationDir + '/%d/',
    data: {
      __index: true
    }
  });
};
```
之后在文章中添加 top 标签可以设置置顶顺序。
```
---
title: 文章名
date: 文章发布时间
tags: 文章标签
top: 100(文章置顶)
---
```

### 显示置顶标志
&emsp;修改 themes/theme-name/layout/index.ejs 文件，在标签
```js
<a href="<%- config.root %><%- post.path %>">
        <h2 class="post-title">
```
下插入
```js
<% if (post.top){ %>
                <i class="fa fa-thumb-tack" style="color:#F4711F" ></i>
                <font color=#F4711F>置顶</font>
            <% } %>                
```
完整的 preview 为：
```js
<% page.posts.each(function(post){ %>
<div class="post-preview">

    <a href="<%- config.root %><%- post.path %>">
        <h2 class="post-title">
            <% if (post.top){ %>
                <i class="fa fa-thumb-tack" style="color:#F4711F" ></i>
                <font color=#F4711F>置顶</font>
            <% } %>                     
            <%- post.title || "Untitled" %>
        </h2>
        <h3 class="post-subtitle">
            <%- post.subtitle || "" %>
        </h3>
        <div class="post-content-preview">
            <%- truncate(strip_html(post.content), {length: 200, omission: '...'}) %>...
        </div>
    </a>
    <% if (config.home_posts_tag){%>
        <p class="post-meta" style="margin: 10px 0;">
            Posted by <%- post.author || config.author %> on
            <%= post.date.format(config.date_format) %>
        </p>
        <div class="tags">
            <% post.tags.forEach(function(tag){ %>
              <a href="<%= config.root %>tags/#<%= tag.name %>" title="<%= tag.name %>"><%= tag.name %></a>
            <% }) %>
        </div>
    <%} else {%>
        <p class="post-meta">
            Posted by <%- post.author || config.author %> on
            <%= post.date.format(config.date_format) %>
        </p>
    <%}%>

</div>
<hr>
<% }); %>
```