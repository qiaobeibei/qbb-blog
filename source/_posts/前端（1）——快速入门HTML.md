---
title: 前端（1）——快速入门HTML
date: 2024-11-06 11:11:11
categories:
- 前端
tags: 
- HTML
typora-root-url: ./..
---



参考：

[W3school](https://www.w3school.com.cn/html/html_form_attributes.asp)

[罗大富](【3小时前端入门教程（HTML+CSS+JS）】https://www.bilibili.com/video/BV1BT4y1W7Aw?p=10&vd_source=cb95e3058c2624d2641da6f4eeb7e3a1)

------

# 1. HTML

我使用的是vs code，在使用之前，先安装以下几个插件：

- Auto Rename Tage
- HTML CSS Support
- Live Server

## 1.1 HTML标签

> `HTML`全称是`Hypertext Markup Language(超文本标记语言)`

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

### 1.3.1 标题标签

html中有六个标题标签，用h1~h6表示，如果我们在body内输入h1，编译器会自动给我们生成一个h1的标签

![1731550637922](/images/$%7Bfiilename%7D/1731550637922.jpg)

![image-20241114101820328](/images/$%7Bfiilename%7D/image-20241114101820328.png)

我们可以在标题标签之间写出我们想要写的内容，比如“标签”

```html
<body>
    <h1>标签</h1>
</body>
```

浏览器的html页面中会出现：

![image-20241114102004909](/images/$%7Bfiilename%7D/image-20241114102004909.png)

同理，也可以使用h2、h3...h6展示一级标题~六级标题

### 1.3.2 段落标签

段落标签可以在编译器中输入p，然后点击Tab，在标签对中写入自己想写的内容：

```html
<body>
    <h1>标签</h1>
    <p>段落标签</p>
</body>
```

浏览器的内容也会相对发生变化：

![image-20241114102333528](/images/$%7Bfiilename%7D/image-20241114102333528.png)

我们也可以将段落标签中内容的格式进行修改，比如加粗的标签对是p

```html
<body>
    <h1>标签</h1>
    <p>段落标签</p>
    <p>更改文本样式：<b>字体加粗</b></p>
</body>
```

![image-20241114102527117](/images/$%7Bfiilename%7D/image-20241114102527117.png)

同理，也可以对字体使用斜体、下划线、删除线：

```html
<body>
    <h1>标签</h1>
    <p>段落标签</p>
    <p>更改文本样式：<b>字体加粗</b></p>
    <p>更改文本样式：<i>字体斜体</i></p>
    <p>更改文本样式：<u>字体下划线</u></p>
    <p>更改文本样式：<s>字体删除线</s></p>
</body>
```

![image-20241114102719226](/images/$%7Bfiilename%7D/image-20241114102719226.png)

我们也可以通过CSS修改字体的大小，后面学习CSS的时候详细介绍

### 1.3.3 其他标签

#### 1.3.3.1 无序列表

`ul`标签代表无序列表，`ul`标签中包含着`li`标签，有几个无序列表就加几个`li`标签，如下：

```html
    <ul>
        <li>无序列表1</li>
        <li>无序列表2</li>
        <li>无序列表3</li>
    </ul>
```

![image-20241114103017007](/images/$%7Bfiilename%7D/image-20241114103017007.png)

#### 1.3.3.2 有序列表

有序列表其实就是将无序列表的标签对`ul`修改为`ol`

```html
    <ol>
        <li>有序列表1</li>
        <li>有序列表2</li>
        <li>有序列表3</li>
    </ol>
```

![image-20241114103144953](/images/$%7Bfiilename%7D/image-20241114103144953.png)

### 1.3.4 表格标签

最外层使用`table`标签对作为表格的根元素；表格中行标签使用`tr`来表示；列标签使用`td`表示，`td`一般被`tr`包含其中，理解为一行中有很多列；列标题使用`th`表示

```html
    <table>
        <tr>
            <th>列标题1</th>
            <th>列标题2</th>
        </tr>
        <tr>
            <td>元素1</td>
            <td>元素2</td>
            <td>元素3</td>
        </tr>
    </table>
```

![image-20241114103537738](/images/$%7Bfiilename%7D/image-20241114103537738.png)

对`talbe`标签进行修改，变为`table border='x'`可对表格增加边框，x是边框的宽度

```html
    <table border="1">
        <tr>
            <th>列标题1</th>
            <th>列标题2</th>
            <th>列标题3</th>
        </tr>
        <tr>
            <td>元素1</td>
            <td>元素2</td>
            <td>元素3</td>
        </tr>
        <tr>
            <td>元素1</td>
            <td>元素2</td>
            <td>元素3</td>
        </tr>
        <tr>
            <td>元素1</td>
            <td>元素2</td>
            <td>元素3</td>
        </tr>
    </table>
```

![image-20241114103752605](/images/$%7Bfiilename%7D/image-20241114103752605.png)

## 1.4 HTML标签属性

每个标签都有属性，比如，在表格标签中，我们可对`table`修改增加属性`border`来修改表格边框的宽度，当然也可以通过其他属性再次修改。

- 基本语法：

```
<开始标签 属性名="属性值">
```

- 每个HTML元素可以具有不同属性

```html
    <p id="describe" class="section">段落标签</p>
    <a href="http://baidu.com">超链接</a>
```

超链接可以通过标签对`a`来使用，属性`href`用于表示链接的地址。

- 属性名称不区分大小写，但属性值对大小写敏感

```html
    <img src="example.jpg" alt="">
    <img SRC="example.jpg" alt="">
    <img src="EXAMPLE.JPG" alt="">
    <!--前两个相同，第三个与前两个不同-->
```

### 1.4.1 适用于大多数HTML元素的属性

| 属性  |                       描述                       |
| :---: | :----------------------------------------------: |
| class | 为HTML元素顶一个或多个类名（类名从样式文件引入） |
|  id   |                 定义元素唯一的id                 |
| style |                规定元素的行内样式                |

### 1.4.2 超链接

上面说过，我们可以使用`a`作为超链接的标签对，属性`href`用于表示链接的地址，其实还有属性`target`表示链接的打开方式：

![image-20241114105217897](/images/$%7Bfiilename%7D/image-20241114105217897.png)

链接的打开方式有四种，介绍如下

- `_self`: Load the URL into the same browsing context as the current one. This is the default behavior.
- `_blank`: Load the URL into a new browsing context. This is usually a tab, but users can configure browsers to use new windows instead.
- `_parent`: Load the URL into the parent browsing context of the current one. If there is no parent, this behaves the same way as `_self`.
- `_top`: Load the URL into the top-level browsing context (that is, the "highest" browsing context that is an ancestor of the current one, and has no parent). If there is no parent, this behaves the same way as `_self`.

我们举个例子，通过一个新窗口打开链接：

```html
<body>
    <a href="https://www.aichitudou.cn/">我的博客</a>
    <a href="https://www.aichitudou.cn/" target="_blank">我的博客</a>
</body>
```

如果不使用`target`标签，默认使用`self`在相同窗口内打开而不创建新的窗口

![image-20241114105606961](/images/$%7Bfiilename%7D/image-20241114105606961.png)

### 1.4.3 换行标签

如果想将两个超链接放在不同行，可以使用换行标签`br`

```html
<body>
    <a href="https://www.aichitudou.cn/">我的博客</a>
    <br>
    <a href="https://www.aichitudou.cn/" target="_blank">我的博客</a>
</body>
```

![image-20241114105552304](/images/$%7Bfiilename%7D/image-20241114105552304.png)

### 1.4.4 水平分割线

或者也可以使用`hr`创建水平分割线

```html
<body>
    <a href="https://www.aichitudou.cn/">我的博客</a>
    <br>
    <hr>
    <a href="https://www.aichitudou.cn/" target="_blank">我的博客</a>
</body>
```

![image-20241114105659909](/images/$%7Bfiilename%7D/image-20241114105659909.png)

### 1.4.5 图片标签

输入`img`点击`tab`会自动生成`img`标签

```html
<img src="" alt="">
```

`src`定义了图片的路径或者图片的来源（比如网址链接）；

`alt`定义了图形的文本，如果图形无法加载，那么就会显示图片的文本

```html
    <img src="./border.png" alt="">
    <br>
    <img src="" alt="图片描述">
```

![image-20241114110131404](/images/$%7Bfiilename%7D/image-20241114110131404.png)

第二张图片无法显示时，会展示我们在`alt`中写入的文本

此外，还可以通过`width`和`height`属性修改图片的高度和宽度

```html
<img src="./border.png" alt="" width="100" height="100">
```

![image-20241114110355760](/images/$%7Bfiilename%7D/image-20241114110355760.png)

## 1.5 HTML区块

**a. 块元素**

块级元素通常用于组织和布局页面的主要结构和内容，例如段落、标题、列表、表格等。它们用于创建页面的主要部分，将内容分隔成逻辑块。

- 块级元素通常会**从新行开始**，并占据整行的宽度，因此它们会在页面上呈现为一块独立的内容块。
- 可以包含其他块级元素和行内元素。
- 常见的块级元素包括`<div>，<p>，<h1>`到`h6`、`ul`、`ol`、`li`、`table`、`form`等

**b. 行内元素**

行内元素通常用于添加文本样式或为文本中的一部分应用样式。它们可以在文本中插入小的元素，例如超链接、强调文本等。

- 行内元素通常在**同一行内呈现**，不会独占一行
- 它们只占据其内容所需的宽度，而不是整行的宽度
- 行内元素不能包含块级元素，但可以包含其他行内元素
- 常见的行内元素包括`span`、`a`、`b`、`em`、`img`、`br`等

### 1.5.1 div标签

`div`标签**没有语义**，仅用于组织内存。我们可以使用`div`标签创建网页的不同布局，比如导航栏、页眉、侧边栏等等。

```html
    <div class="nav-bar">
        <a href="#">链接1</a>
        <a href="#">链接2</a>
        <a href="#">链接3</a>
        <a href="#">链接4</a>
        <a href="#">链接5</a>
        <a href="#">链接6</a>
    </div>

    <div class="content">
        <h1>标题1</h1>
        <p>文章内容</p>
        <p>文章内容</p>
        <p>文章内容</p>
        <p>文章内容</p>
        <p>文章内容</p>
    </div>
```

我们创建了两个`div`区块，其中第一个表示导航栏，用于展示超链接；第二个表示内容，用于展示文章内容。不同区块需要使用`div`分割开来。

![image-20241114111641847](/images/$%7Bfiilename%7D/image-20241114111641847.png)

### 1.5.2 span标签

`span`标签相当于没有特殊元素的`a`标签和`img`标签，作用就是把一部分文本包装起来，以对他们使用样式、CSS或者JS操作

```
    <span>第1个span标签</span>
    <span>第2个span标签</span>
    <span>第3个span标签</span>
    <span>第4个span标签</span>
    <br>
    <span>链接点击这里 <a href="https://www.aichitudou.cn/">个人博客</a></span>
```

![image-20241114112044081](/images/$%7Bfiilename%7D/image-20241114112044081.png)

## 1.6 表单

HTML表单用于收集用户输入的数据。表单通常包含文本框、按钮、复选框、单选按钮等元素，用户填写完表单后，可以通过按钮提交数据到服务器进行处理。

### 1.6.1 form标签

表单的容器，用于包含一个固定区域内的表单，表单构成的其他标签均要被form标签包含。

### 1.6.2 input标签

用于不同类型的输入框。`type="text"` 创建文本框，`type="email"` 用于电子邮件输入，`type="radio"` 用于单选按钮，`type="checkbox"` 用于复选框，`type="submit"` 用于提交按钮。除此之外，`input`还有很多属性。

比如，我们使用一个`text`创建文本框

```
    <form action="">
        <input type="text">
    </form>
```

![image-20241114112913628](./images/$%7Bfiilename%7D/image-20241114112913628.png)

我们可以在文本框中输入内容，浏览器的搜索框也是`text`文本框。

我们也可以通过`placeholder`属性在文本框中默认写入一些文本

```
    <form action="">
        <input type="text" placeholder="请输入内容">
    </form>
```

![image-20241114113100176](./images/$%7Bfiilename%7D/image-20241114113100176.png)

`value`标签和`placeholder`相似，均会在文本框中默认生成一个文本：

```
<input type="text" value="请输入内容">
```

但是，前者不占据文本的实际位置，只是默认提示；后者相当于你写入了一个本文串。

![image-20241114113233498](./images/$%7Bfiilename%7D/image-20241114113233498.png)

如果我们想要让输入内容不可见呢？比如我们要输入密码。我们可以使用`input`元素的`password`属性，这样就可以让密码的输入不可见：

![image-20241114150412210](/images/$%7Bfiilename%7D/image-20241114150412210.png)

我们可以使用`span`标签为一个文本框定义一个名称：

```html
    <form action="">
        <span>用户名：</span>
        <input type="text" placeholder="请输入内容">
        <br><br>
        <span>密码：</span>
        <input type="text" value="请输入内容">
    </form>
```

![image-20241114121517570](/images/$%7Bfiilename%7D/image-20241114121517570.png)

有的网址注册时需要选择性别，我们可以使用`radio`属性将其变为一个单选框：

```html
        <label>性别</label>
        <input type="radio" name="gender">男
        <input type="radio" name="gender" >女
        <input type="radio" name="gender" >私密
```

同时我们需要将性别的三个选项的`name`属性的值设为相同，这是为了保证只能选择三个中的其中一个，如果不设置，那就是多选：

![image-20241114121909513](/images/$%7Bfiilename%7D/image-20241114121909513.png)

虽然我们可以通过`radio`并且不设置`name`属性可以进行多选，但总归不方便，有没有专门用于多选的属性呢？有的，使用`checkbox`便可以实现：

```
        <label for="hobby">爱好</label>
        <input type="checkbox" name="hobby">篮球
        <input type="checkbox" name="hobby" >唱歌
        <input type="checkbox" name="hobby" >rap
```

![image-20241114151430635](/images/$%7Bfiilename%7D/image-20241114151430635.png)

最后，如果我们想要把这些信息提交给我们的服务器呢？我们需要配合`submit`属性进行工作：

```
<input type="submit" value="上传">
```

提交到的地方是`form`标签中的`action`属性对应的位置，也就是后端提交给我们的API

![image-20241114152035553](/images/$%7Bfiilename%7D/image-20241114152035553.png)

### 1.6.3 label标签

`label`标签一般用于为表单控件提供描述。当我们使用`label`标签的时候，编译器自动给我们添加一个`for`属性，for属性用于将`label`标签绑定到`input`元素，一般`for`属性的值和`label`元素中`id`属性的值相同。因为`id`一般是唯一的，所以想要用一个`for`绑定多个`input`元素时，需要将`for`属性去掉。

```html
        <label for="username">同意协议</label>
        <input type="text" placeholder="请输入内容" id="username">
```

### 1.6.4 示例

```html
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>示例表单</title>
</head>
<body>

    <h1>用户信息表单</h1>
    
    <form action="/submit" method="POST">
        <label for="name">姓名:</label><br>
        <input type="text" id="name" name="name" required><br><br>
        
        <label for="email">电子邮件:</label><br>
        <input type="email" id="email" name="email" required><br><br>
        
        <label for="gender">性别:</label><br>
        <input type="radio" id="male" name="gender" value="male" required>
        <label for="male">男</label><br>
        <input type="radio" id="female" name="gender" value="female">
        <label for="female">女</label><br><br>
        
        <label for="hobbies">兴趣爱好:</label><br>
        <input type="checkbox" id="reading" name="hobbies" value="reading">
        <label for="reading">阅读</label><br>
        <input type="checkbox" id="traveling" name="hobbies" value="traveling">
        <label for="traveling">旅行</label><br>
        <input type="checkbox" id="sports" name="hobbies" value="sports">
        <label for="sports">运动</label><br><br>
        
        <label for="message">留言:</label><br>
        <textarea id="message" name="message" rows="4" cols="50"></textarea><br><br>
        
        <input type="submit" value="提交">
    </form>

</body>
</html>
```

![image-20241114152222002](/images/$%7Bfiilename%7D/image-20241114152222002.png)
