# 应用程序与基本执行环境

## 应用程序执行环境与平台支持

### 平台与目标三元组

CC在编译、链接得到可执行文件时，需要知道，程序在哪个**目标平台**上运行，**目标三元组（Target Triplet）**描述了目标平台的：
>
> 1. CPU指令集
> 2. 操作系统类型
> 3. 标准运行时库

```bash
$ rustc --verbose --version
rustc 1.80.0-nightly (c987ad527 2024-05-01)
binary: rustc
commit-hash: c987ad527540e8f1565f57c31204bde33f63df76
commit-date: 2024-05-01
host: x86_64-unknown-linux-gnu
release: 1.80.0-nightly
LLVM version: 18.1.4
```

> host：**目标平台**是`x86_64-unknown-linux-gnu`，CPU架构是x86_64，CPU厂商是Unknow，操作系统是Linux，运行时库是gnu libc
> 我们的目标是将程序移植到RISCV目标平台：`riscv64gc-unknown-none-elf`
>
> * 架构是riscv64gc，厂商是unknown
> * 操作系统是none
> * elf表示没有标准的运行时库。没有任何系统调用的封装支持，但是可以生成ELF格式的执行程序。
> 不选择有linux-gnu支持的`riscv64gc-unknown-linux-gnu` 是因为我们的目标是开发操作系统，而不是在linux系统上运行的应用程序

### 修改目标平台

```bash
cargo run --target riscv64gc-unknown-none-elf
 Compiling os v0.1.0 (/home/marinatoo/tmp/os)
error[E0463]: can't find crate for `std`
  |
  = note: the `riscv64gc-unknown-none-elf` target may not support the standard library
  = note: `std` is required by `os` because it does not declare `#![no_std]`
  = help: consider building the standard library from source with `cargo build -Zbuild-std
```

目标平台上还没有实现Rust标准库std，也不存在任何接受OS支持的系统调用，即**裸机平台**。
Rust中存在一个不需要任何操作系统支持的核心库`core`，包含了rust语言大部分核心机制，很多的第三方库也不依赖标准库std,而仅仅包含核心库core。
所以，我们需要**对标准库std的引用换成核心库core**

## 移除标准库依赖

在`os`目录下新建.cargo目录，并在这个目录下创建config文件，输入

```Rust
# os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"
```

这种编译器运行平台与可执行文件运行的目标平台不一致的情况，叫**交叉编译（CrossCompile）**

### 移除println!宏

`main.rs`的开头加上`#![no_std]`以使用核心库core。得到报错：

```bash
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
error: cannot find macro `println` in this scope
--> src/main.rs:4:5
  |
4 |     println!("Hello, world!");
  |     ^^^^^^^
```

> println!宏是由std提供的，自然要移除。

### 提供语义项

```bash
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
error: `#[panic_handler]` function required, but not found
```

std提供了错误处理函数`#[panic_handler]`，我们需要自己实现

新建子模块`lang_items.rs`，在里面写panic处理函数，通过标记`#[panic_handler]`来告诉编译器我们的实现

```Rust
// os/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

### 移除main函数

```bash
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
error: requires `start` lang_item
```

start语义项代表标准库std在执行应用程序之前所需的一些初始化工作。禁用标准库后，编译器也就找不到这项功能的实现了。
在`main.rs`开头加入`#![no_main]`来告诉编译器我们没有一般意义上的main函数，并将原来的main函数删除，这样编译器就不考虑初始化的事情了。

```bash
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s
```

目前，我们移除了所有标准库依赖，目前代码如下：

```Rust
// os/src/main.rs
#![no_std]
#![no_main]

mod lang_items;

// os/src/lang_items.rs
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}
```

### 分析被移除标准库的程序

我们可以通过一些工具来分析目前的程序：

```bash
[文件格式]
$ file target/riscv64gc-unknown-none-elf/debug/os
target/riscv64gc-unknown-none-elf/debug/os: ELF 64-bit LSB executable, UCB RISC-V, ......

[文件头信息]
$ rust-readobj -h target/riscv64gc-unknown-none-elf/debug/os
   File: target/riscv64gc-unknown-none-elf/debug/os
   Format: elf64-littleriscv
   Arch: riscv64
   AddressSize: 64bit
   ......
   Type: Executable (0x2)
   Machine: EM_RISCV (0xF3)
   Version: 1
   Entry: 0x0
   ......
   }

[反汇编导出汇编程序]
$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
   target/riscv64gc-unknown-none-elf/debug/os:       file format elf64-littleriscv
```

file进行二进制分析可以得知，其是为一个合法的RV64执行程序，但是rust-readobj指出入口地址为0。通过rust-objdump工作进行反汇编，没有生成任何汇编代码，可见，二进制程序虽然合法，但是它是一个空程序，原因是缺少了编译器规定的入口函数`_start`。

## 构建用户态执行环境

### 用户态最小化执行环境

#### 执行环境初始化

给Rust编译器提供入口函数`_start()`

