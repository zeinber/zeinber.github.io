---
title: 使用位段提高委托模式下的程序效率
date: 2017-03-15 09:10:11
tags:
    - 性能优化
categories: iOS
---

## 什么是位段？

位段 (bit-field) 是以位为单位来定义结构体(或联合体)中的成员变量所占的空间。含有位段的结构体称为位段结构。
优点：采用位段结构既能够节省空间，又方便于操作。

<!--more-->

拓展链接：[什么是位段？](http://www.jianshu.com/p/32a91972898a)

## 分析

在实现委托模式时，如果协议中的方法是可选的，经常需要写代码来判断某个委托对象是否能响应特定的选择子，那么就会出现下列代码：

```objc
if ([_delegate respondsToSelector:@selector(personDidSomething:)]) {
     [_delegate personDidSomething:something];
}
```

但是在委托对象本身没变的情况下，如果频繁执行此操作的话，那么除了第一次检测结果是有用之外，后续的检测可能都是多余的。在这里，可以把委托对象是否能响应某个协议方法这一信息缓存起来，以优化代码执行的效率。

```objc
@class Man;
@protocol ManDelegate <NSObject>
@optional
- (void)man:(Man)man playGame:(NSString *)game;
- (void)man:(Man)man eatFood:(NSString *)food;
@end
```

我们可以使用结构体来存储某个代理是否用 respondsToSelector 方法检测过。先在 Man 类下声明一个结构体：

```objc
@interface Man () {
    struct {
       unsigned int playGame : 1;
    }_delegateFlags;
}
```

在上述结构体中， playGame 位段占用 1 个二进制位，它可以表示 0 或 1 这两个值。我们可以通过下面的方法操作上述两个位段。

```objc
//set
 _delegateFlags.playGame  = 1;
//get
if (!_delegateFlags.playGame) {}
```

实现缓存功能所用的代码可以写在 delegate 属性所对应的设置方法里：

```objc
- (void)setDelegate:(id<ManDelegate>)delegate {
    _delegate = delegate;
    _delegateFlags.playGame = [delegate respondsToSelector:@selector(man:playGame:)];
}
```

这样的话，每次调用 delegate 的相关方法之前，就不用检测委托对象是否能响应给定的选择子了，而是直接查询结构体里的标识。

**优化前**：

```objc
if ([_delegate respondsToSelector:@selector(man:playGame:)]) {
     [_delegate man:self playGame:game];
}
```
**优化后**：

```objc
if (_delegateFlags.playGame) {
    [_delegate man:self playGame:game];
}
```

在相关代理方法需要调用多次时，这种缓存优化策略还是很有必要的。
