---
title: Android开机过程
date: 2016-12-01 15:52:22
categories: Android
tags: Android
---

# Android开机过程

## 基于Linux的pc启动过程
我们都知道，所有的程序软件包括操作系统都是运行在内存中的，然而我们的操作系统一般是存放在硬盘上的，当我们按下开机键的时候，此时内存中什么程序也没有，因此需要借助某种方式，将操作系统加载到内存中，而完成这项任务的就是BIOS：Basic Input/Output System（基本输入输出系统）。

<!--more-->


BIOS是我们电脑启动时加载的第一个程序，这个程序不是由Java语言编写也不是由C语言编写，一般是汇编程序。BIOS程序固化在主板上的一块芯片上，是连接计算机硬件与操作系统的桥梁。在BIOS的引导下，陆续将操作系统的其他程序载入内存，最后完成计算机的启动。

## Bootloader
Android系统虽然也是基于linux系统的，但是由于Android属于嵌入式设备，并没有像pc那样的BIOS程序。取而代之的是Bootloader——系统启动加载器。它类似于BIOS，在系统加载前，用以初始化硬件设备，建立内存空间的映像图，为最终调用系统内核准备好环境。

## 系统分区
在Android里没有硬盘，而是ROM，它类似于硬盘存放操作系统，用户程序等。ROM跟硬盘一样也会划分为不同的区域，用于放置不同的程序，在Android中主要划分为一下几个分区：
![](http://o8p68x17d.bkt.clouddn.com/android-partitions.png)

- /boot：存放引导程序，包括内核和内存操作程序

- /system：相当于电脑c盘，存放Android系统及系统应用

- /recovery：恢复分区，可以进入该分区进行系统恢复

- /data：用户数据区，包含了用户的数据：联系人、短信、设置、用户安装的程序

- /cache：安卓系统缓存区，保存系统最常访问的数据和应用程序

- /misc：包含一些杂项内容，如系统设置和系统功能启用禁用设置

- /sdcard：用户自己的存储区，可以存放照片，音乐，视频等文件

## Android 系统启动过程

Android 系统是基于 Linux 系统开发而成，在其中针对移动设备的特性进行了相应的调整，所以 Android 系统大体上可以分为 Linux 内核部分和 应用系统部分。在 Android 系统启动的时候，也会先启动 Linux 内核，然后再启动应用系统。整个启动步骤分为 6 个部分：
![](http://en.miui.com/data/attachment/forum/201403/01/07115949n944zi559xt1zt.jpg.thumb.jpg)

### 1. 启动电源以及系统启动
当电源按下，引导芯片代码开始从预定义的地方（固化在ROM）开始执行。加载引导程序到RAM，然后执行。

### 2. Boot Loader
当设备通电时，首先执行BootLoader引导装载器，加载Linux内核到RAM中。

### 3. 内核模块
内核模块，负责了大部分的硬件、驱动和文件系统的初始化。当内核完成系统设置，它首先在系统文件中寻找”init”文件，然后启动root进程或者系统的第一个进程。

### 4. init进程
init是第一个进程，我们可以说它是root进程或者说所有进程的父进程。init进程有两个责任，一是挂载目录，比如/sys、/dev、/proc，二是运行init.rc脚本。  

Init 进程会启动 Runtime 运行时服务，所谓的 Runtime 服务就是将中间代码静态编译成本地代码。而 Android 所使用的 Java 动态执行，所依附的正式这个 Runtime 环境。

其后，Init 进程会启动一些本地守护进程，这些守护进程启动后，会初始化其相应的模块，在其中最特别的就是 Zygote。不过 Init 进程不会直接启动 Zygote 进程，而是使用 app_process 命令来通过 Android Runtime 来启动，Android Runtime 会启动第一个 Davlik 虚拟机，并调用 Zygote 的 main 方法。

在这个阶段你可以在设备的屏幕上看到“Android”logo了。

### 5. Zygote and Dalvik
对于每一个运行的程序，Android 都会赋予其单独的虚拟机，以支持其运行，但每一次新建虚拟机的开销都不小，那么如何来缩短这个时间了？为了克服这个问题，Android系统创造了”Zygote”。Zygote让Dalvik虚拟机共享代码、低内存占用以及最小的启动时间成为可能。我们其余的应用进程都是通过 fork 这个Zygote进程来实现的，他们共享相同的内存区域，这样能减少不少的内存占用开销和应用启动时间。

为了加速 App 的启动，Zygote 进程会预先加载 App 在运行时所需的资源和 Class 文件到系统 RAM 中。Zygote 会监听其 Socket (/dev/socket/zygote) 来判断是否需要启动 App。每当监听到需要创建 App 的请求时，就 fork 一个进程即可。这样的好处在于最初始的 Zygote 进程，保有所有的系统 Class 和 App 启动可能需要的资源，这样一来，就不需要启动一个 App 时，动态去加载相应的资源。

为何在 Android 中 fork Zygote 进程如此高效了？Linux Kernel 采用了 Copy-On-Write 的技术，Copy-On-Write 的意思是只有在写的时候才单独复制一份，而读的时候不进行操作。 换而言之 fork zygote 实际上并未实际复制什么东西，只有在发生写操作时，才单独复制一份。而另一方面，class 和 Resource 资源文件并不需要重新写，这些文件在绝大多数时候都是只读的。最后，实际的效果就是尽管运行着多个 APP，但实际只有一份 class 和 resource 文件在 Zygote 进程中。

### 6. System Server 和 系统服务
随后Zygote进程会fork出System Server进程，System Server进程负责启动和管理整个framework，包括Activity Manager，PowerManager，Battery Service,Window Manager等服务。

这其中也包括我们熟知的 Activity Manager，ActivityManager会启动首个Android程序即Launcher，启动完成后，会发送一个 Intent.CATEGORY_HOME 的 intent，从而我们就能看到我们熟知的桌面。

另外System Server会创建一个socket客户端，监听启动应用的请求。应用的启动流程看[Android进程启动流程]()。

上面主要涉及到了system_server进程和Zygote进程，概括下两者的特点：

- system_server进程：是用于管理整个Java framework层，包含ActivityManager，PowerManager等各种系统服务;
- Zygote进程：是Android系统的首个Java进程，Zygote是所有Java进程的父进程，包括 system_server进程以及所有的App进程都是Zygote的子进程，注意这里说的是子进程，而非子线程。



## 流程图
![](http://o8p68x17d.bkt.clouddn.com/android-boot.png)

再来一张：  
![](https://user-gold-cdn.xitu.io/2016/11/29/dd459c34fe4e4c60edfae01f621802e9.jpg)

## 总结
- BootLoder引导,然后加载Linux内核.
- 0号进程init启动.加载init.rc配置文件,配置文件有个命令启动了zygote进程
- zygote开始fork出SystemServer进程
SystemServer加载各种JNI库,然后init1,init2方法,init2方法中开启了新线程ServerThread.
- 在SystemServer中会创建一个socket客户端，后续AMS（ActivityManagerService）会通过此客户端和zygote通信
- ServerThread的run方法中开启了AMS,还孵化新进程ServiceManager,加载注册了一溜的服务,最后一句话进入loop 死循环
- run方法的SystemReady调用resumeTopActivityLocked打开锁屏界面


## 参考
[详解 Android 是如何启动的](http://www.woaitqs.cc/android/2016/06/15/how-android-launch-itself.html)  
[Android从开机到打开第一个应用发生了什么？](https://segmentfault.com/a/1190000004676352)  
[Android内核解读-Android系统的开机启动过程](http://blog.csdn.net/singwhatiwanna/article/details/19302593)