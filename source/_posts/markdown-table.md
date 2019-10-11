---
title: markdown 中表格单元格合并的问题
subtitle: 表格单元格合并
catalog: true
header-img: /img/article_header/article_header.png
tags:
  - markdown
categories:
  - markdown
date: 2019-10-11 15:29:44
---


# markdown 中表格单元格合并的问题
我们知道，在markdown中实现表格是很简单的事情：
```
|项目1|项目2|项目3|
|---|---|---|
|a1|a1|a3|
|b1|b2|b3|
|b1|c2|c3|
```
效果如下：
|项目1|项目2|项目3|
|---|---|---|
|a1|a1|a3|
|b1|b2|b3|
|b1|c2|c3|

但是如果我们要将上表中的相同文字单元格进行合并，如a1与a1横向合并，b1与b1纵向合并，单纯的markdown语法是不能实现的。由于markdown支持html，我们可以通过表格在html中的table进行实现。
这个网站可以帮助我们自动生成需要的表格的html代码

http://www.tablesgenerator.com/

需要注意的一点是，在markdown中使用html代码来实现表格的效果，需要在表格的外面套上
```
<escape></escape>
```
（转义），防止markdown直接将代码中的行进行转义成回车，不然会出现表格前空了一大块空白。

```
<escape>
<table>
  <tr>
    <th>项目1</th>
    <th>项目2</th>
    <th>项目3</th>
  </tr>
  <tr>
    <td>a1</td>
    <td colspan="2">a2</td>
  </tr>
  <tr>
    <td rowspan="2">b1</td>
    <td>b2</td>
    <td>b3</td>
  </tr>
  <tr>
    <td>c2</td>
    <td>c3</td>
  </tr>
</table>
</escape>
```
效果：
<escape>
<table>
  <tr>
    <th>项目1</th>
    <th>项目2</th>
    <th>项目3</th>
  </tr>
  <tr>
    <td>a1</td>
    <td colspan="2">a2</td>
  </tr>
  <tr>
    <td rowspan="2">b1</td>
    <td>b2</td>
    <td>b3</td>
  </tr>
  <tr>
    <td>c2</td>
    <td>c3</td>
  </tr>
</table>
</escape>

但同时，引入html会使得markdown的易读易写的特性降低。除非必要，还是推荐使用markdown本身的表格语法。