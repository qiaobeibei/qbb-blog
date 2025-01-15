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







如果报下列错误：

```ymal
undefined reference to `__imp__ZN21QNetworkAccessManagerC1EP7QObject'
```

表明在链接阶段，链接器找不到 `QNetworkAccessManager` 的构造函数的实现，如果我们使用的**qmake**，那么只需要在 .pro 文件中有如下配置：

```
QT += core gui network
```

如果我们使用的是**camke**，那么需要在 CmakeLists.txt 文件中添加以下代码：

```
find_package(Qt5 REQUIRED COMPONENTS Core Network)
target_link_libraries(project-chat PRIVATE Qt5::Core Qt5::Network)
```

手动链接 `QtNetwork` 库





如果报错：

```
'staticMetaObject' is not a member of 'Singleton<HttpMgr>'
```

这个错误是因为 `Singleton<HttpMgr>` 模板类没有定义 `staticMetaObject`，而 Qt 的 MOC工具要求所有需要元对象功能的类必须继承自 `QObject`，并且通过 `Q_OBJECT` 宏声明。

我们需要看 HttpMgr 是否继承自 `QObject` 并声明 `Q_OBJECT`

```cpp
class HttpMgr : public QObject
{
    Q_OBJECT
public:
```

这是对的，必须要将 QObject 放在第一个继承并且声明宏 Q_OBJECT，

我们不能将 QObject 放在第二个继承或者更后面，比如：

```cpp
class HttpMgr : public Singleton<HttpMgr>, public QObject
{
    Q_OBJECT

    // 类的声明和实现...
};
```

`QObject` 在 Qt 的元对象系统中有特殊要求，尤其是与 MOC生成的代码配合时。MOC 默认 `QObject` 是继承列表中的**第一个基类**，这样可以确保它的元对象系统能够正确处理信号和槽。

当 `QObject` 不是第一个基类时：

- MOC 无法正确解析类的元数据。
- 导致链接错误或运行时行为异常。
