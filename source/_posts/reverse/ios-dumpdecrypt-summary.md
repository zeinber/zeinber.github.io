---
title: iOS砸壳总结
date: 2017-11-08 15:43:11
tags:
    - 砸壳
categories: 逆向
toc: true
---

总结一下关于 iOS 砸壳的几种方式，方便以后查找，不定期更新...

<!--more-->

## [dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)
+ 原理：让 App 在启动的时候加载动态库 dumpdecrypted.dylib 并执行里面的解密代码，dump 出被加密部分，最后生成一个脱壳的二进制文件
+ 优势：成功率高
+ 劣势：步骤繁琐，解密以后还是一个 .decrypted 结尾的文件，还需要手动转换为可执行文件
+ 推荐指数：🌟

### 准备步骤
  + 越狱设备
    + 打开 cydia 在搜索中下载安装 OpenSSH、Cycript
  + Mac 电脑
    + 安装 ldid
    ```
    sudo brew install ldid
    ```

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
将 Mac 终端连接到越狱设备

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
+ 优势：相较于 dumpdecrypted 更方便，无需手动注入动态库
+ 劣势：使用 Clutch 解密一些应用会经常失败
+ 推荐指数：🌟🌟🌟

### 准备步骤
  + 越狱设备
    + 打开 cydia 在搜索中下载安装 OpenSSH
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

## [frida-ios-dump](https://github.com/AloneMonkey/frida-ios-dump)
+ 原理：基于 frida 建立一个双向通信通道，然后在 Mac 端用 python 加载一个 dump.js，在运行应用的时候执行 js 内的解密代码，最后生成一个脱壳的二进制文件
+ 优势：高成功率，便利，自动把砸完壳的 App 传输到 Mac 端
+ 劣势：无
+ 推荐指数：🌟🌟🌟🌟🌟

### 准备步骤
  + 越狱设备
    + 打开 cydia 添加源：http://build.frida.re 并在搜索中下载安装 frida，安装完成后在 Mac 端执行 frida-ps -U 查看是否能正常执行
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
### 使用步骤
#### 下载源代码
打开 Mac 终端
```
$ git clone https://github.com/AloneMonkey/frida-ios-dump
```
#### 修改 dump.py 
进入 frida-ios-dump 文件夹，打开 dump.py，以下配置是 Mac 端和越狱设备连接的配置，请根据需求进行修改
```
User = 'root'
Password = 'alpine'
Host = 'localhost'
Port = 2222
```
接下来按上述的默认配置进行说明

#### Mac 连接越狱设备
通过 USB 将 Mac 和越狱设备连接，将 22 映射到 Mac 上的 2222 端口，打开 Mac 终端输入
```
$ iproxy 2222 22
waiting for connection
```

#### 查看应用列表
再开一个终端窗口，输入
```
$ python dump.py -l
  PID  Name                     Identifier
-----  -----------------------  -------------------------------------
12317  App Store                com.apple.AppStore
 4563  信息                       com.apple.MobileSMS
12251  微信                       com.tencent.xin
...
```
#### 解密
这里以微信为例，微信的包名为 com.tencent.xin
```
$ python dump.py com.tencent.xin
```
执行完毕，从 dump.py 同级别目录里获取砸完壳的 App