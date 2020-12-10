---
title: 基于 runtime 的网络请求封装
date: 2017-06-15 12:10:08
tags:
    - runtime
categories: iOS
---

近期涉略了一些 runtime 的资料。学以致用，下面介绍如何用 runtime 封装网络请求。

<!--more-->

## 实现原理

runtime 有一个方法，可以去遍历一个类对象的所有属性。获取到属性名称以及对应属性的值。

```objc
  MyClass *myClass = [[MyClass alloc] init];//创建了类对象
  unsigned int outCount = 0;//记录类对象属性的个数
  Class cls = [myClass class];//获取类名
  objc_property_t* properties = class_copyPropertyList(cls, &outCount);//获取类的所有对象数组properties  outCount表示数组的元素个数
  for (int i = 0; i < outCount; i++) {//遍历properties数组
    objc_property_t property = properties[i];//类对象的每个属性
    const char* char_property_name =  property_getName(property);//转化成char类型
    if (char_property_name) {//判断是否获取成功
        NSString *property_name = [[NSString alloc] initWithCString:char_property_name encoding:NSUTF8StringEncoding];// 转换OC类型的字符串
    }
  }
  free(properties);//释放指针
```
由此可以得到一个启发：网络请求是可以通过类文件来管理的。

### 思路
所有网络请求的类文件都继承一个基类 BaseNetRequest ，然后在子类的 .h 中写上网络请求中需要的请求参数名，在 BaseNetRequest.m 文件中，通过上述 runtime 的方法获取网络请求参数，并在子类的 init 方法里做统一处理。

### 优点
- 网络请求可以通过文件的形式统一管理，方便开发者根据文件结构去寻找对应的请求。
- 需要做统一操作时，可以在 BaseNetRequest 文件中进行处理。

## 如何使用
### 项目目录结构
![目录结构](http://upload-images.jianshu.io/upload_images/3983945-973d62bc8ba9f415.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

### 示例代码
创建对应的请求类 TestNetRequest ，继承自 BaseNetRequest ，在 .h 中写上对应的请求参数。
![请求参数](http://upload-images.jianshu.io/upload_images/3983945-819d1cc828593aa8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

![调用步骤](http://upload-images.jianshu.io/upload_images/3983945-11b2f93904d7f46f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
### 调用结果

![调用结果log输出](http://upload-images.jianshu.io/upload_images/3983945-ac5f2296e92372cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

demo 中有详细的注释和使用方法，地址：[NetRequestDemo](https://github.com/zeinber/NetRequest_Demo.git)
