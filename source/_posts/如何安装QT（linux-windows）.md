---
title: 如何安装QT（linux/windows）
date: 2024-11-05 19:41:27
categories:
- QT
tags: 
typora-root-url: ./..
---

# 1. linux

## 1.1 下载安装程序

进入QT官网，点击右上角下载

www.qt.io/

![img](https://picx.zhimg.com/80/v2-57969abe08134db713fd5a3ec74e5d5b_720w.png?source=d16d100b)

然后选择下载linux版本，这里你需要填写一些信息，注册一些即可

![img](/images/$%7Bfiilename%7D/v2-4c9189bf279b9fb99f4b400d09ce266e_720w.png)

![img](https://picx.zhimg.com/80/v2-3df337f7645c8c8e7f94aece1d580d1a_720w.png?source=d16d100b)

填写之后会出现下面这个网页，我选择的是linux x64版本，你们可输入下面的指令查看你的架构”

```
uname -m
```

- 如果返回 x86_64，则选择 **linux x64**。
- 如果返回 aarch64，则选择 **linux ARM64**

![img](https://picx.zhimg.com/80/v2-db1afb20bad5892ea05dcf209e72d4da_720w.png?source=d16d100b)



下载的文件名类似这样：

![img](/images/$%7Bfiilename%7D/v2-2cd43faf29065a4c14bcb93b36f66325_720w.png)



或者你也可以直接在linux中输入下列指令下载

```
wget 下载链接地址
```

以上QT的安装程序下载完成，接下来介绍如何安装和配置环境

### 1.2 linux操作

首先创建QT-C++目录，并在QT目录下创建QT文件夹

```
mkdir QT-C++
cd QT
mkdir QT
ls
```

然后将你的安装程序移动到QT-C++目录下，然后你cd到QT目录下将QT安装到此文件夹下

![img](https://picx.zhimg.com/80/v2-4d5779b3b75b0d8453d0b45f1ebba467_720w.png?source=d16d100b)

输入以下命令，为该程序添加可执行权限

```
cd QT
../ chmod +x qt-online-installer-linux-x64-4.8.1.run
```

运行该文件

```
../qt-online-installer-linux-x64-4.8.1.run
```

出现该界面表示运行成功

![img](/images/$%7Bfiilename%7D/v2-29cc4145a9a06ae5e84c7df241567cec_720w.png)

输入你刚才在官网注册的账号，点击下一步

![img](https://picx.zhimg.com/80/v2-f722eb4c9df7c13668d377d4ac002616_720w.png?source=d16d100b)

选择安装文件夹为QT文件夹，选择custom installation即可

![img](https://picx.zhimg.com/80/v2-deee918da782d21aac0d9b12dea6f2fd_720w.png?source=d16d100b)

我这里需要安装5.15版本，但是QT安装程序中没有，我们这里点击右边的 `Archive` ，再点击筛选，这样可以看到qt以前的一些其他的版本

![img](/images/$%7Bfiilename%7D/v2-8f0f9847a8d38bd8d47f3a47a96387a3_720w.png)

![img](/images/$%7Bfiilename%7D/v2-8ef802933a1e3cb724172a3b537de541_720w.png)

选择qt5.15，组件只需要选择第一个即可，在使用过程中如果需要其他组件，可对于的添加下载，全部选择需要耗费一百多个g，没必要，我们只用选择我们需要的即可。

![img](/images/$%7Bfiilename%7D/v2-9e165e57195c1c64ba7be0ca24127dcf_720w.png)

![img](/images/$%7Bfiilename%7D/v2-209b9989118c1e9f4634157db15ffc7b_720w.png)

![img](/images/$%7Bfiilename%7D/v2-94099b623b90f38e7f43c5a52ed0b7d0_720w.png)

等待安装完即可

![img](/images/$%7Bfiilename%7D/v2-ea3d58cf17495920b148ab5671e28fd1_720w.png)

![img](/images/$%7Bfiilename%7D/v2-167db3783c4cd28179f62ac906925ca2_720w.png)

点击下一步

![img](https://picx.zhimg.com/80/v2-caa354d2895f4c1f913357ed4665d44b_720w.png)

三个都不选择，点击完成

查找qtcreator位置，cd到你的qt安装目录下，运行

```
find . -name qtcreator
```

![img](/images/$%7Bfiilename%7D/v2-b019e99aaf9f0e0da9eae614a6c5a858_720w.png)

查看箭头所指的位置，cd进去输入下列指令即可运行

```
./qtcreator.sh
```

我们也可以将其设为快捷方式：

```
sudo ln -s /mnt/datab/home/yuanwenzheng/Qt/Tools/QtCreator/bin/qtcreator.sh /usr/local/bin/qtcreator
```

其中，“ /mnt/datab/home/yuanwenzheng/Qt/Tools/QtCreator/bin/qtcreator.sh”是你qtcreator.sh文件所在的绝对路径

这样，输入下列指令即可打开qt

```
qtcreator
```

![img](/images/$%7Bfiilename%7D/v2-8a8d92229c22371245103eab83fff340_720w.png)

### 1.3 创建项目

![img](/images/$%7Bfiilename%7D/v2-29e62955d6ab4bc3295b1d31531186e4_720w.png)

我们这里随便选择一个example，点击，我这里选择了左下角这个。

然后点击configure project

![img](https://picx.zhimg.com/80/v2-5a59d5dd303d215c265c5910e05a9b69_720w.png)

点击绿色的运行

![img](https://picx.zhimg.com/80/v2-e666991baecf5b0d9db541083167e3a7_720w.png)

如果出现以下界面表示基本组件安装成功

![img](https://picx.zhimg.com/80/v2-b24a1096286522a99d43ffd9c212d547_720w.png)

------

如果报错说QT找不到XCB相关的依赖项，通过下面的指令安装xcb相关的组件依赖项

```
sudo apt install libxcb-*
```

## 2. windows

前面的步骤都相同，只不过我们这里安装的windows x64

![img](https://picx.zhimg.com/80/v2-fe6f12330bbc98374eb1ec611517d02a_720w.png?source=d16d100b)

双击打开，然后输入账号密码，一直点下一步

选择安装目录，依旧选择默认安装

![img](/images/$%7Bfiilename%7D/v2-26ca290d5e15fcbd48ec37ba93016257_720w.png)

同理，如果没有想要安装的版本，我这里准备安装5.15，但选择界面没有，我们可以点击右面的archive，然后筛选

![img](/images/$%7Bfiilename%7D/v2-8b1eb22b9da459f0f7b5f860408af38f_720w.png)

重新出现界面后，即可发现以往的QT版本，我这里选择5.15.2。注意，不要将5.15的所有组件都添加，我们这里只选择下面这几个组件，其余的组件等用到的时候也可以添加。其余默认

![img](/images/$%7Bfiilename%7D/v2-105b57641fc9ea4e0927f764a12d2dae_720w.png)

点击安装

![img](/images/$%7Bfiilename%7D/v2-4daab5aab3a06ab214ea70c4cce47f23_720w.png)

等待安装完成即可

![img](/images/$%7Bfiilename%7D/v2-a82bf80c0f437c0c3a851511ccfd8693_720w.png)

之后的步骤和linux相同
