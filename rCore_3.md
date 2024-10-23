# 多道程序与分时多任务

---

# 引言

通过以下行为，可以改善OS在性能上的表现

* 通过提前加载应用程序到内存，减少应用程序切换开销
* 通过协作机制，支持程序主动放弃处理器，提高系统执行效率
* 通过抢占机制支持程序被动放弃处理器，保证不同程序对处理器资源使用的公平性，也进一步提高了应用对IO事件的响应效率

目的：在内存中驻留多个应用

## 协作式操作系统

应用执行IO操作时，主动放弃处理器。*放弃处理器*的操作由用户通过系统调用完成。这样的OS就是支持**多道程序**或**协作式多任务**的协作式OS

## 抢占式OS

应用程序难以在合适的位置放置**放弃处理器的系统调用请求**，所以建立了一种方法，打断了应用程序的执行：
利用固定时长的外设中断（时钟中断）打断程序执行，每一个程序只能运行一个时间片（Time Slice）就会让出处理器，且操作系统可以在处理外设的IO响应后，让不同的应用程序分时占用处理器执行，以此评估程序的资源消耗。这种方式是**分时共享（Time Sharing）**或**抢占式多任务（Multitasking）**，合并到一起就是**分时多任务**。

多道程序和分时多任务系统的共同特点：

* 在内存中同一时间可以驻留多个应用
* 所有的应用都是在系统启动的时候分别加载到内存的不同区域中
* 同一时间最多只有一个应用在执行（即处于运行状态），剩下的应用处于就绪状态或等待状态，需要内核将处理器分配给它们才能开始执行。
* 一旦应用开始执行，它就处于运行状态

## 额外代码

与BatchOS不同的点与为了实现其所需要的努力：

* 多个应用同时放在内存中，所以他们的起始地址是不同的，且地址范围不能重叠
  * 对App的地址空间布局调整，每个应用的地址空间不同，且不能重叠
  * 通过`build.py`来针对每个应用程序修改链接脚本`linker.ld`中的`BASE_ADDRESS`，让编译器在编译不同应用的时候，用到的BASE_ADDRESS都不同，且有足够大的地址间隔。
* 应用在整个执行过程中会暂停或被抢占，即会有主动或被动的任务切换
  * 在Trap上下文切换的基础上，增添Task上下文的切换。
  * 数据结构：`TaskContext`数据结构
  * 具体上下文切换的汇编语言写的函数：`__switch`函数
  * 通过`TaskControlBlock`数据结构来表示应用执行上下文的动态执行过程和状态（运行态、就绪态）
  * `TaskManager`数据结构的全局变量实例`TASK_MANAGER`描述了App初始化所需要的数据，对于TASK_MANAGER的初始化赋值过程是实现这个准备的关键步骤

为了实现这几个差异，我们需要对以下部分修改：

* 应用程序执行过程的管理
* 支持应用程序暂停的系统调用
* 支持主动切换应用程序所需的, 时钟中断机制的管理

* App在Umode中主动暂停：
  * 新的系统调用`sys_yield`
* 抢占式切换
  * 对时钟中断的处理
* 中断后切换到另一个应用
  * `trap_handler`进行扩展
* `TaskManager`数据结构的成员函数`run_netx_task`来具体实现基于任务控制块的任务切换，调用`__switch`函数完成硬件相关部分的任务上下文切换

# 多道程序放置与加载

## 导读

出现这种OS的前提是，计算机中物理内存容量增加了，足以容纳多个应用程序的内容。通过具体实现，可以让多个应用程序一次性地加载到内存中，然后再切换到另一个应用程序会比较快。前一个BatchOS需要清空之前一个应用，然后加载当前应用。

## 多道程序放置

所有的Elf格式执行文件都经过`objcopy`工具丢掉所有ELF header和符号变为二进制img文件，随后以同样的格式通过在操作系统内核中嵌入`link_user.S`文件，在编译的时候直接吧应用链接到内核的数据段中。
不同的是，第二章中应用的加载和执行进度控制都交给`batch`子模块，第三章，将应用的加载这部分功能分离开来在`loader`子模块中，应用的执行和切换交给`task`子模块。

