---
title: frida
date: 2020-03-19 15:43:11
tags:
    - 工具
categories: 逆向
toc: true
---

`Frida` 是一个可以 `hook App` 的动态代码工具包，可以向 `Windows`、`macOS`、`Linux`、`iOS`、`Android` 和`QNX` 的本机应用程序中注入 `JavaScript` 或自己的库代码。

<!--more-->

## 算法原理
其实主要是通过了 ^ 这个运算符交换私钥实现的。
我们通过一个经典的场景，Alice 和 Bob 要在一条不安全的线路上交换秘钥，交换的秘钥不能被中间人知晓。
首先，双方约定使用ECDH秘钥交换算法，这个时候双方也知道了ECDH算法里的一个大素数p，这个p可以看做是一个算法中的常量。
### 操作步骤
手机连接到 MAC 上，然后打开 `控制台.app` , 在左侧列表里找到连接设备，右边列表里就会输出手机上的所有日志。在上方搜索栏里输入 `App的二进制名`，选择 `进程`，过滤出 App 的输出日志。

## 本地查看日志
要完成这个功能需要实现以下两步：
1. 写日志到沙盒
2. 沙盒导出文件

### 写日志到沙盒
Xcode 的日志显示原理其实是将日志写入到一个特定文件中存在本地，然后再显示到我们的 console 中。在真机环境上，如果脱离了 Xcode ，日志将被写入到手机的一个特定目录下。
我们要做到就是重定向文件流，让日志写到我们指定到目录下。要实现这一点，我们得先了解以下函数  freopen
#### freopen 函数
freopen 是被包含于 C 标准库头文件 <stdio.h> 中的一个函数，用于重定向输入输出流。该函数可以在不改变代码原貌的情况下改变输入输出环境，但使用时应当保证流是可靠的。定义如下：
```c
    FILE *freopen(const char *filename, const char *mode, FILE *stream)
```
**形参说明**: 
**filename**
需要重定向到的文件名或文件路径。
**mode**
代表文件访问权限的字符串。例如，r 表示只读访问、w 表示只写访问、a 表示追加写入、+表示读和写。
**stream**
需要被重定向的文件流。例如，stdin 表示标准输入流、stdout 表示标准输出流、stderr表示标准错误输出流

#### 实现原理
我们输出日志通常会调用 NSLog 和 printf。NSLog 内部定向的文件流是 stderr，而printf(...) 的实现等价于调用了fprintf(stdout, ...)，因此我们只需调用 freopen 重定向文件流 stderr 和 stdout，在合适的时机关闭文件流，让日志顺利写入到我们指定的沙盒文件中即可。重定向的代码如下：
```objc
/// 将log的输出信息写入到沙盒的Documents目录下，并输出为log文件
- (void)redirectLogToDocumentFolder {
        /// 存于Document文件夹
    NSArray *paths = NSSearchPathForDirectoriesInDomains(NSDocumentDirectory, NSUserDomainMask, YES);
    NSString *documentDirectory = [paths objectAtIndex:0];
    NSString *fileName = [NSString stringWithFormat:@"%@.log",[[NSDate alloc] initWithTimeIntervalSinceNow:8*3600]];
    NSString *logFilePath = [documentDirectory stringByAppendingPathComponent:fileName];
    /// 将log输入到文件 stdout是输出printf的内容，stderr是输出NSLog的内容
    freopen([logFilePath cStringUsingEncoding:NSASCIIStringEncoding],"a+",stdout);
    freopen([logFilePath cStringUsingEncoding:NSASCIIStringEncoding],"a+",stderr);
}
```

上述的方法一般在程序的入口调用，即：

```objc
- (BOOL)application(UIApplication *)application didFinishLaunchingWithOptions(NSDictionary *)launchOptions {
     /// 保存真机调试日志文件到沙盒 Documents 目录
     [self redirectLogToDocumentFolder];
}
```

**合适的写入时机**
stderr 的文件流是随时写入的，而 stdout 需要你找一个合适的时机关闭文件流写入，这里我们选择程序即将崩溃时。代码如下：
```objc
- (void)applicationWillTerminate:(UIApplication *)application {
    /// 关闭文件流，让日志最终写入到指定目录
    fclose(stdout);
}
```

### 查看日志文件

手机连接上 MAC，然后打开 Xcode ，根据路径 `Xcode -> Window -> Devices and Simulators` 进入到如下页面。

![Devices and Simulators](device-appcontainer-output.png)

在 `INSTALLED APPS` 中找到刚才输出日志的 App 包，点击左下角的齿轮键 ，在弹出菜单中选择 `Download Container...` ，就可以获取 App 沙盒信息，从 `Documents` 中找到刚才输出的日志。

## NSLog 的不足
在平时测试开发的时候我们发现，当 NSLog 的输出信息超过一定长度时，数据会被截断。为了弥补这一缺点，我们试着使用 printf 去打印。但是 printf 无法被 console 捕获，因此采用了方案二中的沙盒保存日志方案弥补这个不足。