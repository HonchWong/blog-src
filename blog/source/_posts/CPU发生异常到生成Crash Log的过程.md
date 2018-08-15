---
title: CPU发生异常到生成Crash Log的过程
date: 2018-08-15 22:52:46
tags: [Crash 分析]
categories: Crash分析系列
---

#### 什么是异常

很多介绍操作系统的书在讲解**“操作系统的运行机制”**的时候都会提到**“现代操作系统是靠中断驱动的软件”**，这句话怎么理解？

中断是指CPU对系统发生的某个事件做出的一种反应，CPU暂停正在执行的程序，保留现场后转去执行相应的处理程序，处理完该事件后再返回断点继续执行被“打断”的程序。

而引入中断技术的初衷是提高多道程序运行环境中CPU的利用率，比如CPU可以在I/O的执行过程中去执行其他指令，不用空闲地去等待(或简单轮询) I/O设备的执行完成，I/O设备执行完成再通过中断通知CPU，以提高CPU利用率。后来中断技术逐步发展，成为操作系统各项操作的基础，比如进程调度，现代操作系统的进程调度一般都是采用基于时间片的优先级调度算法，把CPU的时间划分为很细粒度的时间片，执行一个任务的时间片用完了，时钟通过时钟中断去通知CPU切换任务，再比如下面要讨论到的CPU异常处理，也是基于中断机制去完成的。

中断(interrupt)和异常(exception)在不同的CPU架构里有不同的含义。

- 比如在Intel架构中，中断处理的入口由操作系统内核中的中断分配表定义(interrupt dispatch table, IDT)，IDT中有255个中断向量，其中前20个定义为异常(exception)的处理入口，即中断包含异常。
- 而在ARM架构中，中断处理的入口则是在异常向量(exception vector)中，8个异常向量里边有3个是中断相关的，即异常包含中断。

不管如何界定中断和异常，CPU发生异常时，都会将控制权从异常前的程序交给异常处理程序，而且CPU将获得 ***不会更低*** 的执行权利，比如执行用户态的应用程序发生异常，CPU将切换到内核态，并执行对应的异常处理程序。经典的CPU五级流水线中一条指令的生命周期为[取指、译码、执行、访存、写回]，每个阶段都可能出现CPU异常，比如在ARM架构下：

- 在“执行”阶段产生的“数据中止”异常：若处理器 ***数据访问指令*** 的地址不存在，或该地址不允许当前指令访问时，产生数据中止异常。
- 在“取指”阶段产生的”预取中止“异常：若处理器 ***预取指令*** 的地址不存在，或该地址不允许当前指令访问，存储器会向处理器发出中止信号，但当预取的指令被执行时，才会产生指令预取中止异常。

这两种异常对应的处理程序会直接或者间接调用 Mach 内核的 `` exception_triage() `` 函数，并将 `` EXC_BAD_ACCESS `` 作为入参传进去，`` exception_triage() `` 将会利用Mach消息传递机制投递异常。尽管Intel架构和ARM架构的CPU异常处理有些不同，但异常处理程序都会直接或间接将异常类型(`` exception_type_t ``)传给`` exception_triage() ``函数来处理异常，以此来屏蔽不同机器平台异常处理的差异。

异常类型(`` exception_type_t ``)在Mach层用int变量来存储，在`` osfmk/mach/exception_types.h ``文件中能看到Mach层定义的十几种异常，如常见的

```
#define EXC_BAD_ACCESS      1   /* Could not access memory */
        /* Code contains kern_return_t describing error. */
        /* Subcode contains bad memory address. */
#define EXC_CRASH           10  /* Abnormal process exit */
#define EXC_CORPSE_NOTIFY   13  /* Abnormal process exited to corpse state */
```

```
int main(int argc, const char * argv[]) {
    int *pi = (int*)0x00001111;
    *pi = 17;
    return 0;
}
```

上面这个程序中的非法内存访问将会用到上面列举三个异常类型，下面通过看源码、看书、代码调试来看下``  exception_triage()`` 函数都做了什么。


#### 调试跟踪CPU异常

