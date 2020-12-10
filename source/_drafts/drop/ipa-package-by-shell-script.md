---
title: Shell 脚本打 ipa 包
date: 2017-07-14   10:15:12
tags:
    - 打包
categories: Shell
---

在 iOS 开发中，我们经常需要上传 ipa 包。公司配置的电脑打包速度很慢（看机子和项目大小，反正公司配的苹果盒子很慢，而且每一步都要手点），打包时基本不能做任何其他事情（很卡），极大的浪费了时间。偶然间听说了 Shell 脚本可以帮我们很方便的解决这个问题，看了一篇文章之后，特此记录一下 Shell 打包的流程以及中间遇到的坑。

<!--more-->

## 准备工作

+ 准备要打包的项目，在苹果开发者网站上下载打包用到的证书，这里打测试包作为演示，就下载 adhoc 证书进行测试。下载 adhoc 证书并运行，然后在项目中选中 Targets -> General -> Signing ，勾选 Automatically manage signing ,把 team 选为该证书对应的开发者账号。
+ 下载[ReleaseDir](https://github.com/zeinber/ReleaseDir.git)，将 ReleaseDir 文件夹，放到跟所要打包的项目的根目录（ShellPackageDemo）同级别的目录下。
+ 打开 ReleaseDir 文件夹中的 ExportOptions.plist 文件，这里的四个选项是对包的设置。

```
**  ExportOptions.plist 文件参数说明 **
compileBitcode：不上架App Store，Xcode是否启用Bitcode重新编译，默认为YES。
method：归档类型，包括app-store、ad-hoc、package、enterprise、development以及developer-id。
uploadBitcode：上线App Store是否开启Bitcode，默认为YES。
uploadSymbols：上线App Store，是否开启符号序列化，这是与查crash相关的，默认为YES。
```

因此我们对 ExportOptions.plist 做如下设置：

![ExportOptions设置截图.png](http://upload-images.jianshu.io/upload_images/3983945-ee328ea11d172504.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
**在使用下列方法前，请先使用 Xcode 成功打包一次。（先让用户授权对 Xcode 使用钥匙串中的证书）**

## 调用方法

- 打开终端，cd 至 ReleaseDir 下。假如电脑之前装了 cocoapods (其他有切换过 ruby 环境的操作也算)，请先在终端运行：

```shell
rvm use system
```

将 ruby 切成系统的。

- 根据项目具体情况在终端运行下列对应的命令

```
./release.sh shellPackageDemo -w -e -v 1.0.0 -b 1.0.0    //使用了cocoapods
./release.sh shellPackageDemo -e -v 1.0.0 -b 1.0.0    //未使用cocoapods
```

```
调用格式:
参数说明：
<Project directory name>    第一个参数：所要打包的项目的根目录文件夹名称
-w                          workspace打包，不传默认为project打包
-s <Name>                   对应workspace下需要编译的scheme（不传默认取xcodeproj根目录文件名）
-e                          打包前是否先编译工程（不传默认不编译）
-d                          工程的configuration为 Debug 模式，不传默认为Release
-a                          打包，Version版本号自动＋1（针对多次打测试包时的版本号修改）
-b <Build Num>              Build版本号，指定项目Build号
-v <Version Num>            Version版本号，指定项目Version号
参数-a 与 -v 互斥，只能选择传其中之一
```

演示 demo 未使用 cocoapods ，因此运行：

```shell
./release.sh shellPackageDemo -e -v 1.0.0 -b 1.0.0
```

得到 ipa 包。

**运行结果截图**

![终端运行结果截图.png](http://upload-images.jianshu.io/upload_images/3983945-30e1f1b16b211d09.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![releaseDir目录截图.png](http://upload-images.jianshu.io/upload_images/3983945-7340ffbd3d971dcc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