我们需要调整每个应用被构建时使用的链接脚本`linker.ld`中的起始地址`BASE_ADDRESS`，这个地址是应用被内核加载到内存中的起始地址。Which is：
App会知道自己会被加载到某个地址运行，内核也的确能将App加载到那个地址。（目前OS Kernel的能力比较弱，对App的通用性的支持不够），目前App的编址方式是基于**绝对地址**的，并没有做到与位置无关，Kernel也没有提供相应的地址重定位机制。

使用Python脚本链接（因为`BASE_ADDRESS`都是不同的）

## 多道程序加载

应用的load方式不同:

* **第二章：让所有应用都共享一个固定的加载物理地址**
  * 所以内存中只能驻留一个应用，当它完成或出错推出的时候，有OS的Batch模块加载一个新的App替换它。
* **第三章：所有应用在内核初始化的时候，被一同加载入内存中**
  * 为了避免覆盖，需要加载到不同的物理地址
  * 通过调用`loader`子模块的`load_apps`函数实现的

```Rust
// os/src/loader.rs
pub fn load_apps() {
    extern "C" { fn_num_app(); }
    let num_app_ptr =_num_app as usize as *const usize;
    let num_app = get_num_app();
    let app_start = unsafe {
        core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1)
    };
    // load apps
    for i in 0..num_app {
        let base_i = get_base_i(i);
        // clear region
        (base_i..base_i + APP_SIZE_LIMIT).for_each(|addr| unsafe {
            (addr as*mut u8).write_volatile(0)
        });
        // load app from data section to memory
        let src = unsafe {
            core::slice::from_raw_parts(
                app_start[i] as *const u8,
                app_start[i + 1] - app_start[i]
            )
        };
        let dst = unsafe {
            core::slice::from_raw_parts_mut(base_i as*mut u8, src.len())
        };
        dst.copy_from_slice(src);
    }
    unsafe {
        asm!("fence.i");
    }
}
```

## 执行应用程序

run_next_app会让CPU从S到U,但是这里需要返回不同的Trap上下文：

* 跳转到应用程序i的入口点`entry_i`
* 将使用的栈切换到用户栈`stack_i`

# 任务切换

上一节的OS,一个程序独占CPU,直到出错或主动推出。操作系统还是以程序的一次执行过程（从开始到结束）作为处理器切换程序的时间段。我们需要引入新的概念：

* 任务
* 任务切换
  * 由`__switch`函数实现。
  * 一个应用运行中途就会主动或被动交出CPU使用权，暂停执行，直到内核再次给它分配处理器资源。
* 任务上下文

## 任务的概念形成

我们把应用程序的一次执行过程称作一个**任务**，把App执行过程中的一个时间片段上的执行片段或空闲片段称作**计算任务片**或**空闲任务片**。当应用所有的任务片都完成后，一次任务也就完成了。
一个任务切换到另一个任务成为**任务切换**。
操作系统需要支持让任务的执行“暂停”或“继续”。

**一条控制流需要“暂停-继续”，那么就需要提供一种控制流切换机制，且要保证程序的控制流被切换出去之前和切换回来之后，能继续正常执行：需要保证上下文不变或变化在预期范围内。这些上下文称为**任务上下文（Task Context）**。

## 不同类型的上下文与切换

任务的一次切换涉及到被换出和即将被换入的两条控制流，通常遵循一定的约定来完成任务。

* 第一章：函数调用和栈
  * 需要保存和恢复函数调用的上下文。
* 第二章：批处理系统
  * 涉及到Trap控制流，需要保存和恢复**系统调用（Trap）上下文**
  * 必须利用硬件提供的特权机制

Trap控制流的特点：

* Trap控制流是在Trap一瞬间生成的
* Trap对于应用是透明的
* Trap需要把Trap上下文保存在自己的内核栈上

## 任务切换的设计与实现

任务切换是另一种异常控制流：

* 不设计特权级切换
* 一部分由编译器帮忙完成的
* 对应用透明

