---
title:  iOS 那些被忽视的数据集合
date: 2016-10-01 10:20:12
tags:
    - 冷知识
categories: iOS
---

`Foudation` 框架中我们常用的数据集合类型有： `NSSet` 、 `NSDictionary` 、 `NSArray` 。实际上苹果在 iOS6 之后也推出过与之一一对应的 `NSHashTable` 、 `NSMapTable` 和 `NSPointArray` ，只不过因为前者功能较为强大，能解决平时开发中遇到的大部分问题，因此更容易被大家所熟知。

<!--more-->

### NSPointArray

在数组中添加一个对象时，会使得对象引用计数器 +1，被数组所持有。如果希望在数据容器中保持对对象弱引用，对象移除时，数组中也随之移除时，该如何处理呢？

在 iOS6 之前可以调用 `NSValue` 的 `valueWithNonretainedObject` 方法去弱化这个对象，然后在加到数据集合中可以达到上述要求。

```objc
NSValue *value = [NSValue valueWithNonretainedObject:obj];
NSArray *array = [NSArray arrayWithObject:value];
```

iOS6 之后可以使用 `NSPointArray` 来实现对应的要求。

```objc
///初始化方法
+ (NSPointerArray *)strongObjectsPointerArray;
+ (NSPointerArray *)weakObjectsPointerArray;
```

使用 `strongObjectsPointerArray` 之后得到的数组就是等同于 `NSMutableArray` ，数组对对象的引用是强引用。
使用 `weakObjectsPointerArray` 后得到的数组对对象的持有是弱引用。

因此这样写就能满足刚才的需求：

```objc
NSPointerArray *array = [NSPointerArray weakObjectsPointerArray];
[array addPointer:obj];
```

### NSHashTable

`NSHashTable` 是 `NSSet / NSMutableSet` 的通用版本。

`NSSet / NSMutableSet` 持有成员的强引用，通过 `hash` 和 `isEqual:` 方法来检测成员的散列值和相等性。

`NSHashTable` 具有下面这些特性：

- 可变的，没有不可变的对应版本
- 可以持有成员的弱引用
- 可以在加入成员时进行 copy 操作
- 可以存储任意的指针，通过指针来进行相等性和散列检查

### NSMapTable

`NSMapTable` 是 `NSDictionary` 的通用版本。

`NSDictionary / NSMutableDictionary` 对键进行拷贝，对值持有强引用。

`NSMapTable` 具有下面这些特性：

- 可变的，没有不可变的对应版本
- 可以持有键和值的弱引用，当键或者值当中的一个被释放时，整个这一项就会被移除掉
- 可以在加入成员时进行 copy 操作
- 可以存储任意的指针，通过指针来进行相等性和散列检查
