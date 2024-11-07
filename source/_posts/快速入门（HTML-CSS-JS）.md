---
title: 快速入门（HTML+CSS+JS）
date: 2024-11-06 11:11:11
categories:
- 前端
tags: 
- CSS
- JS
- HTML
typora-root-url: ./..
---



# 1. HTML

我使用的是vs code，在使用之前，先安装以下几个插件：

- Auto Rename Tage
- HTML CSS Support
- Live Server

## 1.1 HTML标签

> `HTML`全称是`Hypertext Markup Language(超文本标记语言`

HTML通过一系列的`标签(也称为元素)`来定义文本、图像、链接等等。HTML,标签是由尖括号包围的关键
字。

标签通常成对出现，包括开始标签和结束标签(也称为双标签)，内容位于这两个标签之间，例如:

```
<p>这是一个段落</p>
<h1>这是一个标题</h1>
<a href="#">这是一个超链接</a>
```

除了双标签，也存在单标签，例如：

```
<input type="text">
<br>
<hr>
```

区别：单标签用于没有内容的元素，双标签用于有内容的元素

## 1.2 HTML文件结构

```html
<!-- 这里放置文档的元信息 -->
<!DOCTYPE html>
<html>
<head>
    <!-- 这里放置文档的元信息 -->
    <title>文档标题</title>
    <meta charset="UTF-8">
    <!-- 连接外部样式表或脚本文件等 -->
    <link rel="stylesheet" type="text/css" href="styles.css">
    <script src="script.js"></script>
</head>
<body>
    <!-- 这里放置页面内容 -->
    <h1>这是一个标题</h1>
    <p>这是一个段落。</p>
    <a href="https://www.example.com">这是一个链接</a>
    <!-- 其他内容 -->
</body>
</html>
```

一个HTML文档通常由以下几个结构组成：

1. 告诉浏览器这是一个HTML文件

```
<!DOCTYPE html>
```

2. html标签对：是HTML文档的根元素，同样也是HTML文档的起始点和文档的最外层容器，包含了整个文档的结构。

```
<html>
...
...
...
</html>
```

3. head标签对：表示文档的头部，包含了一些文件的原信息。比如文件的标题，文件的编码格式，外部的样式表（CSS和JS文件）

```
<head>
    <!-- 这里放置文档的元信息 -->
    <title>文档标题</title>
    <meta charset="UTF-8">
    <!-- 连接外部样式表或脚本文件等 -->
    <link rel="stylesheet" type="text/css" href="styles.css">
    <script src="script.js"></script>
</head>
```

4. body标签对：包含了实际显示在浏览器中页面的内容，比如文本、链接、图形等等

```
<body>
    <!-- 这里放置页面内容 -->
    <h1>这是一个标题</h1>
    <p>这是一个段落。</p>
    <a href="https://www.example.com">这是一个链接</a>
    <!-- 其他内容 -->
</body>
```

### 1.2.1 vs中快速生成HTML结构

在 vscode 中输入`!`，若出现下图现象

![1730862559771(1)](/images/$%7Bfiilename%7D/1730862559771(1).jpg)再点击键盘`Tab`键，编译器会自动帮我们生成HTML结构

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Document</title>
</head>
<body>
    
</body>
</html>
```

编译器代码编辑框内鼠标右键，点击Open with Live Server

![image-20241106111752239](/images/$%7Bfiilename%7D/image-20241106111752239.png)

若没有浏览器界面弹出，检查是否安装Live Server插件，或者手动在浏览器页面输入：

```
127.0.0.1:5500
```

![image-20241106111858405](/images/$%7Bfiilename%7D/image-20241106111858405.png)

点击example.html文件即可打开我们刚才创建的html文件

## 1.3 HTML常用文本标签

未完待续。。。
