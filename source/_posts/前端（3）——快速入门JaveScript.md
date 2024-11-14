---
title: 前端（3）——快速入门JaveScript
date: 2024-11-14 19:56:22
categories:
- 前端
tags: 
- JaveScript
typora-root-url: ./..
---

参考：

[罗大富](【3小时前端入门教程（HTML+CSS+JS）】https://www.bilibili.com/video/BV1BT4y1W7Aw?p=8&vd_source=cb95e3058c2624d2641da6f4eeb7e3a1)

[JavaScript 教程 | 菜鸟教程](https://www.runoob.com/js/js-tutorial.html)

[JavaScript 教程](https://www.w3school.com.cn/js/index.asp)

------

# 1. JaveScript

> `JavaScript` 简称 JS

- `JavaScript` 是一种轻量级、解释型、面向对象的脚本语言。它主要被设计用于在网页上**实现动态效果**，增加用户与网页的交互性。
- 作为一种**客户端脚本语言**，JavaScript 可以直接嵌入 HTML，并在浏览器中执行。
- 与 HTML和 CSS 不同，JavaScript 使得网页不再是静态的，而是可以根据用户的操作动态变化的。

***JavaScript 的作用：***

- 客户端脚本：用于在用户浏览器中执行，实现动态效果和用户交互，
- 与 HTML和 CSS 协同工作，使得网页具有更强的交互性和动态性。网页开发
- 使用 Node.js，JavaScript 也可以在服务器端运行，实现服务器端应用的开发后端开发:

## 1.1 JS的导入方式

JS有常见的三种导入方式：1）在HEAD标签内导入；2）在body标签内导入；3）外联导入（需创建一个js格式的文件）

这三种方式均需配合`script`标签一起使用，我们这里在`scripe`标签内使用`console`的`log`函数，展示这三种方式：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>JS导入方式</title>
    <script>
        console.log('Hello, HEAD标签的内联样式');
    </script>
    <script src="./myscript.js"></script>
</head>
<body>
    <h1>JaveScript 的导入方式</h1>
    <script>
        console.log('Hello, body标签的内联样式');
        alert("你好，内联样式弹窗")
    </script>
</body>
</html>
```

`myscript.js`文件内容如下：

```
console.log('Hello, 外联样式');
```

我们可以在浏览器页面中点击F12，进入console查看日志：

![image-20241114201424406](/images/$%7Bfiilename%7D/image-20241114201424406.png)

很明显，导入成功。

此外，我们可以在`script`标签内做其他尝试，比如我这里使用`alert`函数弹出一个弹窗，显示“你好，内联样式弹窗”：

```html
    <script>
        console.log('Hello, body标签的内联样式');
        alert("你好，内联样式弹窗")
    </script>
```

![image-20241114201520689](/images/$%7Bfiilename%7D/image-20241114201520689.png)

## 1.2 JS的基本语法

### 1.2.1 JS的变量和数据类型

JavaScript 使用 `var`、`let` 和 `const` 来声明变量。

- `var`：声明一个变量，使用 `var` 声明的变量会有函数作用域或全局作用域，容易导致一些问题，现已较少使用。
- `let`：声明一个块级作用域的变量。
- `const`：声明一个常量，值不可改变。

```javascript
<script>
    var x;
    let name = 'Alice';  // 变量可以重新赋值
    const age = 25;      // 常量，不可修改
    console.log(x,name,age);
</script>
```

我们可以调用console的log函数查看这些值：

```javascript
    <script>
        var x;
        let name = 'Alice';  // 变量可以重新赋值
        const age = 25;      // 常量，不可修改
        console.log(x,name,age);
        let y = '示例';
        console.log(y);
        let empty_value = null;
        console.log(empty_value);
    </script>
```

![image-20241114202518029](/images/$%7Bfiilename%7D/image-20241114202518029.png)

------

JavaScript 有多种数据类型，主要分为两类：基本类型（原始类型）和引用类型。

- **基本类型**（Primitive types）：
  - `number`：数字类型
  - `string`：字符串类型
  - `boolean`：布尔类型（`true` 或 `false`）
  - `null`：空值
  - `undefined`：未定义的值
  - `symbol`：唯一标识符（ES6 引入）
  - `bigint`：大整数（ES11 引入）
- **引用类型**（Reference types）：
  - `object`：对象
  - `array`：数组
  - `function`：函数（也是对象）

```javascript
let num = 100;           // number
let name = 'Alice';      // string
let isActive = true;     // boolean
let person = { name: 'Alice', age: 25 };  // object
```

### 1.2.2 JS的控制语句

JavaScript 使用 `if`、`else`、`else if` 来进行条件判断。

```javascript
let age = 18;
if (age >= 18) {
    console.log('成年人');
} else {
    console.log('未成年人');
}
```

还可以使用 `switch` 语句进行多分支判断：

```javascript
let day = 3;
switch(day) {
    case 1:
        console.log('星期一');
        break;
    case 2:
        console.log('星期二');
        break;
    case 3:
        console.log('星期三');
        break;
    default:
        console.log('未知');
}
```

------

JavaScript 支持 `for`、`while`、`do...while` 等循环语句。

- **for 循环**：

  ```javascript
  for (let i = 0; i < 5; i++) {
      console.log(i);  // 输出 0 到 4
  }
  ```

- **while 循环**：

  ```javascript
  let i = 0;
  while (i < 5) {
      console.log(i);  // 输出 0 到 4
      i++;
  }
  ```

- **do...while 循环**：

  ```javascript
  let i = 0;
  do {
      console.log(i);  // 输出 0 到 4
      i++;
  } while (i < 5);
  ```

### 1.2.3 JS的函数

在 JavaScript 中，函数可以通过函数声明或函数表达式定义。

- **函数声明**：

  ```javascript
  function greet(name) {
      console.log('Hello, ' + name);
  }
  greet('Alice');  // 调用函数
  ```

- **函数表达式**：

  ```javascript
  let greet = function(name) {
      console.log('Hello, ' + name);
  };
  greet('Bob');  // 调用函数
  ```

- **箭头函数**（ES6 引入）：

  ```javascript
  let greet = (name) => {
      console.log('Hello, ' + name);
  };
  greet('Charlie');
  ```

### 1.2.4 JS的数组与对象

**数组**：用于存储多个值，索引从 0 开始。

```javascript
let fruits = ['apple', 'banana', 'cherry'];
console.log(fruits[0]);  // 输出 'apple'
```

**对象**：用于存储具有键值对的数据。

```javascript
let person = {
    name: 'Alice',
    age: 25
};
console.log(person.name);  // 输出 'Alice'
```

### 1.2.5 JS事件处理

JavaScript 事件处理是指在用户与网页元素**交互**时，浏览器响应并执行相应的 JavaScript 代码的机制。事件处理是前端开发中的一个重要部分，涉及到如何让网页响应用户的点击、输入、键盘操作、鼠标移动等行为。

#### 1.2.5.1 事件的基本概念

事件（Event）是用户与网页交互时产生的动作。每当用户与网页元素进行交互时，都会触发一个事件。常见的事件包括：

- **鼠标事件**：`click`（点击）、`dblclick`（双击）、`mouseover`（鼠标悬停）、`mouseout`（鼠标离开）
- **键盘事件**：`keydown`（按下键盘）、`keyup`（松开键盘）、`keypress`（按下字符键）
- **表单事件**：`submit`（提交表单）、`input`（输入框变化）、`change`（元素内容变化）
- **窗口事件**：`load`（页面加载完成）、`resize`（窗口大小调整）、`scroll`（滚动）

#### 1.2.5.2 **事件处理的基本方式**

a.  **内联事件处理**

最简单的事件处理方法是直接在 HTML 标签中使用事件属性。例如：

```html
<button onclick="alert('按钮被点击了')">点击我</button>
```

这种方法将事件处理程序直接嵌入到 HTML 标签的 `onclick` 属性中。尽管它使用方便，但通常不推荐这种做法，因为它将行为与结构混合在一起，难以维护。

![image-20241114204312729](/images/$%7Bfiilename%7D/image-20241114204312729.png)

b. **通过 JavaScript 事件处理程序**

通常，我们更倾向于使用 JavaScript 来处理事件，而不是在 HTML 中内联定义。这样可以将事件的处理逻辑与 HTML 内容分离，使代码更加清晰和可维护。

```html
<button id="myButton">点击我</button>

<script>
    let button = document.getElementById('myButton');
    button.onclick = function() {
        alert('按钮被点击了');
    };
</script>
```

这里我们使用 `onclick` 属性将事件处理程序绑定到按钮元素。当按钮被点击时，事件处理程序会被触发并执行。

![image-20241114204401659](/images/$%7Bfiilename%7D/image-20241114204401659.png)

c. **使用 `addEventListener` 方法**

`addEventListener` 是现代 JavaScript 中最推荐的事件绑定方法。它允许你向一个元素添加多个事件监听器，而不会覆盖已有的事件监听器。使用这种方法，可以更加灵活地控制事件的处理方式。

```html
<button id="myButton">点击我</button>

<script>
    let button = document.getElementById('myButton');
    
    // 使用 addEventListener 绑定事件
    button.addEventListener('click', function() {
        alert('按钮被点击了');
    });
</script>
```

d.  **`removeEventListener` 方法**

如果需要在某些条件下移除已经绑定的事件处理程序，可以使用 `removeEventListener` 方法。此方法与 `addEventListener` 相对应，移除之前通过 `addEventListener` 绑定的事件监听器。

```html
<button id="myButton">点击我</button>
<button id="removeButton">移除点击事件</button>

<script>
    let button = document.getElementById('myButton');
    let removeButton = document.getElementById('removeButton');
    
    function handleClick() {
        alert('按钮被点击了');
    }
    
    button.addEventListener('click', handleClick);
    
    removeButton.addEventListener('click', function() {
        button.removeEventListener('click', handleClick);
        alert('事件已被移除');
    });
</script>
```

#### 1.2.5.3 **事件对象**

当事件被触发时，浏览器会生成一个事件对象（`event`），该对象包含了有关事件的所有信息，如触发事件的元素、事件类型、键盘按键等。可以通过事件处理程序访问该对象。

```html
<button id="myButton">点击我</button>
<script>
    let button = document.getElementById('myButton');
    
    button.addEventListener('click', function(event) {
        console.log(event); // 输出事件对象的详细信息
        console.log('事件的目标元素:', event.target);
        console.log('鼠标点击的坐标:', event.clientX, event.clientY);
    });
</script>
```

常用的事件对象属性：

- `event.target`：触发事件的元素。
- `event.type`：事件的类型（例如 `click`、`mouseover` 等）。
- `event.clientX` 和 `event.clientY`：鼠标相对于浏览器窗口的坐标。
- `event.key`：在键盘事件中，返回按下的键的名称（例如 `'Enter'`、`'A'`）。

![image-20241114204631704](/images/$%7Bfiilename%7D/image-20241114204631704.png)

#### 1.2.5.4 **事件冒泡与事件捕获**

JavaScript 中的事件传播机制有两种方式：**事件冒泡**和**事件捕获**。

- **事件冒泡**：事件从目标元素冒泡到父元素。即，事件首先在目标元素上触发，然后依次向上冒泡到祖先元素，直到最顶层的 `document`。

  例如，在一个 `<div>` 内的 `<button>` 上点击时，`button` 的点击事件会先被触发，然后事件会冒泡到外部的 `div` 元素。

- **事件捕获**：事件从最外层的父元素开始捕获，直到目标元素。捕获阶段会先触发父元素的事件处理程序，然后才是目标元素的事件处理程序。

可以通过 `addEventListener` 的第三个参数来控制事件是处于冒泡阶段还是捕获阶段：

```html
element.addEventListener('click', function(event) {
    console.log('捕获阶段');
}, true); // true 表示捕获阶段，false 或不传参数表示冒泡阶段
```

#### 1.2.5.5 **事件委托**

事件委托是通过在父元素上绑定事件，而不是在每个子元素上绑定事件，来优化性能的一种技巧。特别是在处理动态生成的元素时，事件委托非常有用。

例如，假设你有一个列表，里面包含了多个按钮，传统方法会给每个按钮单独绑定事件。但使用事件委托，只需要在父元素上绑定一个事件处理程序即可。

```html
html复制代码<ul id="list">
    <li><button>按钮 1</button></li>
    <li><button>按钮 2</button></li>
    <li><button>按钮 3</button></li>
</ul>

<script>
    let list = document.getElementById('list');
    
    list.addEventListener('click', function(event) {
        if (event.target.tagName === 'BUTTON') {
            alert(event.target.textContent);  // 显示按钮的文本内容
        }
    });
</script>
```

## 1.3 DOM

在 Web 开发中，DOM 通常与 JavaScript 一起使用。

- 当网页被加载时，浏览器会创建页面的文档对象模型，也就是DOM(Document Object Model)。
- 每个 HTML 或 XML 文档都可以被视为一个文栏树，文档树是整个文档的层次结构表示。
- 文档节点是整个文档树的根节点。
- DOM 为这个文档树提供了一个编程接口，开发者可以使用 JavaScript 来操作这个树状结构。

![image-20241114210611196](/images/$%7Bfiilename%7D/image-20241114210611196.png)

一个节点树如上图所示。

