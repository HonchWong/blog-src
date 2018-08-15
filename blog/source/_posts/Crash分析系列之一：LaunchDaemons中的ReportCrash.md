---
title: Crash分析系列之一：LaunchDaemons中的ReportCrash
date: 2018-05-05 16:05:37
tags: [Crash 分析, LaunchDaemons, ReportCrash]
categories: Crash分析系列
---

很多公司都有一套 Crash收集、上报、统计的机制，接入这套Crash机制后，开发者平时只需关心Crash Report的内容就好了，但我最近有时间，便来学习下Crash收集、上报、统计的整个过程，并用Crash 分析系列文章记录学习，本文是系列第一篇《Crash分析系列之一：LaunchDaemons中的ReportCrash》。

Crash 分析系列打算一共写五篇：

- 《Crash分析系列之一：LaunchDaemons中的ReportCrash》
- 《Crash分析系列之二：Mach异常、BSD信号》
- 《Crash分析系列之三：Crash收集、符号化、上传》
- 《Crash分析系列之四：Crash分析之ARM汇编基础》
- 《Crash分析系列之五：Crash分析实践》

(注：文中参考的Darwin xnu tag 是4570.1.46）

## 0x0 iOS 生成的Crash Report

其实 Apple也为iOS提供了Crash 收集的机制，对于收集到的Crash 日志，可以通过几种方式查看：

- iPhone 用户直接在“设置-隐私-分析-分析数据”中查看.
- 如果iPhone用户在设置中允许与开发者共享分析数据的话，开发者便可以从 iTunes Connect 或是 Xcode 中查看开发者账号下App对应的崩溃上报。

### iOS 的 Crash Report有局限性

iOS 收集的Crash Log对开发者不友好：

- Crash Log 上报决策权在用户。
 
	iPhone 用户如果不和开发者共享分析数据的话，只能拿到iPhone 设备才可以导出Crash Report（通过Xcode -> Window -> Devices and Simulators -> Devices -> View Device Logs）。
- 即使拿到了Crash Log 还需要进行符号化。

	不符号化会增加定位问题代码的困难，比如没有符号化只有十六进制地址的话，但只要能拿到问题代码的地址偏移量，就可以使用lldb的 image lookup -address 命令 或者使用Hopper 等反汇编工具来定位代码，显然不如直接根据符号在项目工程里查找块。

### iOS 的 Crash Report不可缺失

可见使用第三方Crash 收集框架的重要性，但第三方Crash收集框架却对某些Crash束手无策，hockeyapp的官网上有篇[文章](https://support.hockeyapp.net/kb/client-integration-ios-mac-os-x-tvos/which-types-of-crashes-can-be-collected-on-ios-and-os-x)写了哪些 Crash可以被收集哪些不行，文章中提到：

>
Crashes that are actually kills by the iOS system, and therefor can not be detected:
>
- Crashes caused by low memory situations
>
	A low memory warning is generated when the app was killed by the system because there was not enough memory to satisfy the app’s demands. 
>	
- Crashes caused by timing issues (Watchdog timeouts)
>
	A watchdog timeout is generated when an app takes too long to launch, terminate, or respond to system events.

因为三方的Crash 收集框架在对应的应用进程中建立 handler 来记录应用行为，但如果操作系统从外部终止进程，这个 handler 就永远无法执行了。下面就举一个具体的例子，例子所用的Crash Log 是我从iPhone 的“分析数据”中随便找了一个，由于Crash Log的分析不是本文的重点，我只截取其中的 Exception Information（[完整版](https://github.com/HonchWong/blog-src/blob/master/zhuanzhuan-2018-05-01-124141.ips)已上传github）：

```
Exception Type:  EXC_CRASH (SIGKILL)
Exception Codes: 0x0000000000000000, 0x0000000000000000
Exception Note:  EXC_CORPSE_NOTIFY
Termination Reason: Namespace ASSERTIOND, Code 0x8badf00d
Triggered by Thread:  0
```

- Exception Type: 在 `` EXC_CRASH (SIGKILL) `` 中
	- `` EXC_CRASH `` 为Mach层的异常类型，定义在[darwin-xnu的/osfmk/mach/exception_types.h](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/osfmk/mach/exception_types.h)，`` EXC_CRASH `` 表示进程异常退出了。通常是因为未捕获的Objective-C/C++的异常导致进程被终止，这时BSD层的信号应该为 SIGABRT，Exception Type为 `` EXC_CRASH (SIGABRT) ``
	- 后者为 BSD层的信号，定义在[darwin-xnu的/bsd/sys/signal.h](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/bsd/sys/signal.h)，SIGKILL表示进程被系统终止，而且这个信号不能被阻塞、处理和忽略。这时可以查看Termination Reason字段了解终止的原因。
	- （Mach层异常和BSD层的信号的区别和关系在系列其他文章中讨论）
- Exception Codes: 这个字段一般用不上，当崩溃报告包含一个未命名的异常类型时，这个异常类型将用这个字段表示，形式是十六进制值。
- Exception Note: `` EXC_CORPSE_NOTIFY `` 和 `` EXC_CRASH ``定义在同一个文体中，意思是进程异常进入 CORPSE状态。
- Termination Reason: 这里主要关注 Code 0x8badf00d，可以[在Apple 文档](https://developer.apple.com/library/content/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-EXCEPTION_INFO)中查看到 0x8badf00d 表示进程因为watchdog 超时而被iOS 结束进程。

	关于watchdog：为了避免应用陷入错误状态导致界面无响应，Apple 设计了看门狗 (WatchDog) 机制，一旦超时，强制杀死进程，比如说应用启动时在主线程进行同步网络请求操作，超时系统就会杀死进程。在不同的生命周期，触发看门狗机制的超时时间有所不同：（注：调试模式下看门狗机制处于禁用状态。）
	
| 生命周期       | 超时时间      | 
| ------------- |:-------------:| 
| 启动 Launch	     | 	20 s | 
| 恢复 Resume     | 	10 s      | 
| 悬挂 Suspend | 	10 s      | 
| 退出 Quit | 	6 s      | 
| 后台 Background | 		10 min     | 
	
- Triggered by Thread：异常发生所在线程。

在 Exception Type为 EXC_CRASH (SIGKILL) 的 Exception中，Termination Reason中的code 除了上面的 0x8badf00d还有其他种可能，比如 0xdead10cc 。0x8badf00d 和 0xdead10cc 都是一种使用十六进制表示的英语单词拼写，0x8badf00d表示"ate bad food"，是想说看门狗吃坏肚子所以结束进程？？而0xdead10cc 表示 "dead lock" ，[在Apple 文档](https://developer.apple.com/library/content/technotes/tn2151/_index.html#//apple_ref/doc/uid/DTS40008184-CH1-EXCEPTION_INFO)中说明0xdead10cc 是因为进程在挂起期间持有文件锁或sqlite数据库锁，如果应用在挂起时对锁定了的文件或sqlite数据库执行操作，则必须请求额外的后台执行时间才能完成这些操作，并在挂起之前放弃锁定。


## 0x1 iOS 是怎么生成 Crash Report的

iOS的 Crash Report其实是由后台的守护程序（daemon）ReportCrash 来生成的，ReportCrash在iOS 目录中的/System/Library/CoreServices/ReportCrash ，而这个ReportCrash 是在什么时候启动的呢，怎么进行Crash 收集？

### iOS 启动流程

一般计算机系统的启动分为前后两个过程，先是底层硬件固件程序的运行以加载操作系统的内核，后是操作系统接管之后的相关进程启动过程。iOS启动流程可以大致分为

``` 
引导ROM > LLB > iBoot > 加载内核 > 启动launchd > 启动守护程序和代理程序
```

这里就不展开整个流程，而只关心 `` 加载内核 > 启动launchd > 启动守护程序和代理程序 `` 。

1. 在内核引导的过程中，kernel_bootstrap([osfmk/kern/startup.c](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/osfmk/kern/startup.c)) 函数主要负责设置和初始化 Mach 内核的核心子系统，比如：

	- `` kernel_bootstrap ``里调用`` vm_mem_bootstrap ``(osfmk/vm/vm_init.c)，该函数执行大量初始化函数，设置虚拟内存。
	- `` kernel_bootstrap ``里还调用`` sched_init `` (osfmk/kern/sched_prim.c)来初始化调度器子系统
	- `` kernel_bootstrap `` 还初始化了Mach的一些关键抽象：IPC、clock、ledger、task、thread
	- `` kernel_bootstrap ``最后会正式创建第一个线程`` kernel_bootstrap_thread `` ，并加载`` kernel_bootstrap_thread ``线程的上下文成为这个线程，这个线程接管初始化的工作，处理更负责的子系统（意味着kernel_bootstrap函数没有返回）

2. `` kernel_bootstrap_thread ``线程中初始化了IOKit、BSD子系统等等，其中需要关注的是初始化BSD子系统，调用的函数是 `` bsd_init() ``([bsd/kern/bsd_init.c](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/bsd/kern/bsd_init.c))，XNU整个BSD层的初始化都是由这个`` bsd_init() ``进行的。

3. 在`` bsd_init() ``快要结束的时候调用 `` bsd_utaskbootstrap() ``(定义在[bsd/kern/bsd_init.c](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/bsd/kern/bsd_init.c)) ，这个函数负责间接启动PID 1，这是第一个要进入用户态的进程，在`` bsd_utaskbootstrap() ``里首先调用 `` cloneproc() `` 函数返回得到一个 thread，然后用这个thread作为参数执行`` act_set_astbsd(thread) ``，执行后产生一个AST（异步系统陷阱），Mach的 AST异步处理程序会特别处理这个情况：调用`` bsd_ast() ``（定义在[bsd/kern/kern_sig.c](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/bsd/kern/kern_sig.c)），`` bsd_ast() `` 调用 `` bsdinit_task() ``（定义在[bsd/kern/kern_sig.c](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/bsd/kern/bsd_init.c)）。


4. 在`` bsdinit_task() ``中将PID 1的进程名字设置为"init"。
	- 接下来调用`` ux_handler_init() ``，在这个调用里面创建一个独立的内核线程`` ux_handler ``负责处理UNIX异常——也就是在一个全局的`` ux_exception_port ``端口上接受消息。
	- 接下来就是注册"init"线程的异常端口，将全局端口`` ux_exception_port ``注册为自己的端口，这样就可以保证所有的UNIX 进程的所有UNIX 异常都会被这个`` ux_handler ``线程处理。（因为后面所有的进程都是PID 1的"init"进程的后代）
	- 最后调用`` load_init_program() ``，这个函数里调用`` load_init_program_at_path() ``， `` load_init_program_at_path() `` 里将调用`` execve() ``执行launchd程序（文件在/sbin/launchd），launchd被设计成只能这种形式执行，用户没有权限去进行手动启动，但可以使用launchctl命令来和launchd进行交互，借此可以控制后台守护程序的启动或终止。（注：《》指出load_init_program() 把PID 1的"init"进程变成了launchd，但我还没完全搞懂这个逻辑）

### launchd 

launchd 一个很重要的职责就是派生各种各样的后台守护程序和代理程序。

> 
> - 守护程序（daemon）在启动时运行，是后台服务，通常和用户没有交互，不考虑是否有用户登录进系统（OS X上）。
> - 代理程序（agent）是一类特殊的守护程序，只有用户登陆的时候才启用，可以和用户交互，有的程序还有GUI。

由于iOS不需要登录，所以只有一个系统范围的launchd，并且它是系统运行期间唯一不能终止的进程，当系统关闭时，它作为最后一个进程退出。

launchd是怎样来启动这些守护程序的呢？其原理是，launchd通过查看特定文件夹中的plist属性文件，根据这些plist文件来决定启动哪些守护程序。这几个特定的文件夹目录路径见以下表格：


| 目录       | 用途      | 
| ------------- |:-------------:| 
| /System/Library/LaunchDaemons     | 	存放属于系统本身的守护程序plist文件 | 
| /Library/LaunchDaemons    | 	存放第三方程序的守护程序plist文件      | 
| /System/Library/LaunchAgents（iOS没有这个目录） | 	存放属于系统本身的代理程序plist文件     | 
| /Library/LaunchAgents | 存放第三方程序的代理程序plist文件，通常为空     | 
| ~/Library/LaunchAgents | 		用户自有的launch代理程序，只有对应的用户才会执行     | 
	
[在这可以看到iOS的 /System/Library/LaunchDaemons 下面都有哪些plist文件](https://www.theiphonewiki.com/wiki//System/Library/LaunchDaemons)，可以看到其中有 com.apple.ReportCrash.plist和com.apple.ReportCrash.SafetyNet.plist，这两个plist文件内容如下：
 
 com.apple.ReportCrash.plist
 
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Disabled</key>
	<false/>
	<key>Label</key>
	<string>com.apple.ReportCrash</string>
	<key>Program</key>
	<string>/System/Library/CoreServices/ReportCrash</string>
	<key>MachServices</key>
	<dict>
		<key>com.apple.ReportCrash</key>
		<dict>
			<key>ExceptionServer</key>
            <dict></dict>
		</dict>
	</dict>
    <key>MachExceptionHandler</key>
    <string>com.apple.ReportCrash.SafetyNet</string>
</dict>
</plist>
```

com.apple.ReportCrash.SafetyNet.plist

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple Computer//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
	<key>Disabled</key>
	<false/>
	<key>Label</key>
	<string>com.apple.ReportCrash.SafetyNet</string>
	<key>MachServices</key>
	<dict>
		<key>com.apple.ReportCrash.SafetyNet</key>
		<true/>
	</dict>
	<key>ProgramArguments</key>
	<array>
		<string>/System/Library/CoreServices/ReportCrash</string>
		<string>-s</string>
	</array>
    <key>MachExceptionHandler</key>
    <true/>
</dict>
</plist>
```

[Apple 的官文档有给出关于launchd.plist的解释](https://developer.apple.com/legacy/library/documentation/Darwin/Reference/ManPages/man5/launchd.plist.5.html)，

- Program是指launchd要执行的程序是哪个。
- MachServices里的Value是个字典，这个字典中的每个key都应该是要公布的服务的名称，在Mach引导子系统时用这个key来指定注册的Mach服务，这个字典中的value 可以是一个布尔值并设置为true，或者使用字典来代替简单的真实值。

launchd 是xnu的一部分，所以它也是开源的，但我还没仔细看launchd 是怎么把ReportCrash 设置为MachExceptionHandler 的。但上文提到，初始化BSD子系统的时候，在`` bsdinit_task()`` 注册了异常端口和异常处理线程，而所有的进程launchd的子进程，也就是任何一个进程发生异常时，ReportCrash会自动根据需要而进行 Crash收集。

## 0x2 by the way

### tell launchd to launch Crash Reporter
在xnu源码中找到一个函数定义（文件在[osfmk/kern/exception.c](https://github.com/apple/darwin-xnu/blob/0a798f6738bc1db01281fc08ae024145e84df927/osfmk/kern/exception.c)）,但还没搞懂是哪里调了这个函数，然后这个函数去通知 launchd 启动Crash Reporter

```
/*
 * Raise an exception on a task.
 * This should tell launchd to launch Crash Reporter for this task.
 */
kern_return_t task_exception_notify(exception_type_t exception,
	mach_exception_data_type_t exccode, mach_exception_data_type_t excsubcode)
{
	mach_exception_data_type_t	code[EXCEPTION_CODE_MAX];
	wait_interrupt_t		wsave;
	kern_return_t kr = KERN_SUCCESS;

	code[0] = exccode;
	code[1] = excsubcode;

	wsave = thread_interrupt_level(THREAD_UNINT);
	kr = exception_triage(exception, code, EXCEPTION_CODE_MAX);
	(void) thread_interrupt_level(wsave);
	return kr;
}

```

### classdump iphoneheaders

github找到个[仓库](https://github.com/ichitaso/iOS10.2-iphoneheaders)存了iOS 一些私有Framwork、内置App的header文件，比如SpringBoard.app的，还有本文在讨论的ReportCrash，在文件夹/System/Library/CoreServices/ReportCrash 下面可以看到 CrashReport、JetsamReport 这两个类都是继承AppleErrorReport ，并且都遵守 <ConcreteReport> 协议，看了一下协议ConcreteReport

```
@protocol ConcreteReport <NSObject>
@optional
-(id)overrideFileExtension;
-(id)additionalIPSMetadata;
-(BOOL)isActionable;

@required
-(id)reportNamePrefix;
-(id)appleCareDetails;
-(void)generateLogAtLevel:(BOOL)arg1 withBlock:(/*^block*/id)arg2;
-(id)problemType;

@end

```

我猜生成Crash Report的方法是这个吧～ `` -(void)generateLogAtLevel:(BOOL)arg1 withBlock:(/*^block*/id)arg2; `` ，后面有时间可以逆向看下 CrashReport的实现，但还是先看下开源的第三方Crash 收集框架吧～


也就是说0x2 的内容纯属记录一下坑。


	