```rust
// os/src/main.rs
#[no_mangle]
extern "C" fn _start {
  loop{};
}
```

重编译后分析：

```bash
$ cargo build
   Compiling os v0.1.0 (/home/shinbokuow/workspace/v3/rCore-Tutorial-v3/os)
    Finished dev [unoptimized + debuginfo] target(s) in 0.06s

[反汇编导出汇编程序]
$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os
   target/riscv64gc-unknown-none-elf/debug/os:       file format elf64-littleriscv

   Disassembly of section .text:

   0000000000011120 <_start>:
   ;     loop {}
     11120: 09 a0            j       2 <_start+0x2>
     11122: 01 a0            j       0 <_start+0x2>
```

反汇编出的两条指令就是一个死循环， 这说明编译器生成的已经是一个合理的程序了。 用 qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os 命令可以执行这个程序。

#### 程序正常退出

我们把 _start() 函数中的循环语句注释掉，重新编译并分析，看到其汇编代码是：

```bash

$ rust-objdump -S target/riscv64gc-unknown-none-elf/debug/os

 target/riscv64gc-unknown-none-elf/debug/os: file format elf64-littleriscv


 Disassembly of section .text:

 0000000000011120 <_start>:
 ; }
   11120: 82 80              ret

```

执行后出现报错：

```bash
$ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os
  段错误 (核心已转储)
```

Why？
> QEMU有两种运行模式:
>
> * `User Mode`模式，即用户态模拟。`qemu-riscv64` 能够模拟不同处理器用户态指令的执行，并可以直接解析ELF可执行文件，加载运行那些为不同处理器编译的用户级Linux应用程序
> * `System Mode`模式，即系统态模式。`qemu-system-riscv64` 能模拟一个完整的基于不同CPU的硬件系统，包括处理器、内存及其他外部设备，支持运行完整的操作系统。

代码还缺少退出机制

```rust
// os/src/main.rs

const SYSCALL_EXIT: usize = 93;

fn syscall(id: usize, args: [usize; 3]) -> isize {
    let mut ret;
    unsafe {
        core::arch::asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id,
        );
    }
    ret
}

pub fn sys_exit(xstate: i32) -> isize {
    syscall(SYSCALL_EXIT, [xstate as usize, 0, 0])
}

#[no_mangle]
extern "C" fn _start() {
    sys_exit(9);
}
```

### 有显示支持的用户态执行环境

#### 实现输出字符串的相关函数

对`SYSCALL_WRITE`系统调用进行封装

```Rust
const SYSCALL_WRITE: usize = 64;

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize {
  syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}
```

实现基于`Write` Trait的数据结构，并完成`Write` Trait所需要的`write_str`函数，并用`print`函数进行包装。

```Rust
struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        sys_write(1, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}
```

基于print函数，实现**格式化宏**

```Rust
#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

调整应用程序，让它发出显示字符串和退出的请求

```Rust
#[no_mangle]
extern "C" fn _start() {
    println!("Hello, world!");
    sys_exit(9);
}
```

编译，执行，可以看到输出

```bash
$ cargo build --target riscv64gc-unknown-none-elf
   Compiling os v0.1.0 (/media/chyyuu/ca8c7ba6-51b7-41fc-8430-e29e31e5328f/thecode/rust/os_kernel_lab/os)
  Finished dev [unoptimized + debuginfo] target(s) in 0.61s

$ qemu-riscv64 target/riscv64gc-unknown-none-elf/debug/os; echo $?
  Hello, world!
  9
```

## 构建裸机执行环境

本次我们尝试将Hello World从用户态迁移到内核态

### 裸机启动过程

用`qemu-system-riscv64`来模拟RISC-V 64计算机，命令如下：

```bash
qemu-system-riscv64 \
            -machine virt \
            -nographic \
            -bios $(BOOTLOADER) \
            -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
```

* `-bios $(BOOTLOADER)` 硬件加载了一个BootLoader程序，RustSBI
* `-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)`表示内存中特定的位置
  * `$(KERNEL_ENTRY_PA)`放置了操作系统的二进制代码`$(KERNEL_BIN)`。`$(KERNEL_ENTRY_PA)`的值是`0x80200000`

行为：

1. 上电后，CPU的其他GPR清0，PC指向0x1000的位置，这里有固化在硬件中的引导代码。
2. 很快跳转到`0x80000000`的RustSBI中。
3. `RustSBI`完成硬件初始化后， 会跳转到`$(KERNEL_BIN)`所在的内存位置`0x80200000`，执行操作系统的第一条指令

* * * RustSBI：SBI是RISC-V的一种底层规范，RustSBI是一种实现，操作系统内核与RustSBI的关系类似于应用与操作系统内核的关系，后者向前者提供一定服务，只是SBI提供的服务很少，比如关机、显示字符等。

### 实现关机公能

使用`ecall`调用RustSBI实现关机

```Rust
// bootloader/rustsbi-qemu.bin 直接添加的SBI规范实现的二进制代码，给操作系统提供基本支持服务

