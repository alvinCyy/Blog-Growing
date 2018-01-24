---
title: 在vs环境下编译zxing-cpp
date: 2018-01-23 14:52:22
categories:
 术业专攻
tags:
 [Zxing,Win32]
---
<Excerpt in index | 首页摘要>
<!-- more -->
<The rest of contents | 余下全文>

#在Win32中使用Zxing-cpp
## 前言
    最近接手一个项目需要识别通过监控摄像机识别二维码，本客户端已经做好RtspClient收流和H.264和H.265解码。这就只需要通过在播放器解码后获取到YUV数据解析得到二维码数据。
### 环境与工具
>* window10
>* visual studio 2017
>* OpenCv 3.4.0
>* cmake gui

### zxing-cpp系列
>* [在vs环境下编译zxing-cpp](https://alvincyy.github.io/2018/01/24/win32下zxing的使用-1/)
>* [在vs项目中使用zxing-cpp](https://alvincyy.github.io/2018/01/24/在vs项目中使用zxing-cpp/)

## 正文
### Zxing-cpp的编译
我们将使用visual studio 2017 配合OpenCv 3.4.0来编译Zxing-cpp

#### 下载地址:
>* [Zxing-cpp](https://github.com/glassechidna/zxing-cpp)
>* [cmake gui](https://cmake.org/download/)
>* [opencv](http://opencv.org/releases.html)

#### 准备工作</br>
>* 下载好Zxing-cpp源码解压到常用工作的目录
    文件结构如下
~~~
    zxing-cpp
    |--cli
    |--cmake
    |--core
    |--opencv
    |--opencv-cli
~~~
#### 安装并配置CMake
  打开 CMake GUI 工具
  设置 source 目录为:/Zxing-cpp，build 目录为: D:/zxing-cpp-master/build.
  打开 Configure 窗口, 选择 Visula studio 15 2017, 单击 Finish 完成设置.
  单击 Generate 按钮, 生成 VS2017 工程.
  在build目录下就可以找到生成的vs项目。最后用vs2017打开项目编译即可。

### 参考文献
>* [如何在visual studio下编译zxing cpp，以及zxing c++的使用](http://blog.csdn.net/sinat_29957455/article/details/60467090)<br>
>* [C++用zxing识别二维码](http://blog.csdn.net/coolingcoding/article/details/25804129)<br>

### 最后的话<br>
>* 如果遇到问题可以给我发邮件[chenyiyu@gmail.com](mailto:chenyiyu@gmail.com)<br>
>* 转载请注明原地址，[Alvin的博客](http://alvinCyy.github.io) 谢谢！