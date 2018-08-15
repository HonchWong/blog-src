---
title: 通过 Crash Log 分析多线程下 Setter 的崩溃
date: 2018-04-25 10:21:02
tags: [Crash 分析, 多线程问题]
categories: Crash 分析
---

## 0x0 问题表现
某天实习导师想让我学习一下如何分析Crash Log并抛给了我一个Crash Log，那是我第一次分析Crash Log，Crash Log 是怎么生成的，有哪些重要信息可以帮助开发者解决问题等等关于Crash Log 的介绍就不在这篇文章讲了。（待完善）

这里直接贴上Crash Log的重要信息：

```
Exception Type: SIGSEGV
Exception Codes: SEGV_ACCERR at 0x00000001c8c6bec8 
Crashed Thread: 19

Thread 19 Crashed: 
0  libobjc.A.dylib                0x0000000192ab4174 objc_release + 16
9  libdispatch.dylib              0x00000001930e13ac __dispatch_call_block_and_release +  24
10 libdispatch.dylib              0x00000001930e136c __dispatch_client_callout +  16
3  libdispatch.dylib              0x00000001930ed40c __dispatch_root_queue_drain +  1152
14 libdispatch.dylib              0x00000001930ee75c __dispatch_worker_thread3 +  108
15 libsystem_pthread.dylib        0x00000001932bd2e4 _pthread_wqthread + 812
1  libsystem_pthread.dylib        0x00000001932bcfa8 __pthread_set_self + 12

Thread 19 crashed with ARM 64 Thread State:
     x0:  0x0000000172020a80    x1: 0x0000000141631530    x2: 0x0000000141631530     x3: 0x0000000000000010
     x4:  0x0000000000000010    x5: 0x0000000000000001    x6: 0x00000001831e1fec     x7: 000000000000000000
     x8:  0x00000001c8c6bea8    x9: 000000000000000000   x10: 0x00000001741f6d00    x11: 0x000000060000000f
    x12: 0x00000001741f6d80    x13: 0x000001a503064c7d   x14: 0x0000900d0add7186    x15: 0x000000000000008c
    x16: 0x0000000192ab4100    x17: 0x0000000100d5535c   x18: 000000000000000000    x19: 0x0000000172e78940
    x20: 0x0000000173e7f600    x21: 0x00000001872c9f26   x22: 0x00000001872c9f34    x23: 0x000000017444afb0
    x24: 0x0000000141631530    x25: 0x0000000000000008   x26: 0x000000017003e560    x27: 0x0000000102b12e78
    x28: 0x000000010300d000    fp: 0x000000010e3a3e40    lr: 0x0000000100688568    
    sp: 0x000000010e3a3d40     pc: 0x0000000192ab4174    cpsr: 0x20000000

Binary Images: 
0x1000b4000 - 0x1029cbfff +[项目名称] arm64 <abe5490e76c2398da44fcc8b42218a7b> /private/var/mobile/Containers/Bundle/Application/EC944360-8942-441B-869F-A338B3C659BF/QQReaderUI.app/QQReaderUI
0x192dc8000 - 0x192e99fff  libsqlite3.dylib arm64 <286839512b673f7c938aa79ac70bde15> /usr/lib/libsqlite3.dylib
0x181fe0000 - 0x18221efff  CoreData arm64 <33c0d795a45e35c9affed5cf9d83a8a1> /System/Library/Frameworks/CoreData.framework/CoreData
0x183124000 - 0x183378fff  Foundation arm64 <fb0544132648377c8d2683d597a3583d> /System/Library/Frameworks/Foundation.framework/Foundation
0x186ae8000 - 0x18745cfff  UIKit arm64 <31ac3f3fa5153620907fbfbfd1d671b0> /System/Library/Frameworks/UIKit.framework/UIKit
0x181c90000 - 0x181e9bfff  CFNetwork arm64 <68adcebf440d30769bd2d67adc7932a2> /System/Library/Frameworks/CFNetwork.framework/CFNetwork
0x182b24000 - 0x182be2fff  CoreMedia arm64 <af73ae8152763066a3fc18bcbcdecf94> /System/Library/Frameworks/CoreMedia.framework/CoreMedia
0x192a94000 - 0x192c90fff  libobjc.A.dylib arm64 <e6224d745a023588af8e5bb67498139a> /usr/lib/libobjc.A.dylib
0x182220000 - 0x18257cfff  CoreFoundation arm64 <83a9627362333366a8543e8c2d28166e> /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
0x1930e0000 - 0x193106fff  libdispatch.dylib arm64 <e19d74563485344485b6d4457a93e89d> /usr/lib/system/libdispatch.dylib
0x193108000 - 0x19310afff  libdyld.dylib arm64 <7387fcdff0b93cbcab450356edc79b67> /usr/lib/system/libdyld.dylib
0x193208000 - 0x193228fff  libsystem_kernel.dylib arm64 <c9a63d5247363e37917b9202cd05b13f> /usr/lib/system/libsystem_kernel.dylib
0x1932bc000 - 0x1932c4fff  libsystem_pthread.dylib arm64 <ddb19a684b2d3281a81fb6688c7e5a78> /usr/lib/system/libsystem_pthread.dylib
0x18b3c0000 - 0x18b3d3fff  GraphicsServices arm64 <b5078b39bd36372190e4ad5e7d991f68> /System/Library/PrivateFrameworks/GraphicsServices.framework/GraphicsServices
0x180cb4000 - 0x180d18fff  libAVFAudio.dylib arm64 <017d90360b443ae788ef31cfd73d17f6> /System/Library/Frameworks/AVFoundation.framework/libAVFAudio.dylib
0x1846d4000 - 0x184aeefff  MediaToolbox arm64 <0468767c75bb342cbbefc64bdf948be5> /System/Library/Frameworks/MediaToolbox.framework/MediaToolbox
> /System/Library/Frameworks/CoreFoundation.framework/CoreFoundation
0x1930e0000 - 0x193106fff  libdispatch.dylib arm64 <e19d74563485344485b6d4457a93e89d> /usr/lib/system/libdispatch.dylib

```
Crash 发生在Thread 19，通过Thread 19的调用栈看不到问题所在（待完善），来看一下Thread State 有什么可利用信息。lr: 0x0000000100688568 ，lr寄存器保存了方法调用完后返回的地址，来验证下这个地址是否在项目工程代码里，0x1000b4000 < 0x100688568 < 0x1029cbfff，lr的内容地址是在项目工程里的，离真相近了一步，再计算一下偏移地址 0x100688568 - 0x1000b4000 = 0x5D4568。拿到了偏移地址，下一节就去定位这个偏移地址看看问题所在。（具体原理待完善）