在《深入解析Mac OS & iOS 操作系统》中有讲解xnu异常处理的过程，但不是特别详细，而且书的参考代码与最新代码也有出入，要把内核的异常处理流程弄清楚，需要看书、看源码，当然少不了断点调试。

##### 调试xnu


在MacOS上调试XNU要比在iOS上调试简单，使用到的工具是：LLDB + VMware Fusion + Kernel Debug Kit ，调试环境的搭建只需简单几个步骤即可，可参考 [《MacOS内核调试环境搭建》](http://www.cnblogs.com/elvirangel/p/9096517.html) ，iOS上的调试可以参考lan beer 分享的 [build your own iOS kernel debugger](https://bugs.chromium.org/p/project-zero/issues/detail?id=1417#c16)，链接里有分享的PPT和PoC ，可惜目前的Poc仅支持iOS 11.1.2 

这里记录个在MacOS上调试XNU的坑，如果虚拟机到达“wait for the debugger” 阶段，并且在主机通过“kdp-remote” 连接虚拟机成功，但虚拟机继续启动的过程中一直卡在“Waiting for link to become available”，导致调试无法继续，就像[这个帖子](https://forums.developer.apple.com/thread/35464)中描述的问题一样

虽然我也没找到问题的具体原因，但摸出了个解决办法，就是在虚拟机启动时同时按下Option、Command、P 和 R，以reset NVRAM，将会进入到恢复模式，使用终端工具关闭虚拟机的SIP ，即输入命令csrutil disable，然后重启，启动后再走一遍 “内核替换”-> "设置boot-args" -> "清除kext缓存" -> "重启虚拟机" -> "主机连接虚拟机" 的流程，这时将会有百分之七十的概率能让虚拟机正常启动并可调试，如果不行就再试一次。

注：我使用的MacOs 版本是10.13.5，对应的XNU是4570.61.1，对应版本的源码没有放出，对比了前几个版本，我需要参考的源码都没有变动，所以参考源码是github上的[xnu-4570.1.46](https://github.com/apple/darwin-xnu/)

##### 跟踪CPU异常

```
int main(int argc, const char * argv[]) {
    char c = getchar();
    int *pi = (int*)0x00001111;
    *pi = 17;
    return 0;
}
```

首先使用gcc来把上面这个程序编译成二进制可执行程序，然后运行。在程序等待键盘输入的时候，可以用ps命令查看进程PID。

todo 图片

在运行程序之前我在`` osfmk/kern/exception.c `` 的 `` exception_triage_thread() `` 函数实现处打了三个断点

```
breakpoint set --file exception.c --line 447
breakpoint set --file exception.c --line 459
breakpoint set --file exception.c --line 472
```
447、459、472 分别是往 thread 层、task 层、host 层的异常端口数组投递异常，对应以下三行代码

```
（447）kr = exception_deliver(thread, exception, code, codeCnt, thread->exc_actions, mutex);
（459）kr = exception_deliver(thread, exception, code, codeCnt, task->exc_actions, mutex);
（472）kr = exception_deliver(thread, exception, code, codeCnt, host_priv->exc_actions, mutex);
```

这三个断点只有一个断住了，那就是第472 行代码，到这里可以验证以下结论

首先通过lldb在终端输出函数调用栈、线程状态、进程PID

```
(lldb) bt
* thread #1, stop reason = breakpoint 4.1
  * frame #0: 0xffffff800f97f0c9 kernel.development`exception_triage_thread(exception=1, code=0xffffff8014debf50, codeCnt=2, thread=0xffffff801c7c2a10) at exception.c:472 [opt]
    frame #1: 0xffffff800fad71fb kernel.development`user_trap [inlined] exception_triage(code=0x0000000000000001) at exception.c:504 [opt]
    frame #2: 0xffffff800fad71df kernel.development`user_trap [inlined] i386_exception(exc=1, code=<unavailable>) at trap.c:1152 [opt]
    frame #3: 0xffffff800fad71d7 kernel.development`user_trap [inlined] user_page_fault_continue(kr=<unavailable>) at trap.c:232 [opt]
    frame #4: 0xffffff800fad71d1 kernel.development`user_trap(saved_state=0xffffff8017246b20) at trap.c:1093 [opt]
    frame #5: 0xffffff800f921102 kernel.development`hndl_alltraps + 226
(lldb) e struct proc *$p_proc = (struct proc *)thread->task->bsd_info
(lldb) po $p_proc->p_pid
352
(lldb) po thread->state
4

```

```
（注：线程状态用int变量存储，int state ，#define TH_SUSP 0x02 /*停止，或请求停止*/)
```

以上log结合源码和《深入解析Mac OS & iOS 操作系统》可以得出结论：

在Intel架构上，CPU执行用户态程序发生异常时会将对应进程挂起，并将CPU工作状态设置为内核态，还将执行XNU内核的异常处理程序。大多数操作系统都不会为每一个陷阱(异常)设置独立的处理程序，而是为所有的陷阱设置一个处理程序，然后这个处理程序通过`` switch() ``进行不同的处理，或者根据预定义的表跳转到不同的函数。XNU的做法也是如此，`` hndl_alltraps ``是公共陷阱处理程序，`` user_trap ``负责处理实际的陷阱，`` hndl_alltraps ``是用汇编语言写的，而`` user_trap `` 是用C语言写的，在`` user_trap `` 的实现里会调用`` i386_exception ``函数 ，`` i386_exception ``函数会调用`` exception_triage ``将陷阱转换为Mach 异常，在上面的程序中Mach 异常是 `` EXC_BAD_ACCESS `` 。

`` exception_triage() ``函数的实现只有两行代码

```
kern_return_t
exception_triage(
    exception_type_t    exception,
    mach_exception_data_t   code,
    mach_msg_type_number_t  codeCnt)
{
    thread_t thread = current_thread();
    return exception_triage_thread(exception, code, codeCnt, thread);
}
```

第一行获取当前线程，这是因为第二行调用 `` exception_triage_thread `` 把异常投递到异常端口时需要用到current thread，thread、task的异常端口数组都需要通过 thread 获取到：

```
thread->exc_actions;
task = thread->task;
task->exc_actions;

host_priv = host_priv_self();
host_priv->exc_actions;
```

而thread、task的异常端口默认是NULL，host的异常端口是第一个用户态进程 launchd(PID 1)初始化的时候就设置好的了，而且内核初始化成功后所有的用户态进程都是launchd 的子进程，子进程通过父进程fork继承了父进程的异常端口，因此所有的用户态进程出现异常时，异常都能在host层得到统一处理。

launchd 进程是如何设置host的异常端口的？接受到异常消息如何处理？

内核初始化的过程中，第一个用户态进程launchd 是在`` bsdinit_task() ``函数里启动的，在启动launchd 进程前通过调用`` host_set_exception_ports() `` 函数，把所有的Mach 异常消息都定向到端口`` ux_exception_port ``，这个端口由一个内核线程持有，这个内核线程里执行的`` ux_handle() ``函数，这个函数里会在一个死循环里调用`` mach_msg_receive() ``来接受`` ux_exception_port ``端口上的消息，而且`` mach_msg_receive() `` 会阻塞线程。

`` ux_handle() ``函数里接受到Mach消息后，会调用`` mach_exc_server()``，而`` mach_exc_server `` 会调用下面的handlers ，具体调用哪个由参数 `` exception_behavior_t behavior ``决定，该参数是设置异常端口时调用`` host_set_exception_ports() ``传入的

```
catch_mach_exception_raise() 对应 EXCEPTION_DEFAULT  1  ，表示 xx
catch_mach_exception_raise_state() 对应 define EXCEPTION_STATE  2 ，表示
catch_mach_exception_raise_state_identity() 对应 define EXCEPTION_STATE_IDENTITY  3，表示
```

`` catch_mach_exception_raise() ``这些handle 会调用 `` ux_exception()``将Mach异常转换成Unix信号，比如 `` EXC_BAD_ACCESS ``将会转换成 ``SIGSEGV`` 或 ``SIGBUS`` ，如代码所示

```
static
void ux_exception(
        int         exception,
        mach_exception_code_t   code,
        mach_exception_subcode_t subcode,
        int         *ux_signal,
        mach_exception_code_t   *ux_code)
{
    switch(exception) {

    case EXC_BAD_ACCESS:
        if (code == KERN_INVALID_ADDRESS)
            *ux_signal = SIGSEGV;
        else
            *ux_signal = SIGBUS;
        break;
    ....
    }
    ....
}

```
在 `` catch_mach_exception_raise()`` 里拿到Mach异常对应的Unix信号后会再调用 `` threadsignal() ``投递Unix信号，在`` threadsignal`` 的实现里通过几层函数调用，最后会调用到`` act_set_astbsd()`` ，在该函数里设置了AST（异步软件中断）信号

```
void
act_set_astbsd(
    thread_t    thread)
{
    act_set_ast( thread, AST_BSD );
}
```

AST 是人工引发的非硬件触发的陷阱，AST 是内核操作的关键部分，而且是调度事件的底层机制，也是BSD信号（Unix信号）投递的实现基础。当系统从一个陷阱返回时（`` return_from_trap``），系统不会立即返回用户态，而是要检查线程的ast字段以判断是否存在AST 需要处理。如代码所示，此时AST的标志位是 `` AST_BSD``，此标志位对应的handler 是`` bsd_ast() `` 函数。这时如果在`` exception_triage() ``下了断点，断点将会被断住，此时可以通过lldb在终端输出 函数调用栈、进程PID、线程状态

```
(lldb) bt
* thread #1, stop reason = breakpoint 1.17
  * frame #0: 0xffffff800fe75fc9 kernel.development`proc_prepareexit [inlined] exception_triage(exception=10, code=0x000000000b100001, codeCnt=2) at exception.c:504 [opt]
    frame #1: 0xffffff800fe75fbc kernel.development`proc_prepareexit [inlined] task_exception_notify(exception=10, exccode=185597953, excsubcode=4369) at exception.c:547 [opt]
    frame #2: 0xffffff800fe75f96 kernel.development`proc_prepareexit(p=0xffffff8018d90b60, rv=<unavailable>, perf_notify=1) at kern_exit.c:889 [opt]
    frame #3: 0xffffff800fe75d86 kernel.development`exit_with_reason(p=0xffffff8018d90b60, rv=11, retval=<unavailable>, thread_can_terminate=1, perf_notify=1, jetsam_flags=<unavailable>, exit_reason=<unavailable>) at kern_exit.c:830 [opt]
    frame #4: 0xffffff800fe90675 kernel.development`postsig_locked(signum=11) at kern_sig.c:3140 [opt]
    frame #5: 0xffffff800fe90b07 kernel.development`bsd_ast(thread=<unavailable>) at kern_sig.c:3420 [opt]
    frame #6: 0xffffff800f973e44 kernel.development`ast_taken_user at ast.c:207 [opt]
    frame #7: 0xffffff800f9211bc kernel.development`return_from_trap + 172
(lldb) e struct proc *$proc_1 = (struct proc *)thread->task->bsd_info
(lldb) po $proc_1->p_pid
478
(lldb) po thread->state
4
```

可以看到`` bsd_ast() `` 将会调用`` postsig_locked() ``数，从`` /bsd/kern/kern_sig.c `` `` postsig_locked() ``的实现可知，如果当前进程没有设置 sigaction 捕获Unix信号的话，默认处理是调用 `` exit_with_reason() ``，`` exit_with_reason() ``间接调用`` task_exception_notify() ``，`` task_exception_notify() ``的作用是通知launchd 去启动ReportCrash 生成CrashLog，通知的方式也是通过Mach消息传递机制，所以断点会在`` exception_triage() ``断住。

launchd 在初始化的过程中设置了异常端口，并且将 MachExceptionHandler 设置为`` /System/Library/CoreServices/ReportCrash （iOS中的路径）``，ReportCrash将会生成Crash Log。前面说了 ``exception_triage ``调用 `` exception_triage_thread() ``投递异常，而`` exception_triage_thread() ``函数里执行异常投递的函数是`` exception_deliver() ``，查看上面log中的``frame #0 ``可以看到函数入参`` exception=10 （EXC_CRASH）``，这是断点第二次在这断住，第一次断住是CPU异常转成Mach异常的时候，当时的`` exception=1 (EXC_BAD_ACCESS) ``，`` exception_deliver() ``函数将会利用入参 exception 从异常数组中取出具体的异常端口，所以第一次投递异常(CPU异常转Mach异常)和第二次投递异常给ReportCrash不会冲突。

此时再断点放掉，在 `` exception_triage_thread() ``处将会再出现一次断点

```
(lldb) bt
* thread #1, stop reason = breakpoint 2.1
  * frame #0: 0xffffff800f97ef47 kernel.development`exception_triage_thread(exception=13, code=0xffffff806fce3e40, codeCnt=2, thread=0xffffff801cded250) at exception.c:445 [opt]
    frame #1: 0xffffff800f9acffe kernel.development`task_deliver_crash_notification(task=0xffffff801d9af000, thread=0xffffff801cded250, etype=<unavailable>, subcode=<unavailable>) at task.c:1798 [opt]
    frame #2: 0xffffff800f9b6537 kernel.development`thread_terminate_self at thread.c:594 [opt]
    frame #3: 0xffffff800f9bab30 kernel.development`thread_apc_ast(thread=0xffffff801cded250) at thread_act.c:934 [opt]
    frame #4: 0xffffff800f973e6b kernel.development`ast_taken_user at ast.c:220 [opt]
    frame #5: 0xffffff800f9211bc kernel.development`return_from_trap + 172
```

可以从函数调用栈看出，这也是设置AST 导致的，此时的`` exception=13（EXC_CORPSE_NOTIFY）``，表示进程状态是僵尸状态，也就相当于死了。

通过打断点可以看出一个用户态应用程序非法访问内存导致的CPU异常，将会依次用到 `` EXC_BAD_ACCESS、EXC_CRASH、EXC_CORPSE_NOTIFY ``这三个Mach异常类型。

##### 小结

(todo 换图片)
CPU异常 -> Mach异常 -> BSD层的Unix信号 -> 用户态App Handler / 系统生成Crash Log 的流程可以简单粗略地画一个图

![s](http://m.qpic.cn/psb?/V10JaO4w40EHz4/1OfXzk0RWT*5eExl0A5ilUXtBXPMYkyQBWwJwE5akK4!/b/dC8BAAAAAAAA&bo=VwY4BAAAAAADB08!&rf=viewer_4)

#### 异常收集

虽然iOS \ macOS 都提供了 ReportCrash用来收集Crash 信息，Debug模式下也提供了 lldb 的debugserver 捕获程序异常，但App 发版上架后出现Crash 不方便开发者收集，比如在iOS上需要用户允许与开发者共享分析数据，开发者才可以从 iTunes Connect 查看到Crash 上报信息，不然则要拿到发生Crash的设备才能查看到Crash信息。

为了方便快速定位、解决Crash，可以借鉴 ReportCrash 或 debugserver 捕获异常的思路来做一个三方的Crash 收集的框架，收集思路主要有三种：

- 捕获 Mach 异常
- 捕获 Unix 信号
- NSSetUncaughtExceptionHandler

##### 捕获Mach 异常

Mach 虽然非常底层，但也提供了API给用户态应用程序使用，捕获Mach异常可以使用以下几个API
- 调用 mach_port_allocate 创建异常处理端口
- 调用 mach_port_insert_right 获取端口的权限
- 调用 xxx_set_exception_ports 设置异常端口
- 调用 mach_msg 等待异常端口上的消息

这里有两个需要注意的点：



##### 捕获Unix 信号

##### NSSetUncaughtExceptionHandler

##### 埋点


https://xiaozhuanlan.com/topic/6280793154
https://nianxi.net/ios/ios-crash-reporter.html
https://www.jianshu.com/p/80268ee99ddf
https://www.jianshu.com/p/34b98060965b

#### 堆栈恢复 Crash Log 格式