---
title: Clutch 砸壳
date: 2018-10-01 18:35:54
tags:
    - 逆向
categories: iOS
---

## 工具

+ [Clutch](https://github.com/KJCracks/Clutch)
+ 一台越狱手机（实验机为 iPhone5S，系统 iOS 8.1.3）
+ 终端

<!--more-->

## 具体操作

### 安装 Clutch
推荐使用 Clutch-2.0.4 版本。
```
$ git clone https://github.com/KJCracks/Clutch Clutch-2.0.4
$ cd Clutch-2.0.4
```

### 获取 Clutch.app 的可执行文件

**方式一：通过 xcodebuild 指令获取**

cd 到 clone 下来的 Clutch 目录下，执行：

```
$ xcodebuild -project Clutch.xcodeproj -configuration Release ARCHS="armv7 armv7s arm64" build
```
生成出来的 Unix 可执行文件 clutch 就在当前目录下。

**方式二：通过 Clutch.app 包获取**

打开 Clutch.xcodeproj ，编译成功之后，在 Products 目录下找到 Clutch.app，在包内获取 Clutch.app 的可执行文件 clutch。

将可执行文件拷贝到手机上:
```
scp <Clutch Executable directory> root@<your.device.ip>:/usr/bin/
```

### 使用 ssh 连接手机

打开终端界面，使用 ssh 连接手机：

```
$ ssh root@<your.device.ip>
$ clutch -i
```
终端输出：

> Installed apps:
> 1:   Flashlight <com.bigblueclip.led>
> 2:   微信 <com.tencent.xin>
> 3:   QQ同步助手 <com.tencent.QQPim>

根据列表中显示的包名进行砸壳，这里以微信为例

```
$ clutch -d com.tencent.xin
```
可以看到 Clutch 砸壳后的 ipa 文件放到了`/private/var/mobile/Documents/Dumped/`目录下。
修改成一个简单的名字，然后拷贝回电脑：

```
$ mv /private/var/mobile/Documents/Dumped/com.tencent.xin-iOS8.0-\(Clutch-2.0.4\).ipa /private/var/mobile/Documents/Dumped/wechat.ipa
$ scp root@<your.device.ip>:/private/var/mobile/Documents/Dumped/wechat.ipa ~/Desktop
```
砸壳完毕。
