---
title: iOS 反调试以及反反调试总结
date: 2017-11-08 15:43:11
tags:
    - 调试
categories: 逆向
toc: true
---

应用开发者开发完 App，和苹果爸爸撕完逼，将 App 上架到 App Store。苹果爸爸会对应用进行一次加密，也就是我们俗称的壳。但是攻击者可以通过砸壳的方式将 App 进行破解，再通过对应用进行重签名，连上调试器，就可以快乐地进行动态调试。因此，我们需要知道如何进行反调试。
本文主要介绍关于反调试的原理及方案和对应的反反调试的方案。
<!--more-->

<!-- 反调试按逻辑分的话，主要有2种方式, 一种是直接阻止调试器附加, 另一种就是根据特征检测调试器是否附加。 --> 

## ptrace
ptrace 可以对另一个进程实现调试跟踪，以下是 ptrace.h 的头文件。
```c
#define PT_DENY_ATTACH  31
/**
 * @param request：请求 ptrace 执行的操作
 * @param pid：目标进程的 id，传 0 为当前进程
 * @param addr：目标进程的地址值
 * @param data：作用则根据 request 的不同而变化，如果需要向目标进程中写入数据，data 存放的是需要写入的数
据；如果从目标进程中读数据，data 将存放返回的数据
 * @return 标志位，0 为成功，非 0 为失败
 */
int ptrace(int _request, pid_t _pid, caddr_t _addr, int _data);
```
这里主要关注第一个入参 _request，它是一个枚举值，具体说明可在 ptrace.h 找到，传入 PT_DENY_ATTACH (31) 可以让系统阻止调试器附加。
### 反调试
#### 函数调用 ptrace
```c
// 可以通过申明的方式，这里为了省事，直接去 Mac 端找到头文件并导入
#import "ptrace.h"
#pragma mark - 直接调用 ptrace
/// 函数调用 ptrace
+ (void)antiDebugCheck_ptrace_01 {
    ptrace(PT_DENY_ATTACH, 0, 0, 0);
}
```

#### dlopen+dlsym 调用 ptrace
```c
#import "fishhook.h"
#import <sys/sysctl.h>
#import "ptrace.h"
#pragma mark - dlopen+dlsym 调用 ptrace
/// dlopen+dlsym 调用 ptrace
+ (void)antiDebugCheck_ptrace_02 {
    typedef int (*PTRACE_T)(int request, pid_t pid, caddr_t addr, int data);
    void *handle = dlopen(NULL, RTLD_GLOBAL | RTLD_NOW);
    PTRACE_T ptrace_ptr = dlsym(handle, "ptrace");
    ptrace_ptr(PT_DENY_ATTACH, 0, 0, 0);
}
```

### 反反调试
#### 函数调用 ptrace
```c
#import "fishhook.h"
#pragma mark - 函数调用 ptrace
typedef int (*ptrace_ptr_t)(int _request,pid_t _pid, caddr_t _addr,int _data);
static ptrace_ptr_t orig_ptrace = NULL;
int my_ptrace(int _request, pid_t _pid, caddr_t _addr, int _data);
int my_ptrace(int _request, pid_t _pid, caddr_t _addr, int _data){
    //request不是PT_DENY_ATTACH（31），执行原有调用，否则直接return 0
    if (_request != 31) {
        return orig_ptrace(_request,_pid,_addr,_data);
    }
    NSLog(@"⚠️⚠️⚠️ptrace_01 被成功绕过⚠️⚠️⚠️");
    return 0;
}

+ (void)antiDebugCrack_ptrace_01 {
    rebind_symbols((struct rebinding[1]){{"ptrace", my_ptrace, (void*)&orig_ptrace}},1);
}
```

#### dlopen+dlsym 调用 ptrace
```c
#import "fishhook.h"
#pragma mark - dlopen+dlsym 调用 ptrace
/// dlopen+dlsym 调用 ptrace
typedef void* (*dlsym_ptr_t)(void * __handle, const char* __symbol);
static dlsym_ptr_t orig_dlsym = NULL;
void* my_dlsym(void* __handle, const char* __symbol);
void* my_dlsym(void* __handle, const char* __symbol){
    ///查找 dlsym 的入参 __symbol 符号为 ptrace，直接 return 0
    if (strcmp(__symbol, "ptrace") != 0) {
        return orig_dlsym(__handle, __symbol);
    }
    NSLog(@"⚠️⚠️⚠️ptrace_02 被成功绕过⚠️⚠️⚠️");
    return my_ptrace;
}

#### syscall 调用 ptrace
查看头文件
​```c

+ (void)antiDebugCrack_ptrace_02 {
    rebind_symbols((struct rebinding[1]){{"dlsym", my_dlsym, (void*)&orig_dlsym}},1);
}

```

## sysctl
当一个进程被调试时，该进程中会有一个标记位来标记它正在被调试。可以通过 sysctl 函数査看当前进程的信息，通过是否有这个标记位即可检査当前调试的状态。

### 反调试




### 反反调试