// os/src/sbi.rs
fn sbi_call(which: usize, arg0: usize, arg1: usize, arg2: usize) -> usize {
 let mut ret;
  unsafe {
      core::arch::asm!(
          "ecall",
...

const SBI_SHUTDOWN: usize = 8;

pub fn shutdown() -> ! {
    sbi_call(SBI_SHUTDOWN, 0, 0, 0);
    panic!("It should shutdown!");
}

// os/src/main.rs
#[no_mangle]
extern "C" fn _start() {
    shutdown();
}
```

应用程序访问操作系统的系统调用：`ecall`，由`User Mode`（程序）转向`Supervisor Mode`（操作系统）
操作系统访问RustSBI：`ecall`，由`Supervisor Mode`转向`MachineMode`

结果：

```bash
# 编译生成ELF格式的执行文件
$ cargo build --release
 Compiling os v0.1.0 (/media/chyyuu/ca8c7ba6-51b7-41fc-8430-e29e31e5328f/thecode/rust/os_kernel_lab/os)
  Finished release [optimized] target(s) in 0.15s
# 把ELF执行文件转成bianary文件
$ rust-objcopy --binary-architecture=riscv64 target/riscv64gc-unknown-none-elf/release/os --strip-all -O binary target/riscv64gc-unknown-none-elf/release/os.bin

# 加载运行
$ qemu-system-riscv64 -machine virt -nographic -bios ../bootloader/rustsbi-qemu.bin -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
# 无法退出，风扇狂转，感觉碰到死循环
```

问题出现在：入口地址不是RustSBI约定的`0x80200000`。

### 设置正确的程序内存布局

通过**链接脚本（Linker script）**调整链接器行为

``` linker.ld
// os/.cargo/config
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]
```

``` link
OUTPUT_ARCH(riscv)
ENTRY(_start)
BASE_ADDRESS = 0x80200000;

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }

    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

第一行设置了目标平台，第二行设置了整个程序的入口为`_start`，第三行定义了常量`BASE_ADDRESS`为`0x80200000`

### 正确配置栈空间布局

```asm
# os/src/entry.asm
    .section .text.entry
    .globl _start
_start:
    la sp, boot_stack_top
    call rust_main

    .section .bss.stack
    .globl boot_stack
boot_stack:
    .space 4096 * 16
    .globl boot_stack_top
boot_stack_top:
```

8行，有64KiB的空间用于作为操作系统的栈空间。栈顶地址为`boot_stack_top`，栈底为`boot_stack`。这块空间为`.bss.stack`。

`_start`作为OS的入口地址，将依据链接脚本被放到`BASE_ADDRESS`处。
`la sp, boot_stack_top`作为OS的第一条指令，将sp设置为栈空间的栈顶。目前不考虑栈溢出。
第二条指令调用函数`rust_main`。

在main.rs中嵌入这些汇编代码，并声明应用入口`rust_main`:

```Rust
// os/src/main.rs
#![no_std]
#![no_main]

mod lang_items;

core::arch::global_asm!(include_str!("entry.asm"));

#[no_mangle]
pub fn rust_main() -> ! {
    shutdown();
}
```

第七行，使用`global_asm`宏，将同目录下的汇编文件`entry.asm`嵌入到代码中。
9行开始，声明了应用入口rust_main，标记为`#[no_mangle]`以避免编译器对它名字进行混淆，否则链接时，`entry.asm`将找不到`main.rs`提供的外部符号`rust_main`。

生成与运行：

```bash
# 教程使用的 RustSBI 版本比代码框架稍旧，输出有所不同
$ qemu-system-riscv64 \
> -machine virt \
> -nographic \
> -bios ../bootloader/rustsbi-qemu.bin \
> -device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000
[rustsbi] Version 0.1.0
.______       __    __      _______.___________.  _______..______   __
|   _  \     |  |  |  |    /       |           | /       ||   _  \ |  |
|  |_)  |    |  |  |  |   |   (----`---|  |----`|   (----`|  |_)  ||  |
|      /     |  |  |  |    \   \       |  |      \   \    |   _  < |  |
|  |\  \----.|  `--'  |.----)   |      |  |  .----)   |   |  |_)  ||  |
| _| `._____| \______/ |_______/       |__|  |_______/    |______/ |__|

[rustsbi] Platform: QEMU
[rustsbi] misa: RV64ACDFIMSU
[rustsbi] mideleg: 0x222
[rustsbi] medeleg: 0xb1ab
[rustsbi] Kernel entry: 0x80200000
```

### 清空.bss段

**清零.bss段**

```rust
// os/src/main.rs
fn clear_bss() {
    extern "C" {
        fn sbss();
        fn ebss();
    }
    (sbss as usize..ebss as usize).for_each(|a| {
        unsafe { (a as *mut u8).write_volatile(0) }
    });
}

pub fn rust_main() -> ! {
    clear_bss();
    shutdown();
}
```

### 添加裸机打印相关函数

修改一下之前的那个为用户态实现的`println`，我们也可以用`println`来重写`panic`，使得能打印错误位置。
