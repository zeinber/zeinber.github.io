---
title: 基于 CADisplayLink 的 FPS 指示器
date: 2018-02-21 22:30:15
tags:
    - 冷知识
categories: iOS
---

## CADisplayLink

CADisplayLink 是一个频率和 iOS 设备刷新频率(60HZ)相同的定时器。

<!--more-->

**重要属性**

- frameInterval

  NSInteger 类型的值，用来设置间隔多少帧调用一次 selector 方法，默认值是1，即每帧都调用一次。


- duration

  readOnly 的 CFTimeInterval 值，表示两次屏幕刷新之间的时间间隔。需要注意的是，该属性在 target 的 selector 被首次调用以后才会被赋值。

  selector 的调用间隔时间计算方式是：间隔时间 = duration × frameInterval。

**使用场景**

CADisplayLink 适合做界面的不停重绘。

- 在屏幕刷新时使用

  CADisplayLink 以特定模式注册到 runloop 后，每当屏幕显示内容刷新结束，runloop 就会向 CADisplayLink 指定的 target 发送一次指定的 selector 消息，即它对应的 selector 就会被调用一次。

- 延迟

  iOS设备的屏幕刷新频率是固定的，CADisplayLink 在正常情况下会在每次刷新结束都被调用，精确度相当高。但如果调用的方法比较耗时，超过了屏幕刷新周期，就会导致跳过若干次回调调用机会。

  如果 CPU 过于繁忙，无法保证屏幕原有的刷新率，就会导致跳过若干次调用回调方法的机会，跳过次数取决于 CPU 的忙碌程度。

## 具体实现

### 思路

既然 CADisplayLink 可以以屏幕刷新的频率调用指定 selector，而 iOS 系统中正常的屏幕刷新率为60Hz，那么只要在这个方法里面统计指定次数所花的时间，然后通过

```
屏幕FPS = 次数 / 时间
```
就可以得出当前屏幕的刷新频率了。

### 代码

```objc
- (void)setupDisplayLink {
   //创建CADisplayLink，并添加到当前run loop的NSRunLoopCommonModes
   _displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(linkTicks:)];
   [_displayLink addToRunLoop:[NSRunLoop currentRunLoop] forMode:NSRunLoopCommonModes];
}

- (void)linkTicks:(CADisplayLink *)link {
   //执行次数
   _scheduleTimes ++;
   //当前时间戳
   if(_timestamp == 0){
       _timestamp = link.timestamp;
   }
   CFTimeInterval timePassed = link.timestamp - _timestamp;
   if(timePassed >= 1.f){
       //fps
       CGFloat fps = _scheduleTimes/timePassed;
       NSLog("fps:%.1f, timePassed:%f\n", fps, timePassed);
       //reset
       _timestamp = link.timestamp;
       _scheduleTimes = 0;
   }
}
```

具体见 [Demo](https://github.com/zeinber/ZBFPSLabel_Demo)