## 0x1 分析问题
这里通过使用hopper 看看在0x5D4568 发生了什么

![图1](http://r.photo.store.qq.com/psb?/V10JaO4w40EHz4/lKD6NI0*iY1b.xuOGmp8aPmPm78YVFkddWlF3WcJiak!/r/dEQBAAAAAAAA)

通过hopper定位到 lr 寄存器的内容指向下面这行代码 

``` 
mov x0, x24 
```

所以Crash 就发生在下面这行代码执行之后

 ``` 
 bl imp___stubs__objc_msgSend 
 ```

再看上面的汇编代码selector是啥，很明显是```setHeaderAdModel:```，回到项目工程里看到headerAdModel 是 nonatomic 的，在多线程下进行setter操作便可能对象被重复relaese的问题导致Crash，可以通过[runtime的源码](https://github.com/opensource-apple/objc4/blob/master/runtime/objc-accessors.mm)看到问题所在：

![](https://upload-images.jianshu.io/upload_images/2301467-f8758df065b63f00.png)

### FYI

- 一般来说 arm64上 x0 – x7 分别会存放方法的前 8 个参数。
- 一般在[self method]中，x0 存self地址，x1 存selector地址。
- 如果参数个数超过了8个，多余的参数会存在栈上，新方法会通过栈来读取。
- 方法的返回值一般都在 x0 上。
- 如果方法返回值是一个较大的数据结构时，结果会存在 x8 执行的地址上。

## 0x2 解决问题

对于多线程读写操作的问题，网上有各种解决方案，我还在学习中，这个小节待完善：）