任务切换来自两个App在Kernel中的Trap控制流的切换。一个App Trap到S Mode，这个Trap控制流调用`__switch`函数。
`__switch`表面上就是一个普通函数调用：在`__switch`返回之后，继续从函数调用位置向下执行。但是，调用这个函数时，原Trap控制流*A*会被暂停并切换出去，CPU转而运行*B*的Trap控制流。
`__switch`函数和一个普通函数之间的区别就是，它会**换栈**。

> [!NOTE]
> 情况说明：
> 每一个任务都会有一个自己的KernelStack，在Kernel Stack中存在Trap Context和Trap Handler Stack（正常函数调用逻辑）。即，每一个Task都有自己的Trap Context来存储。Trap控制流也可以以类似的方式切换。
> 存在**任务上下文（Task Context）**，保存一些内容（后面介绍）。Task Context存储在`TaskManager`中，其中有一个数组`tasks`，每一项都是一个**任务控制块（`TaskControlBlock`）**，负责保存一个任务的状态。`TaskContext`存储在`TaskControlBlock`中。
> Kernel运行的时候会初始化`TaskManager`的全局实例`TASK_MANAGER`，因此，所偶TaskContext实际都存储在`TASK_MANAGER`中，从内存布局来看，就是Kernel的全局数据`.data`。我们将TaskContext保存后，再次执行这个Task时，就会从同样位置恢复TaskContext

当前正在执行任务的Trap控制流我们用`current_task_cx_ptr`的变量，来存储当前上下文的地址。
用`next_task_cx_ptr`来存储下一个要执行任务的上下文地址，即：

``` c
TaskContext *current_task_cx_ptr = &tasks[current].task_cx;
TaskContext *next_task_cx_ptr    = &tasks[next].task_cx;
```

`__switch`四阶段：

1. 在Trap控制流A调用`__switch`之前，A的Kernel Stack只有Trap Context和Trap Handler Stack
2. A在A任务上下文空间中保存CPU当前寄存器快照
3. 读取`next_task_cx_ptr`指向的B Task Context，根据B Task Context来恢复`ra`寄存器、`s0-s11`寄存器以及`sp`寄存器。

* 只有通过这一步，`__switch`才能做到一个函数跨越两条控制流执行，也就是，**通过换栈实现了控制流的执行**

4. 上一部寄存器恢复后，通过恢复`sp`寄存器，切换到Task B的KernelStack了。

* 此后，当CPU执行`ret`指令完成`__switch`函数返回后，任务B就从调用`__switch`的位置继续执行。
* A在保存Task Context之后进入暂停状态，B则恢复了上下文并在CPU上运行。

Task Context结构：

```Rust
// os/src/task/context.rs
pub struct TaskContext {
    ra: usize,
    sp: usize,
    s: [usize; 12],
}
```

`ra`很重要，其关乎了`__switch`函数返回后应该返回那里执行，从而在完成任务切换后，`ret`到正确的位置。
> > [!NOTE]
> 对于一般的函数而言，Rust/C 编译器会在函数的起始位置自动生成代码来保存 s0~s11 这些被调用者保存的寄存器。
> `__switch`是汇编写成的，不会被Rust/C编译器处理，所以需要手动保存。
> 不用保存其他寄存器的原因：
> >其它寄存器中，属于调用者保存的寄存器 是由编译器在高级语言编写的调用函数中自动生成的代码来完成保存的
> > 还有一些寄存器属于临时寄存器，不需要保存和恢复。

我们会将这段汇编代码中的全局符号 __switch 解释为一个 Rust 函数：

``` Rust
// os/src/task/switch.rs

global_asm!(include_str!("switch.S"));

use super::TaskContext;

extern "C" {
    pub fn __switch(
        current_task_cx_ptr: *mut TaskContext,
        next_task_cx_ptr: *const TaskContext
    );
}
```

选择函数调用而不是直接跳转符号`__switch`所在的地址，是因为调用前后，Rust编译器会插入保存/恢复调用者保存寄存器的汇编代码。
