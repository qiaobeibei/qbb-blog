---
title: 如何在windows上下载和编译grpc
date: 2024-11-03 12:20:39
categories:
- windows
tags: 
- 配置环境
typora-root-url: ./..
---



首先，将github上的grpc项目git到本地，grpc网址：

[GitHub - grpc/grpc: The C based gRPC (C++, Python, Ruby, Objective-C, PHP, C#)github.com/grpc/grpc编辑![img](/images/$%7Bfiilename%7D/icon-default-1730607675597-272.png)https://link.zhihu.com/?target=https%3A//github.com/grpc/grpc](https://link.zhihu.com/?target=https%3A//github.com/grpc/grpc)

将grpc仓库clone到本地后，进入grpc目录并将grpc仓库中的所有子模块初始化和更新：

```
git clone https://github.com/grpc/grpc.git 
cd ./grpc/
git submodule update  --init
```

因为国内进入github比较慢，我用vpn加速了这个过程，如果不会使用梯子的，可以通过博主恋恋风辰的方法，使用国内的代码工具gitee进行clone：

[恋恋风辰官方博客llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2QSEHcC1he1RgiewYG93ilaAMiY![img](/images/$%7Bfiilename%7D/icon-default-1730607675597-272.png)https://link.zhihu.com/?target=https%3A//llfc.club/category%3Fcatid%3D225RaiVNI8pFDD5L4m807g7ZwmF%23%21aid/2QSEHcC1he1RgiewYG93ilaAMiY](https://link.zhihu.com/?target=https%3A//llfc.club/category%3Fcatid%3D225RaiVNI8pFDD5L4m807g7ZwmF%23!aid/2QSEHcC1he1RgiewYG93ilaAMiY)

然后，使用cmake编译该文件，首先在grpc目录下新建一个vs文件夹

![img](/images/$%7Bfiilename%7D/format,png-1730607675597-269.png)

打开cmake软件，选择编译文件目录和保存目录，注意，编译文件目录得选择**CMakeLists.txt**文件的所属目录，保存目录选择刚才建立的vs文件夹。

点击configure，选择你的vs软件，我这里是vs2019，平台就默认选择x64

出现下面的界面，默认全选，点击generate：

![img](/images/$%7Bfiilename%7D/format,png-1730607675597-270.png)

最后，进入vs文件夹下打开grpc.sln进行编译，选择ALL_BUILD



![img](/images/$%7Bfiilename%7D/format,png-1730607675597-271.png)

右键->重新生成，等待编译成功即可。

恋恋风辰的博客中说编译grpc需要安装nasm、go和perl，但我这里都没有安装，直接进行了编译，而且也可以编译成功，如果你们遇到了编译错误的问题，可参考恋恋风辰的编译过程：

[恋恋风辰官方博客](https://llfc.club/category?catid=225RaiVNI8pFDD5L4m807g7ZwmF#!aid/2VWIJgH3zKEww0BpLnYQX0NMpQ9)
