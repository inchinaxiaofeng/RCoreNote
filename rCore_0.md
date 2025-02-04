# 第零章

## 什么是操作系统

系统软件包括:

* 操作系统内核
* 驱动程序
* 工具软件
* 用户界面
* 软件库

操作系统的定义:

> 操作系统是一种系统软件，主要功能是向下管理CPU、内存和各种外设等硬件资源，并形成软件执行环境来向上管理和服务应用软件。

操作系统的组成部分:

* 操作系统内核
* 系统工具和软件库
* 用户接口

对于通用应用程序,我们需要关注以下问题:

* 一个运行的程序如何能输出字符信息？如何能获得输入字符信息？
* 一个运行的程序可以要求更多（或更少）的内存空间吗？
* 一个运行的程序如何持久地存储用户数据？
* 一个运行的程序如何与连接到计算机的设备通信并通过它们与物理世界通信？
* 多个运行的程序如何同步互斥地对共享资源进行访问？
* 一个运行的程序可以创建另一个程序的实例吗？需要等待另外一个程序执行完成吗？一个运行的程序能暂停或恢复另一个正在运行的程序吗？

## 操作系统抽象

操作系统具有四个抽象：

* 执行环境
* 进程
* 地址空间
* 文件

### 执行环境

定义:负责给在其上执行的软件提供相应的功能和资源,并可以在计算机系统中形成多层次的执行环境.

我们可以给应用程序的执行环境一个基本的定义：执行环境是应用程序正确运行所需的服务与管理环境，用来完成应用程序在运行时的数据与资源管理、应用程序的生存期等方面的处理，它定义了应用程序有权访问的其他数据或资源，并决定了应用程序的行为限制范围。

程序的控制流（Flow of Control or Control Flow）是指以一个程序的指令、语句或基本块为单位的执行序列。

* 普通控制流（CCF）
  平滑的序列
* 异常控制流(ECF)
  发出系统调用的请求、外设中断、CPU异常等情况，进入不同的执行环境，发生了**执行环境的切换**

控制流上下文的定义：
> 能确保下一时刻能继续*正确* 执行控制流指令的物理资源内容称为控制流的上下文（Context），也可称为控制刘所在执行环境的状态。

异常控制流：
>
> * 中断
>   * 由外部设备引起的外部IO事件
>   * 产生中断后，操作系统需要进行中断处理来响应中断请求，打断先前应用程序的上下文
> * 异常
>   * 在处理器执行指令期间检测到不正常的或非法的内部事件。
>   * 产生异常后，操作系统需要进行异常处理，打断先前应用程序的上下文
> * 陷入
>   * 是程序在执行过程中由于要通过系统调用请求操作系统服务而有意引发的事件
>   * 产生陷入后，操作系统需要执行系统调用服务来响应系统调用请求，打断先前

### 进程

一个正在运行中的程序。在一段时间内，往往有多个程序同时或交替在操作系统上运行，程序不能独占整个操作系统。由操作系统执行环境来管理程序执行过程中的**进程上下文**

这里的进程上下文是指程序在运行中的各种物理/虚拟资源（寄存器、可访问的内存区域、打开的文件、信号等）的内容，特别是与程序执行相关的具体内容：内存中的代码和数据，栈、堆、当前执行的指令位置（程序计数器的内容）、当前执行时刻的各个通用寄存器中的值等。进程上下文如下图所示：

***更好的定义：***
> 一个进程是一个具有一定独立功能的程序在一个数据集合上的一次动态执行过程。操作系统中的进程管理需要采用某种调度策略将处理器资源分配给程序并在适当的时候回收，并且要尽可能充分利用处理器的硬件资源。

### 地址空间

是对物理内存的虚拟化和抽象，也称虚存（Virtual Memory）。操作系统通过CPU/SOC中的MMU（内存管理单元）硬件支持，而给应用程序和用户提供的**虚拟存储空间**，他具有这个特征：
>
> * 大的
>   可能超过计算机中的物理内存容量
> * 连续的
>   连续的地址空间编址
> * 私有的
>   其他应用程序无法破坏

为了达到以上的效果，OS需要将内存与外存结合起来管理，为用户提供一个容量比内存大的多的虚拟存储器。

### 文件

这个概念是用于描述持久存储对象的，并能进一步拓展到外设。
操作系统需要用文件来屏蔽不同物理介质的差异。外设因其具有基本读写操作的特性，也通过文件进行统一管理

## 操作系统的特征

操作系统具有五个方面的特征：

* 虚拟化
* 并发性
* 异步性
* 共享性
* 持久性

### 虚拟性

#### 内存虚拟化

操作系统建立了一个*地址固定，空间巨大*的虚拟内存给应用程序来用，即**内存虚拟化**。内存虚拟化的核心问题：如何将虚拟地址与物理地址对应。

**内存虚拟化**也是一种**空间虚拟化**，进一步细分为**内存地址虚拟化**和**内存大小虚拟化**
对于**内存地址虚拟化**，cc和ld处理的方法就是将起始地址固定，产生包含虚拟地址的代码。当程序需要运行时，操作系统会建立虚拟地址与物理地址映射关系。
对于**内存大小虚拟化**，则是通过策略的将内存部分放到硬盘中存储以解决。

#### CPU虚拟化

**CPU虚拟化也是一种时间虚拟化**

### 并发性

应用程序分时执行，并由操作系统完成应用在运行时的切换

### 异步性

异步是由于操作系统的调度和中断，会不时打断当前运行中的程序，使得整个过程走走停停。即，完成时间是不可预测的。

### 共享性

多个应用并发时，宏观上体现出它们可以同时访问一个资源，即资源可以被共享。

### 持久性

操作系统提供的文件系统可以保存到持久性介质中。
