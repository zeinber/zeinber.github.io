---
title: iOS App 重签名
date: 2017-11-08 15:43:11
tags:
    - 签名
categories: 逆向
toc: true
---

一旦 App 内的二进制文件被修改，或者，应用本身的签名就会被破坏。如果想将修改的文件安装到手机上，就需要对应用进行重签名

<!--more-->

## [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)
+ 原理：让 App 在启动的时候加载动态库 dumpdecrypted.dylib 并执行里面的解密代码，dump 出被加密部分，最后生成一个脱壳的二进制文件
+ 优势：成功率高
+ 劣势：
    + 步骤繁琐，解密以后还是一个 .decrypted 结尾的文件，还需要手动转换为可执行文件
+ 推荐指数：🌟

### 准备步骤
  + 越狱设备
    + OpenSSH、Cycript（没有请从 Cydia 上搜索下载）
  + Mac 电脑
    + [ldid](https://github.com/rpetrich/ldid)（签名工具）

### 使用步骤
#### 下载源代码并编译 
打开 Mac 终端
```
$ git clone https://github.com/stefanesser/dumpdecrypted
$ cd dumpdecrypted
$ make
```
得到一个动态库 dumpdecrypted.dylib

#### 动态库签名
苹果会对非系统的动态库校验签名，因此需要使用 ldid 对 dumpdecrypted.dylib 进行签名
```linux
$ ldid -S dumpdecrypted.dylib
```

#### 定位待解密的可执行文件
请确保越狱设备以及 Mac 保持在同一个网段，
将 Mac 终端链接到越狱设备

```linux
$ ssh root@<设备wifiIp地址>
```
设备默认密码为 alpine

#### 查看进程号和可执行文件路径

```linux  
$ ps -e
```
找到 App 可执行文件的进程号和路径并记录下来

#### 获取目标 App 的 Documents 目录
附加到指定进程

```cycript
$ cycript -p <App进程号>
```
 得到 App 的 Documents 文件夹路径
  ```objc
$ [[NSFileManager defaultManager] URLsForDirectory:NSDocumentDirectory inDomains:NSUserDomainMask][0]
  ```

Mac 终端新开一个窗口
```linux
$ scp <dumpdecrypte.dylib 路径> root@<设备 wifiIp 地址>:<App Documents路径> 
```
将 dumpdecrypted.dylib 拷贝到 App 的 Documents 目录

#### 解密
ctrl+z 退出 cy 状态，执行砸壳

```c 
$ DYLD_INSERT_LIBRARIES=<dumpdecrypted.dylib路径> <App可执行文件路径>
```
#### 保存砸壳文件
 Mac 终端执行

```linux 
$ scp root@<设备wifiIp地址>:<app可执行文件名>.decrypted <Mac端指定路径>
```
将 App 的 Documents 路径下以 .decrypted 结尾的解密后可执行文件，从越狱设备里拷到电脑上

## [Clutch](https://github.com/KJCracks/Clutch)
+ 原理：通过 posix_spawnp 创建一个进程，然后暂停进程并 dump 内存，最后生成一个脱壳的二进制文件
+ 优势：相较于 dumpdecrypted 更方便,无需手动注入动态库
+ 劣势：
    + 使用 Clutch 解密一些应用会经常失败
+ 推荐指数：🌟🌟🌟

### 准备步骤
  + 越狱设备
    + OpenSSH（没有请从 Cydia 上搜索下载）
  + Mac 电脑

### 使用步骤
#### 下载源代码并编译
推荐使用 Clutch-2.0.4 版本。
```
$ git clone --branch 2.0.4 https://github.com/KJCracks/Clutch
```

**方式一：通过 xcodebuild 指令获取**

cd 到 clone 下来的 Clutch 目录下，执行：

```
$ xcodebuild -project Clutch.xcodeproj -configuration Release ARCHS="armv7 armv7s arm64" build
```
生成出来的可执行文件 clutch 就在当前目录下

**方式二：通过 Clutch.app 包获取**

打开 Clutch.xcodeproj ，编译成功之后，在 Products 目录下找到 Clutch.app，在包内获取 Clutch.app 的可执行文件 clutch

#### 将文件复制到手机中

将可执行文件拷贝到手机上:
```
scp <可执行文件clutch路径> root@<设备wifIp地址>:/usr/bin/
```

#### 解密

越狱设备以及 Mac 保持在同一个网段
打开 Mac 终端，使用 ssh 连接手机：

```
$ ssh root@<设备wifiIp地址>
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
可以看到 Clutch 砸壳后的 ipa 文件放到了`/private/var/mobile/Documents/Dumped/`目录下

#### 保存砸壳文件
修改成一个简单的名字，然后拷贝回电脑：

```
$ mv /private/var/mobile/Documents/Dumped/com.tencent.xin-iOS8.0-\(Clutch-2.0.4\).ipa /private/var/mobile/Documents/Dumped/wechat.ipa
$ scp root@<设备wifiIp地址>:/private/var/mobile/Documents/Dumped/wechat.ipa ~/Desktop
```

### [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)
+ 原理：基于 frida 建立一个双向通信通道，然后在 Mac 端用 python 加载一个 dump.js，在运行应用的时候执行 js 内的解密代码，最后生成一个脱壳的二进制文件
+ 优势：高成功率，便利，自动把砸完壳的 App 传输到 Mac 端
+ 劣势：无
+ 推荐指数：🌟🌟🌟🌟🌟

### 准备步骤
  + 越狱设备
    + OpenSSH（没有请从 Cydia 上搜索下载）
  + Mac 电脑
    + 安装 python
      ```
      $ brew install python
      ```
    + 安装 wget
      ```
      $ brew install wget
      ```
    + 安装 pip
      ```
      $ wget https://bootstrap.pypa.io/get-pip.py
      $ sudo python get-pip.py
      ```
    + 安装 frida
      ```
      $ sudo pip install frida –upgrade –ignore-installed six
      ```
    + 安装脚本依赖环境
      ```
      $ sudo pip install -r requirements.txt --upgrade
      ```
    + 安装 usbmuxd 与手机通信
      ```
      $ brew install usbmuxd
      ```
    