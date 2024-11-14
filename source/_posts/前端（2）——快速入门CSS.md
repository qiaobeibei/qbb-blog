---
title: 前端（2）——快速入门CSS
date: 2024-11-14 15:53:04
categories:
- 前端
tags: 
- CSS
typora-root-url: ./..
---



参考：

[罗大富](【3小时前端入门教程（HTML+CSS+JS）】https://www.bilibili.com/video/BV1BT4y1W7Aw?p=8&vd_source=cb95e3058c2624d2641da6f4eeb7e3a1)

[CSS 参考手册 | 菜鸟教程](https://www.runoob.com/cssref/css-reference.html#font)

[W3school](https://www.w3school.com.cn/cssref/index.asp)

------

# 1. CSS

CSS全名是`层叠样式表`，中文名`层叠样式表`。用于定义网页样式和布局的样式表语言。
通过 CSS，你可以指定页面中各个元素的颜色、字体、大小、间距、边框、背景等样式，从而实现更精确的页面
设计。

------

CSS通常由选择器、属性和属性值组成，多个规则可以组合在一起，以便同时应用多个样式

```
选择器{
	属性1: 属性值1;
	属性2: 属性值2;
}
```

1. 选择器的声明中可以写无数条属性
2. 声明的每一行属性，都需要以英文分号结尾
3. 声明中的所有属性和值都是以键值对这种形式出现的

选择器就是选择要应用样式的HTML元素或者标签，它可以选择所有元素或特定元素或特定类或特定id。

示例：

```css
/* p 标签选择器 */
p{
	color: bulue;
	font-size: 16px;
}
```

## 1.1 CSS导入方式

下面是三种常见的 CSS 导入方式：

1. 内联样式(Inline styles)：直接将样式放在标签中，标签有`style`属性，可以直接定义`style`格式
2. 内部样式表(Internal Stylesheet)：放在html标签头部
3. 外部样式表(External Stylesheet)：单独放在CSS文件中，允许多个网页使用同一个样式

三种导入方式的优先级：内联样式>内部样式表>外部样式表

### 1.1.1 内部样式表

内部样式表就是将CSS样式放在HTML文档的头部，head标签中

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS导入方式</title>

    <style>
        p{
            color: blue;
            font-size: 16px;
        }
    </style>
</head>
<body>
    <p>应用CSS样式的文本</p>
</body>
</html>
```

对于所有的`p`标签，它的字体颜色均为蓝色，大小均为16px

![image-20241114160450844](/images/$%7Bfiilename%7D/image-20241114160450844.png)

### 1.1.2 内联样式

```html
<h1 style="color: red">这是一级标题，内联样式</h1>
```

### 1.1.3 外部样式

须在头部添加`link`标签，`href`属性中是外部样式文件的路径地址：

```html
<link rel="stylesheet" href="./style.css">
```

其中，style.css文件内容如下：

```css
h3{
    color:brown;
}
```

------

综合测试：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS导入方式</title>
    <link rel="stylesheet" href="./style.css">

    <style>
        p{
            color: blue;
            font-size: 16px;
        }

        h2{
            color: rgb(0, 238, 255);
        }
    </style>
</head>
<body>
    <p>应用CSS样式的文本</p>
    <h1 style="color: red">这是一级标题，内联样式</h1>
    <h2>这是二级标题，内部样式</h2>
    <h3>这是三级标题，外部样式</h3>
</body>
</html>
```

![image-20241114161510652](/images/$%7Bfiilename%7D/image-20241114161510652.png)

## 1.2 选择器

选择器是 CSS中 的关键部分，它允许你针对特定元素或一组元素定义样式

- 元素选择器
- 类选择器
- ID 选择器
- 通用选择器
- 子元素选择器
- (包含选择器)后代选择器
- 并集选择器(兄弟选择器)
- 伪类选择器

我们上面对标签格式修改使用的是元素选择器。

### 1.2.1 类选择器

我们可以在style标签对中使用元素选择器对标签对进行格式修改，使用`.`

```html
    <style> 
        h2{
            color:aqua;
        }

        h3{
            background-color: yellowgreen;
        }
    </style>
```

但如果我们只想将`h3`标签对中`class`属性值为`highlight`的标签才修改，而其他的不修改，应该怎么做：

我们只需将`h3`改为.`highlight`，就能将对应`clss`属性值相同的标签修改为相同的格式：

```html
    <style> 
        .highlight{
            background-color: yellowgreen;
        }

    </style>

    <h3 class="highlight">类选择器示例</h3>
    <h3">另一个类选择器示例</h3>
```

![image-20241114162652476](/images/$%7Bfiilename%7D/image-20241114162652476.png)

### 1.2.2 id选择器

我们也可以根据使用id选择器，对特定id的标签格式进行修改，使用`#`

```html
    <style> 
        #header{
            color: blueviolet;
            font-size: 33px;
        }
    </style>
    
    <h4 id="header">id选择器示例</h4>
```

![image-20241114162926386](/images/$%7Bfiilename%7D/image-20241114162926386.png)

### 1.2.3 通用选择器

通用选择器就是选择所有元素，使用`*`

```html
        *{
            font-family: 'KaiTi';
        }
```

这里将所有元素的字体设为楷体。

![image-20241114181329293](/images/$%7Bfiilename%7D/image-20241114181329293.png)

### 1.2.4 子元素选择器

选择位于父元素内部的子元素，就是一个大标签嵌套一个小标签，子元素选择器用于选择小标签。

```html
    <div class="father">
        <p class="son">这是一个子元素选择器示例</p>
    </div>
```

比如，`div`就是一个大标签，嵌套一个子标签`p`，子元素选择器是通过大标签和子标签的类名来定义的：

```html
        /*子元素选择器*/
        .father > .son{
            color:rgb(255, 145, 0)
        }
```

这里，更换类名为father的父标签下类名为son的子标签的颜色

![image-20241114181341153](/images/$%7Bfiilename%7D/image-20241114181341153.png)

### 1.2.5 后代选择器

如果子标签还嵌套一个更小的子标签呢？比如：

```html
<div class="father">
        <p class="son">这是一个子元素选择器示例</p>
        <div>
            <p class="grandson">这是一个后代选择器示例</p>
        </div>
    </div>
```

那么子元素选择器对类名为`grandson`的嵌套子标签就没作用了，因为**它不是father的直系子标签**。

这时，必须使用后代选择器：

```
        /*后代选择器*/
        .father p{
            color:rgb(81, 126, 229);
            font-size: larger;
        }
```

该选择器会将类名为`father`下的所有子标签`p`的颜色都变为设定颜色。

![image-20241114181418801](/images/$%7Bfiilename%7D/image-20241114181418801.png)

第二行字体的颜色确实变了，但是第一行为什么不变，第一行也是类名为`father`下的子标签呀。

其实这涉及到优先级的问题，id>类>标签名，因为子元素选择器相当于类选择器（通过.类名构造），所以肯定比p标签选择器优先大，所以第一行不改变。

### 1.2.6 兄弟选择器

```
    /*相邻兄弟选择器*/
    h3 + p{
        background-color: red;
    }
        
    <p>普通的p标签</p>
    <h3>相邻兄弟选择器示例</h3>
    <p>另一个普通的p标签</p>
```

该选择器会将紧跟在`h3`后面第一个的`p`标签格式进行修改：

![image-20241114181924686](/images/$%7Bfiilename%7D/image-20241114181924686.png)

### 1.2.7 伪类选择器

用于选择HTML文档的元素的特定状态或位置，不仅仅是元素的属性。它以冒号开头，用于构造用户交互结构，比如，鼠标悬停在一个元素上，元素的格式或者状态可以被改变，这种方式可以通过伪类选择器实现。

```
/*伪类选择器*/
#element:hover{
    background-color: aquamarine;
}

<h3 id="element">伪类选择器示例</h3>
```

![image-20241114182349631](/images/$%7Bfiilename%7D/image-20241114182349631.png)

一开始伪类选择器文本无背景颜色，当鼠标放在上面之后，会有背景颜色生成：

![image-20241114182444721](/images/$%7Bfiilename%7D/image-20241114182444721.png)

也可以通过伪类选择器，选择父元素中的第一个元素，

------

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS选择器</title>

    <style> 
        /* 元素选择器*/
        h2{
            color:aqua;
        }
        /* 类选择器*/
        .highlight{
            background-color: yellowgreen;
        }
        /* id选择器*/
        #header{
            color: blueviolet;
            font-size: 33px;
        }
        /* 通用选择器*/
        *{
            font-family: 'KaiTi';
        }
        /*子元素选择器*/
        .father > .son{
            color:rgb(255, 145, 0)
        }
        /*后代选择器*/
        .father p{
            color:rgb(81, 126, 229);
            font-size: larger;
        }
        /*相邻兄弟选择器*/
        h3 + p{
            background-color: red;
        }
        /*伪类选择器*/
        #element:hover{
            background-color: aquamarine;
        }
        /*
        选中第一个子元素 :first-child
        选择第n个子元素  :nth-child()
        链接的状态       :active
        */
    </style>
</head>
<body>
    <h1>不同类型的 CSS 选择器</h1>
    <h2>元素选择器示例</h2>

    <h3 class="highlight">类选择器示例</h3>
    <h3">另一个类选择器示例</h3>

    <h4 id="header">id选择器示例</h4>

    <div class="father">
        <p class="son">这是一个子元素选择器示例</p>
        <div>
            <p class="grandson">这是一个后代选择器示例</p>
        </div>
    </div>

    <p>普通的p标签</p>
    <h3>相邻兄弟选择器示例</h3>
    <p>另一个普通的p标签</p>

    <h3 id="element">伪类选择器示例</h3>

</body>
</html>
```



## 1.3 CSS常用属性

现在我们可以通过选择器指定元素并对其进行修改，现在可以学习属性，对选择的元素添加样式。CSS的属性有很多，可以参考 [W3school]([CSS 参考手册](https://www.w3school.com.cn/cssref/index.asp))和[CSS 参考手册 | 菜鸟教程](https://www.runoob.com/cssref/css-reference.html)我们这里只做简单了解：

行内元素无法添加高度和宽度，只有块级别可以自定义：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS常用属性</title>
    <style>
        .block{
            background-color: aqua;
            width: 150px;
            height: 150px;
        }
        .inline{
            color: red;
        }
        .inline-block{
            width: 300px;
            height: 200px;
        }
    </style>

</head>
<body>
    <h1 style="font: bolder 50px 'KaiTi';">这是一个font复合属性</h1>
    <p style="line-height: 40px;">这是一个长文本这是一个长文本这是一个长文本这是一个长文本这是一个长文本这是一个长文本这是一个长文本这是一个长文本这是一个长文本</p>

    <div class="block">块级元素</div>
    <span class="inline">行级元素</span> 
    <img src="./border.png" alt="" class="inline-block">
</body>
</html>
```

![image-20241114184740364](/images/$%7Bfiilename%7D/image-20241114184740364.png)

## 1.4 盒子模型

盒子模型是CSS中常用于布局的基本概念，描述了文档中每个元素都可以被看作一个矩形的盒子，这个盒子包含了内容`content`，高度`height`，宽度`width`，内边距`padding`，文本边框`border`，外边距`margin`，如下图所示

![image-20241114184811814](/images/$%7Bfiilename%7D/image-20241114184811814.png)

|     属性名      |                             说明                             |
| :-------------: | :----------------------------------------------------------: |
|  内容(Content)  |             盒子包含的实际内容，比如文本、图片等             |
| 内边距(Padding) | 围绕在内容的内部，是内容与边框之间的空间。可以使用`padding`属性来设置。 |
|  边框(Border)   | 围绕在内边距的夕部，是盒子的边界。可以使用`border、属性来设置。 |
| 外边距(Margin)  | 围绕在边框的外部，是盒子与其他元素之间的空间，可以使用margin属性来设置 |

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS盒子模型</title>
    <style>
        .border-demo{
            background-color: rgb(166, 222, 53);
            width: 300px;
            height: 200px;
            border-style: solid;
            border-width: 10px 5px 20px;
            border-color: blueviolet;
        }
        .demo{
            background-color: aqua;
            display: inline-block;
            border:5px solid rgb(11, 11, 11);
            padding: 50px;
            margin: 40px;
        }
    </style>
</head>
<body>
    <div class="demo">这是一个demo</div> <br><br>
    <div class="border-demo">这是边框demo</div>
</body>
</html>
```

![image-20241114190143949](/images/$%7Bfiilename%7D/image-20241114190143949.png)

**我们可以修改盒子模型，来改变元素在页面中的位置和大小**

## 1.5 浮动

在学习浮动之前，先了解传统的网页布局方式。

网页布局方式有以下五种：

- 标准流(普通流、文档流)：网页按照元素的书写顺序依次排列
- 浮动
- 定位
- Flexbox`和`Grid(自适应布局)

`标准流`是由块级元素和行内元素按照默认规的方式来排列，块级就是占一行，行内元素一行放好多个元素。

------

**浮动**：元素脱离文档流，根据开发者的意愿漂浮到网的任意方向。

`浮动`属性用于创建浮动框，将其移动到一边，直到左边缘或右边缘触及包含块或另一个浮动框的边缘，这样即可使得元素进行浮动。

**语法:**

```
选择器{
	float: left/right/none
}
```

注意：**浮动是相对于父元素浮动，只会在父元素的内部移动**

**浮动的三大特性**：

- 脱标：脱离标准流。
- 一行显示，顶部对齐
- 具备行内块元素特性

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS浮动</title>
    <style>
        .father{
            background-color: aquamarine;
            height: 150px;
            border: 3px solid black;
            /* overflow: hidden; */
        }
        
        .left-son{
            width:100px;
            height: 100px;
            background-color: red;
            float: left;
        }

        .right-son{
            width:100px;
            height: 100px;
            background-color: rgb(209, 14, 141);
            float:right;
        }
    </style>
</head>
<body>
    <div class="father">
        <div class="left-son">左浮动</div>
        <div class="right-son">右浮动</div>
    </div>
    
</body>
</html>
```

![image-20241114191645767](/images/$%7Bfiilename%7D/image-20241114191645767.png)

如果父元素坍塌，我们可以在父元素构造器中加`overflow: hidden;`来清除坍塌。

## 1.6 定位

定位布局可以精准定位，但缺乏灵活性

**定位方式：**

- 相对定位：相对于元素在文档流中的正常位置进行定位。
- 绝对定位：相对于其最近的已定位祖先元素进行定位，不占据文档流。
- 固定定位：相对于浏览器窗口进行定位。不占据文档流，固定在屏幕上的位置，**不随滚动而移动。**

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>CSS定位</title>
    <style>
        .box1{
            height: 350px;
            background-color: aquamarine;
        }
        .box-normal{
            height: 100px;
            width: 100px;
            background-color: rgb(83, 28, 232);
        }
        .box-relative{
            height: 100px;
            width: 100px;
            background-color: rgb(234, 20, 20);
            position: relative;
            left: 130px;
            top: 40px;
        }
        .box2{
            height: 350px;
            background-color: rgb(153, 219, 12);
        }
        .box-absolute{
            height: 100px;
            width: 100px;
            background-color: rgb(234, 20, 20);
            position: absolute;
            left: 130px;
            top: 40px;
        }
        .box3{
            height: 350px;
            background-color: rgb(153, 219, 12);
        }
        .box-fixed{
            height: 100px;
            width: 100px;
            background-color: rgb(11, 184, 124);
            position: fixed;
            left: 430px;
            top: 40px;
        }
    </style>
</head>
<body>
    <h1>相对定位</h1>
    <div class="box1">
        <div class="box-normal"></div>
        <div class="box-relative"></div>
        <div class="box-normal"></div>
    </div>
    <h1>绝对定位</h1>
    <div class="box2">
        <div class="box-normal"></div>
        <div class="box-absolute"></div>
        <div class="box-normal"></div>
    </div>
    <h1>固定定位</h1>
    <div class="box3">
        <div class="box-normal"></div>
        <div class="box-fixed"></div>
        <div class="box-normal"></div>
    </div>
</body>
</html>
```

![image-20241114192750353](/images/$%7Bfiilename%7D/image-20241114192750353.png)
