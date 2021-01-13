---
title: iOS 键盘与剪切板的监控方案
date: 2021-01-04 21:00:00
tags:
    - 响应链
categories: 研究
toc: true
---

键盘与剪切板是用户使用频率最高的两个组件，本文主要探究的是键盘与剪切板的监控方案。

<!--more-->

## 键盘
能唤起键盘的方案有3种，归类如下
+ UITextField、UITextView 原生输入框
+ WebView 上的 INPUT 和 TEXTAREA
+ 非继承重写的 TextView，知名的有 YYKit 中的 YYTextView
  
为了下面方便介绍，将上述3种情况依次简称为情况一、情况二、情况三。

### 通知
对于原生的 UITextField 和 UITextView，可以在 UIKit.framework 下的头文件 UITextField.h 和 UITextView.h 中找到以下2个通知
```objectivec
// UITextField.h
UIKIT_EXTERN NSNotificationName const UITextFieldTextDidChangeNotification;
```
```objectivec
// UITextView.h
UIKIT_EXTERN NSNotificationName const UITextViewTextDidChangeNotification;
```
通过监听 UITextFieldTextDidChangeNotification 和 UITextViewTextDidChangeNotification 可以实现对原生 UI 控件输入事件的监控，代码如下
```objectivec
    /// monitor textField
    [[NSNotificationCenter defaultCenter] addObserverForName:UITextFieldTextDidChangeNotification object:nil queue:NSOperationQueue.mainQueue usingBlock:^(NSNotification * _Nonnull note) {
        UITextField *textField = note.object;
        NSLog(@"[%s] - textField：%@",__func__,textField.text);
    }];
    /// monitor textView
    [[NSNotificationCenter defaultCenter] addObserverForName:UITextViewTextDidChangeNotification object:nil queue:NSOperationQueue.mainQueue usingBlock:^(NSNotification * _Nonnull note) {
        UITextView *textView = note.object;
        NSLog(@"[%s] - textView：%@",__func__,textView.text);
    }];
```
但是上述的方案只能满足情况一，对于情况二和情况三就显得力不从心了。 
因此需要另辟蹊径。重新审视需求，发现三者有一个共同点，就是通过键盘进行输入。
因此接下来就从如何监听键盘点击事件入手。


### 响应链
从原理上分析，用户的点击其实就是一个响应链的过程，因此为了寻找新的解决方案，需要先介绍一下什么是响应链。
当一个点击事件产生后，会触发寻找事件接受者和触发事件响应这2个步骤。
#### 寻找事件接受者
+ 当一个触摸亊件生成时，系统会将其加入 UIApplication 的事件队列中。
+ UIApplication 会取出队列最前面的事件，通过 sendEvent: 方法分发到应用程序的主窗口 window。
+ 主窗口 window 会在当前视图层次结构中找到一个最合适的视图来处理触摸事件，具体流程如下
  + 调用当前视图的 pointInside:withEvent: 方法，判断触摸点是否在当前视图内。
  + 如果返回 NO，那么 hitTest:WithEvent: 就返回 nil。如果返回 YES，就继续遍历子视图，发送 hitTest:withEvent: 消息，直到有视图返回非空对象时返回该对象（或者在全局视图遍历完毕并都返回空对象时返回自身）。

#### 触发事件响应
+ 触发事件将沿着响应者链传递，传递规则如下，
    + 如果当前 view 是另一个 view 的子 view，那么它的父 view 就是下一个响应者。
    + 如果当前 view 是控制器的 view，那么控制器就是下一个响应者。
    + 如果在视图顶层还不能处理事件，那么就传给 window 对象处理。
    + 如果 window 对象也不能处理，则将其传给 UIApplication 对象。
    + 如果 UIApplication 对象也不能处理，就可能传给 UIAppDelegate 对象处理。
    + 如果都不能处理，事件将被丢弃。

### 破局
根据响应链流程中提及的公开的 API，可以对以下2个时机进行 hook，
+ 查找响应视图：`-[UIView hitTest:WithEvent:]`和`-[UIView pointInside:WithEvent:]`
+ 分发事件对象：`-[UIApplication sendEvent:]`

#### 查找响应视图
这2个是主窗口 window 在当前视图层次结构中找到一个最合适的视图来处理触摸事件的方法，因此对于一次 touch 事件，会被频繁调用的方法作为监控入口显然是不合适的。

#### 分发事件对象
-[UIApplication sendEvent:] 是分发事件给 window 的方法，在一个事件发生时只会触发一次，因此比较适合做监控入口。
先来看一下该方法的定义
```objectivec
// UIApplication.h
- (void)sendEvent:(UIEvent *)event;
```
根据响应链的定义，UIApplication 会取出队列最前面的事件，通过 sendEvent: 方法分发到应用程序的主窗口 window。
在 UIEvent.h 的头文件里找到了一个 set 集合 - allTouches，定义如下
```objectivec
// UIEvent.h
@property(nonatomic, readonly, nullable) NSSet <UITouch *> *allTouches;
```
通过以上属性找到的 UITouch 事件就是要找的 touch 对象。
由于上述的 sendEvent: 方法会响应所有的事件。因此，需要制定规则筛选出指定的 touch 事件。

