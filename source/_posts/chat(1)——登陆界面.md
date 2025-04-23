---
title: 登陆界面头像处理
date: 2025-01-10 16:30:34
categories:
- C++
- 即时通讯项目
tags: 
- QT
typora-root-url: ./..
---

# 一、day1

1.[iconfont Logo](https://www.iconfont.cn/)下载一个图标png，大小为256像素，前面颜色自己看着弄

![image-20250110163215514](/images/$%7Bfiilename%7D/image-20250110163215514.png)

2.png转ico,[转化链接](https://www.aconvert.com/cn/icon/png-to-ico/)

![image-20250110163235010](/images/$%7Bfiilename%7D/image-20250110163235010.png)

上传完下载的png文件后选择24*24，开始转换。然后右键输出文件，选择将链接另存为，保存到桌面即可

![image-20250110163423081](/images/$%7Bfiilename%7D/image-20250110163423081.png)

cmake 修改 .exe 程序图标

1. 获得一个logo.ico图标(假设这个ico图标的名称为logo)
   ![微信](/images/$%7Bfiilename%7D/%E5%BE%AE%E4%BF%A1.png)

2. 创建一个名称为logo.rc文件，在文件中添加下面一行文本

   ```python
   DI_ICON1     ICON    DISCARDABLE     "logo.ico"
   ```

3. 在QT工程中添加资源文件，并将logo.ico和logo.rc添加到资源文件中，具体步骤是右键点击"你的项目名称"=>添加新文件=>QT=>创建QT Resource File（名称设为 images），然后右键点击生产资源文件`images.qrc`=>选择添加现有文件，添加logo.ico和logo.rc

4. 修改`cMakeLists`内容

   ```cmake
   if(${QT_VERSION_MAJOR} GREATER_EQUAL 6)
       qt_add_executable(project-chat
           MANUAL_FINALIZATION
           ${PROJECT_SOURCES}
           images.qrc
       )
   ```

   在上面代码中添加资源文件路径`images/logo.rc`

   ```cmake
   if(${QT_VERSION_MAJOR} GREATER_EQUAL 6)
       qt_add_executable(project-chat
           MANUAL_FINALIZATION
           ${PROJECT_SOURCES}
           images.qrc
           images/logo.rc
       )
   ```

5. 重新编译即可修改`你的项目名称.exe`程序图标

**修改显示页面左上角图标**

1. 首先按照1.1.1中的方法将`logo.ico`添加到资源文件中
2. 在ui界面进行设置，具体步骤为：双击mainwindow.ui=>windowIcon=>选择资源文件=>选择logo.ico

![image-20250110163957712](/images/$%7Bfiilename%7D/image-20250110163957712.png)

![image-20250110164035110](/images/$%7Bfiilename%7D/image-20250110164035110.png)

重新运行即可，可以看到左上角变为了`logo.ico`图标了

![image-20250110164117093](/images/$%7Bfiilename%7D/image-20250110164117093.png)





![image-20250110164435464](/images/$%7Bfiilename%7D/image-20250110164435464.png)

![image-20250110164508601](/images/$%7Bfiilename%7D/image-20250110164508601.png)

![image-20250110164539473](/images/$%7Bfiilename%7D/image-20250110164539473.png)

![image-20250110164614234](/images/$%7Bfiilename%7D/image-20250110164614234.png)

![image-20250110164645711](/images/$%7Bfiilename%7D/image-20250110164645711.png)

建立垂直布局，将Vertical Layout右键拖拽到布局界面中

![image-20250110164737723](/images/$%7Bfiilename%7D/image-20250110164737723.png)

将Vertical Layout拖拽过去后，选择最大的框，点击箭头指向的位置，我们将login layout变为垂直布局

![image-20250110164838153](/images/$%7Bfiilename%7D/image-20250110164838153.png)

![image-20250110164929609](/images/$%7Bfiilename%7D/image-20250110164929609.png)

然后我们会在这里面增加很多不同的水平布局，

![image-20250110165009703](/images/$%7Bfiilename%7D/image-20250110165009703.png)

这就是我们新添加的水平布局

![image-20250110165128534](/images/$%7Bfiilename%7D/image-20250110165128534.png)

我们将垂直布局的熟悉设为如下：

![image-20250110165240297](/images/$%7Bfiilename%7D/image-20250110165240297.png)

并拖拽一个标签label和line Edit至布局框内

![image-20250110165318191](/images/$%7Bfiilename%7D/image-20250110165318191.png)

![image-20250110165342043](/images/$%7Bfiilename%7D/image-20250110165342043.png)

然后调整布局，以及label内容修改为“用户名：”

![image-20250110165819771](/images/$%7Bfiilename%7D/image-20250110165819771.png)

将widget拖拽过去

![image-20250110165813158](/images/$%7Bfiilename%7D/image-20250110165813158.png)

拖拽至最上面

![image-20250110170200289](/images/$%7Bfiilename%7D/image-20250110170200289.png)

然后往widget中拖拽一个label

![image-20250110170241829](/images/$%7Bfiilename%7D/image-20250110170241829.png)

widget用于展示我们的头像，我们这里选择一个图片添加至 `image.qrc`中

然后我们在设计界面找到该label的pixmap，选择资源为刚才加入的图片，并设置为水平和垂直中心对齐

![image-20250110171054667](/images/$%7Bfiilename%7D/image-20250110171054667.png)

![image-20250110171105166](/images/$%7Bfiilename%7D/image-20250110171105166.png)

同理，为密码创建一个水平布局、label和line edit

![image-20250110171352712](/images/$%7Bfiilename%7D/image-20250110171352712.png)

再新建一个水平布局、label以及horizontal spacer用于找回密码

![image-20250110171624826](/images/$%7Bfiilename%7D/image-20250110171624826.png)

![image-20250110171728934](/images/$%7Bfiilename%7D/image-20250110171728934.png)

再添加一个水平布局，并添加push Button

![image-20250110171824525](/images/$%7Bfiilename%7D/image-20250110171824525.png)

并添加两个horizontal spacer至 push Button 左右

![image-20250110171915507](/images/$%7Bfiilename%7D/image-20250110171915507.png)

![image-20250110172044573](/images/$%7Bfiilename%7D/image-20250110172044573.png)

同样，为注册也添加一个水平布局，并添加push Button和两个horizontal spacer

![image-20250110172215623](/images/$%7Bfiilename%7D/image-20250110172215623.png)

设计好界面后，再 mainwindow.h 中将 login.h 包含进去，并创建一个 Login* 类型的成员变量 _login

```cpp
#ifndef MAINWINDOW_H
#define MAINWINDOW_H
#include <QMainWindow>
#include "login.h"
QT_BEGIN_NAMESPACE
namespace Ui {
class MainWindow;
}
QT_END_NAMESPACE

class MainWindow : public QMainWindow
{
    Q_OBJECT

public:
    MainWindow(QWidget *parent = nullptr);
    ~MainWindow();

private:
    Ui::MainWindow *ui;
    Login* _login;
};
#endif // MAINWINDOW_H
```

在 mainwindow.cpp 中加入如下代码

```cpp
#include "mainwindow.h"
#include "./ui_mainwindow.h"

MainWindow::MainWindow(QWidget *parent)
    : QMainWindow(parent)
    , ui(new Ui::MainWindow)
{
    ui->setupUi(this);
    _login = new Login();
    setCentralWidget(_login);
    _login->show();
}

MainWindow::~MainWindow()
{
    delete ui;
}
```

运行会弹出我们设计的界面

![image-20250110172903449](/images/$%7Bfiilename%7D/image-20250110172903449.png)

登录设计完，我们再添加一个注册界面，创建的步骤和登录一样，然后进入注册的设计界面。

首先添加一个垂直布局，并将整个布局设置为垂直布局：

![image-20250110173112815](/images/$%7Bfiilename%7D/image-20250110173112815.png)

添加 stacked widget

![image-20250110173224405](/images/$%7Bfiilename%7D/image-20250110173224405.png)

该组件会创建两页，我们在第一页添加一个水平布局

![image-20250110173326839](/images/$%7Bfiilename%7D/image-20250110173326839.png)

然后点击StackWidget选择水平布局，并添加label和line edit

![image-20250110173506093](/images/$%7Bfiilename%7D/image-20250110173506093.png)

并再次添加一个水平布局、label和line edit

![image-20250110173828459](/images/$%7Bfiilename%7D/image-20250110173828459.png)

然后依次拖拽部件设计如下界面：

![image-20250110174652403](/images/$%7Bfiilename%7D/image-20250110174652403.png)

同理，在 mainwindow.h 和 mainwindow.cpp中修改代码



最后效果图：

![image-20250113101920693](/images/$%7Bfiilename%7D/image-20250113101920693.png)



[c/c++项目：半小时手把手教你用QT写出QQ登录界面！完美复刻_哔哩哔哩_bilibili](https://www.bilibili.com/video/BV19jewezEcw/?spm_id_from=333.337.search-card.all.click&vd_source=29868cdbb6b2fb1514ce3c7c31892d68)
