---
title: chat(2)——界面优化
date: 2025-01-13 10:27:05
categories:
- C++
- 即时通讯项目
tags: 
- QT
typora-root-url: ./..
---

# 二、day2

在 register 类的构造函数中添加如下代码，将注册界面输入密码的 lineedit 设为密码模式：

```cpp
ui->pass_edit->setEchoMode(QLineEdit::Password);
ui->confirm_edit->setEchoMode(QLineEdit::Password);
```