#### 筛选 touch 规则
列出 UITouch.h 头文件中几个比较重要的属性
```objectivec
// UITouch.h
typedef NS_ENUM(NSInteger, UITouchPhase) {
    UITouchPhaseBegan,             // whenever a finger touches the surface.
    UITouchPhaseMoved,             // whenever a finger moves on the surface.
    UITouchPhaseStationary,        // whenever a finger is touching the surface but hasn't moved since the previous event.
    UITouchPhaseEnded,             // whenever a finger leaves the surface.
    UITouchPhaseCancelled,         // whenever a touch doesn't end but we need to stop tracking (e.g. putting device to face)
};
// 触摸状态
@property(nonatomic,readonly) UITouchPhase        phase;
// 处理事件的 window
@property(nullable,nonatomic,readonly,strong) UIWindow                        *window;
// 能响应触摸事件的 view
@property(nullable,nonatomic,readonly,strong) UIView                          *view;
```
打印一下点击键盘时 UITouch 对象的属性值，
```objectivec
(lldb) po [touch _ivarDescription]
<UITouch: 0x101710650>:
in UITouch:
	_phase (long): 3
	_window (UIWindow*): <UIRemoteKeyboardWindow: 0x103908800>
	_view (UIView*): <UIKeyboardDockView: 0x103a10720>
```
这里看到 window 为 UIRemoteKeyboardWindow，它是键盘事件响应的 window，因此对于键盘事件可以通过判断 window 是否为 UIRemoteKeyboardWindow 进行过滤
```objectivec
/// 是否为键盘事件
- (BOOL)isKeyboardWithTouch:(UITouch *)touch {
    // window 为UIRemoteKeyboardWindow，说明touch在键盘上
    return [touch.window isKindOfClass:NSClassFromString(@"UIRemoteKeyboardWindow")];
}
```

## 剪切板
对于剪切板，可以通过 +[UIPasteBoard generalPasteboard] 方法获取到系统的剪切板对象，当粘贴事件发生时，粘贴的内容会被写入到 UIPasteboard。手动写入粘贴内容到粘贴板代码如下
```objectivec
UIPasteboard.generalPasteboard.string = @“粘贴的内容”;
```
遗憾的是，当用户复制粘贴的时候，并没有调用上述方法，系统应该是通过更底层的 API 实现功能的。
重新回到监控剪切板的响应链方案，问题就转换为从众多的 touch 事件中筛选出剪切板工具栏上的按钮点击事件。

### 筛选 touch 规则
有了之前的经验，我直接打印点击工具栏时 touch 的属性值
```objectivec
(lldb) po [touch _ivarDescription]
<UITouch: 0x1080237d0>:
in UITouch:
	_phase (long): 3
	_window (UIWindow*): <UITextEffectsWindow: 0x105300f50>
	_view (UIView*): <UICalloutBarButton: 0x108025d20>
```
根据上述信息得出，点击剪切板工具栏的点击事件筛选条件如下
```objectivec
- (BOOL)isClipboardWithTouch:(UITouch *)touch {
    // view 为UICalloutBarButton且window为UITextEffectsWindow，说明touch在工具栏上
    return [touch.view isKindOfClass:NSClassFromString(@"UICalloutBarButton")] && [touch.window isKindOfClass:NSClassFromString(@"UITextEffectsWindow")];
}
```
如果需要确定用户的具体操作，还需要在这基础上在做过滤。
用户点击的事件触发在 UICalloutBarButton 上，而 UICalloutBarButton 显然是一个按钮。在 iOS 中，按钮允许 target-action 的方式对其进行监控。
点击剪切板上的 Copy 并打印 UICalloutBarButton 的属性值，发现 m_action 的结果如下，
```objectivec
(lldb) po [touch.view _ivarDescription]
in UICalloutBarButton:
	m_action (SEL): paste:
```
通过判断 m_action 的值即可知道用户在剪切板上的具体操作。
附剪切板工具栏上的复制、粘贴、剪切的 m_action 值。
```objectivec
copy:
paste:
cut:
```
## 总结
经过上述的探究以及分析，已经可以实现对键盘与剪切板的监控。
方案大致如下，
+ 通过 hook 分发事件方法 -[UIApplication sendEvent:] 找到点击事件的时机
+ 根据键盘和剪切板点击时的特征制定筛选条件
+ 对响应事件进行处理，回调
demo 的完整代码已经上传到 GitHub。
附上[demo地址](https://github.com/zeinber/ZBUserOperationMonitor)