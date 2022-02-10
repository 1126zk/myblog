# 实现分时多任务系统与抢占式调用

## 第三章代码树

分时多任务的为例

```
./os/src
Rust        18 Files   511 Lines
Assembly     3 Files    82 Lines

├── bootloader
│   ├── rustsbi-k210.bin
│   └── rustsbi-qemu.bin
├── LICENSE
├── os
│   ├── build.rs
│   ├── Cargo.toml
│   ├── Makefile
│   └── src
│       ├── batch.rs(移除：功能分别拆分到 loader 和 task 两个子模块)
│       ├── config.rs(新增：保存内核的一些配置)
│       ├── console.rs
│       ├── entry.asm
│       ├── lang_items.rs
│       ├── link_app.S
│       ├── linker-k210.ld
│       ├── linker-qemu.ld
│       ├── loader.rs(新增：将应用加载到内存并进行管理)
│       ├── main.rs(修改：主函数进行了修改)
│       ├── sbi.rs(修改：引入新的 sbi call set_timer)
│       ├── sync
│       │   ├── mod.rs
│       │   └── up.rs
│       ├── syscall(修改：新增若干 syscall)
│       │   ├── fs.rs
│       │   ├── mod.rs
│       │   └── process.rs
│       ├── task(新增：task 子模块，主要负责任务管理)
│       │   ├── context.rs(引入 Task 上下文 TaskContext)
│       │   ├── mod.rs(全局任务管理器和提供给其他模块的接口)
│       │   ├── switch.rs(将任务切换的汇编代码解释为 Rust 接口 __switch)
│       │   ├── switch.S(任务切换的汇编代码)
│       │   └── task.rs(任务控制块 TaskControlBlock 和任务状态 TaskStatus 的定义)
│       ├── timer.rs(新增：计时器相关)
│       └── trap
│           ├── context.rs
│           ├── mod.rs(修改：时钟中断相应处理)
│           └── trap.S
├── README.md
├── rust-toolchain
├── tools
│   ├── kflash.py
│   ├── LICENSE
│   ├── package.json
│   ├── README.rst
│   └── setup.py
└── user
    ├── build.py(新增：使用 build.py 构建应用使得它们占用的物理地址区间不相交)
    ├── Cargo.toml
    ├── Makefile(修改：使用 build.py 构建应用)
    └── src
        ├── bin(修改：换成第三章测例)
        │   ├── 00power_3.rs
        │   ├── 01power_5.rs
        │   ├── 02power_7.rs
        │   └── 03sleep.rs
        ├── console.rs
        ├── lang_items.rs
        ├── lib.rs
        ├── linker.ld
        └── syscall.rs
```

## 本节的任务

本节实现一个支持 对中断的处理和对应用程序的抢占，设计实现更加公平和高效交互的抢占式操作系统

当外设想要触发中断的时候则输入一个高电平或正边沿，处理器会在每执行完一条指令之后检查一下这根线，看情况决定是继续执行接下来的指令还是进入中断处理流程

本章的操作系统支持把多个应用的代码和数据放置到内存中；并能够执行每个应用；在应用程序发出 `sys_yeild` 系统调用时，协作式地切换应用；并能通过时钟中断来实现抢占式调度并强行切换应用，从而提高了应用执行的灵活性、公平性和交互效率。

## 本节文件解析

### 在bootloader 文件夹下

### rustsbi-qemu.bin

### 在 os 文件夹下

#### os/.cargo/config

```
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]

```

#### os/build.rs

```rust
use std::io::{Result, Write};
use std::fs::{File, read_dir};

fn main() {
    println!("cargo:rerun-if-changed=../user/src/");
    println!("cargo:rerun-if-changed={}", TARGET_PATH);
    insert_app_data().unwrap();
}

static TARGET_PATH: &str = "../user/target/riscv64gc-unknown-none-elf/release/";

fn insert_app_data() -> Result<()> {
    let mut f = File::create("src/link_app.S").unwrap();
    let mut apps: Vec<_> = read_dir("../user/src/bin")
        .unwrap()
        .into_iter()
        .map(|dir_entry| {
            let mut name_with_ext = dir_entry.unwrap().file_name().into_string().unwrap();
            name_with_ext.drain(name_with_ext.find('.').unwrap()..name_with_ext.len());
            name_with_ext
        })
        .collect();
    apps.sort();

    writeln!(f, r#"
    .align 3
    .section .data
    .global _num_app
_num_app:
    .quad {}"#, apps.len())?;

    for i in 0..apps.len() {
        writeln!(f, r#"    .quad app_{}_start"#, i)?;
    }
    writeln!(f, r#"    .quad app_{}_end"#, apps.len() - 1)?;

    for (idx, app) in apps.iter().enumerate() {
        println!("app_{}: {}", idx, app);
        writeln!(f, r#"
    .section .data
    .global app_{0}_start
    .global app_{0}_end
app_{0}_start:
    .incbin "{2}{1}.bin"
app_{0}_end:"#, idx, app, TARGET_PATH)?;
    }
    Ok(())
}
```

#### os/Cargo.toml

```
[package]
name = "os"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
riscv = {git = "https://github.com/rcore-os/riscv", features = ["inline-asm"]}
lazy_static = {version = "1.4.0", features = ["spin_no_std"]}

[features]
board_qemu = []
board_k210 = []
```

#### os/src/config.rs

``` rust
pub const USER_STACK_SIZE: usize = 4096 * 2;
pub const KERNEL_STACK_SIZE: usize = 4096 * 2;
pub const MAX_APP_NUM: usize = 4;
pub const APP_BASE_ADDRESS: usize = 0x80400000;
pub const APP_SIZE_LIMIT: usize = 0x20000;

#[cfg(feature = "board_k210")]
pub const CLOCK_FREQ: usize = 403000000 / 62;

#[cfg(feature = "board_qemu")]
pub const CLOCK_FREQ: usize = 12500000;
```

#### os/src/console.rs

```rust
use core::fmt::{self, Write};
use crate::sbi::console_putchar;

struct Stdout;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        for c in s.chars() {
            console_putchar(c as usize);
        }
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}



```

#### os/src/entry.asm

```assembly
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

#### os/src/lang_items.rs

```rust
use core::panic::PanicInfo;
use crate::sbi::shutdown;

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
    if let Some(location) = info.location() {
        println!("[kernel] Panicked at {}:{} {}", location.file(), location.line(), info.message().unwrap());
    } else {
        println!("[kernel] Panicked: {}", info.message().unwrap());
    }
    shutdown()
}

```

#### os/src/link_app.S

```assembly
    .align 3
    .section .data
    .global _num_app
_num_app:
    .quad 3
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_2_end

    .section .data
    .global app_0_start
    .global app_0_end
app_0_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/00write_a.bin"
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end
app_1_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/01write_b.bin"
app_1_end:

    .section .data
    .global app_2_start
    .global app_2_end
app_2_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/02write_c.bin"
app_2_end:

```



#### os/src/linker-qemu.ld

```
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
        *(.srodata .srodata.*)
    }

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }

    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}
```

#### os/src/sbi.rs

``` rust
#![allow(unused)]
const SBI_SET_TIMER:usize = 0;
const SBI_CONSOLE_PUTCHAR:usize = 1;
const SBI_CONSOLE_GETCHAR:usize = 2;
const SBI_CLEAR_IPI:usize = 3;
const SBI_SEND_IPI:usize = 4;
const SBI_REMOTE_FENCE_I:usize = 5;
const SBI_REMOTE_SFENCE_VMA:usize = 6;
const SBI_REMOTE_SFENCE_VMA_ASID:usize = 7;
const SBI_SHUTDOWN:usize = 8;

use core::arch::asm;
#[inline(always)]
fn sbi_call(which:usize, arg0:usize, arg1:usize, arg2: usize) -> usize{
	let mut ret;
	unsafe{
		asm!(
			"ecall",
			inlateout("x10") arg0 => ret,
			in("x11") arg1,
			in("x12") arg2,
			in("x17") which,
		);
	}
	ret
}

pub fn console_putchar(c:usize){
	sbi_call(SBI_CONSOLE_PUTCHAR, c, 0, 0);
}

pub fn console_getchar() -> usize {
	sbi_call(SBI_CONSOLE_GETCHAR, 0, 0, 0)
}

pub fn shutdown() -> ! {
	sbi_call(SBI_SHUTDOWN, 0, 0, 0);
	panic!("It should shutdown!");
}

pub fn set_timer(timer: usize) {
	sbi_call(SBI_SET_TIMER, timer, 0, 0);
}
// sbi 子模块有一个 set_timer 调用，是一个由 SEE 提供的标准 SBI 接口函数，它可以用来设置 mtimecmp 的值。
```

#### os/src/timer.rs

``` rust
use riscv::register::time;
use crate::sbi::set_timer;


pub fn get_time() -> usize{
    time::read()
}
// timer 子模块的 get_time 函数可以取得当前 mtime 计数器的值；

use crate::config::CLOCK_FREQ;

const TICKS_PER_SEC: usize = 100;

pub fn set_next_trigger() {
    set_timer(get_time() + CLOCK_FREQ / TICKS_PER_SEC);
}
// timer 子模块的 set_next_trigger 函数对 set_timer 进行了封装，
// 它首先读取当前 mtime 的值，然后计算出 10ms 之内计数器的增量，
// 再将 mtimecmp 设置为二者的和。这样，10ms 之后一个 S 特权级时钟中断就会被触发。

// 至于增量的计算方式，常数 CLOCK_FREQ 是一个预先获取到的各平台不同的时钟频率，
// 单位为赫兹，也就是一秒钟之内计数器的增量。它可以在 config 子模块中找到。
// CLOCK_FREQ 除以常数 TICKS_PER_SEC 即是下一次时钟中断的计数器增量值。

const MICRO_PRE_SEC: usize = 1_000_000;

pub fn get_time_ms() -> usize {
    time::read() / (CLOCK_FREQ / MICRO_PRE_SEC)
}
// timer 子模块的 get_time_us 以微秒为单位返回当前计数器的值，这让我们终于能对时间有一个具体概念了
```



#### os/src/sync/mod.rs

```rust
mod up;

pub use up::UPSafeCell;
```

#### os/src/sync/up.rs

```rust
use core::cell::{RefCell, RefMut};

/// Wrap a static data structure inside it so that we are
/// able to access it without any `unsafe`.
///
/// We should only use it in uniprocessor.
///
/// In order to get mutable reference of inner data, call
/// `exclusive_access`.
pub struct UPSafeCell<T> {
    /// inner data
    inner: RefCell<T>,
}

unsafe impl<T> Sync for UPSafeCell<T> {}

impl<T> UPSafeCell<T> {
    /// User is responsible to guarantee that inner struct is only used in
    /// uniprocessor.
    pub unsafe fn new(value: T) -> Self {
        Self { inner: RefCell::new(value) }
    }
    /// Panic if the data has been borrowed.
    pub fn exclusive_access(&self) -> RefMut<'_, T> {
        self.inner.borrow_mut()
    }
}
```

#### os/src/syscall/fs.rs

```rust
const FD_STDOUT: usize = 1;

pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize {
    match fd {
        FD_STDOUT => {
            let slice = unsafe { core::slice::from_raw_parts(buf, len) };
            let str = core::str::from_utf8(slice).unwrap();
            print!("{}", str);
            len as isize
        },
        _ => {
            panic!("Unsupported fd in sys_write!");
        }
    }
}
```

#### os/src/syscall/mod.rs

```rust
const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;
const SYSCALL_YIELD: usize = 124;
const SYSCALL_GET_TIME: usize = 169;

mod fs;
mod process;

use fs::*;
use process::*;

pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize{
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        SYSCALL_YIELD => sys_yield(),
        SYSCALL_GET_TIME => sys_get_time(),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```



#### os/src/syscall/process.rs

``` rust
use crate::task::{
    suspend_current_and_run_next,
    exit_current_and_run_next,
};

pub fn sys_exit(exit_code: i32) -> ! {
    println!("[kernel] Application exited with code {}", exit_code);
    // 在调用它之前我们打印应用的退出信息并输出它的退出码
    exit_current_and_run_next();
    // 基于 task 子模块提供的 exit_current_and_run_next 接口
    panic!("Unreachable in sys_exit!");
}

pub fn sys_yield() -> isize {
    suspend_current_and_run_next();
    // task 子模块提供的 suspend_current_and_run_next 接口
    0
}
// sys_yield 表示应用自己暂时放弃对CPU的当前使用权，进入 Ready 状态

use crate::timer::get_time_ms;

pub fn sys_get_time() -> isize {
    get_time_ms() as isize
}
```

#### os/src/task/context.rs

```rust
#[derive(Copy, Clone)]
#[repr(C)]
pub struct TaskContext {
    ra: usize,
    // 记录__switch 函数返回之后应该跳转到哪里继续执行
    // 从而在任务切换完成并 ret 之后能到正确的位置
    sp: usize,
    s: [usize; 12],
    // 对于一般的函数而言，Rust/C 编译器会在函数的起始位置自动生成代码来保存 s0~s11 这些被调用者保存的寄存器。
    // 但 __switch 是一个用汇编代码写的特殊函数，它不会被 Rust/C 编译器处理，
    // 所以我们需要在 __switch 中手动编写保存 s0~s11 的汇编代码
    // 不用保存其它寄存器是因为：其它寄存器中，
    // 属于调用者保存的寄存器是由编译器在高级语言编写的调用函数中自动生成的代码来完成保存的；
    // 还有一些寄存器属于临时寄存器，不需要保存和恢复。
}

impl TaskContext {
    pub fn zero_init() -> Self {
        Self {
            ra: 0,
            sp: 0,
            s: [0; 12],
        }
    }
    pub fn goto_restore(kstack_ptr: usize) -> Self {
        // 构造每个任务保存在任务控制块中的任务上下文
        extern "C" { fn __restore(); }
        Self {
            ra: __restore as usize,
            // 它设置任务上下文中的内核栈指针将任务上下文的 ra 寄存器设置为 __restore 的入口地址
            // 这样，在 __switch 从它上面恢复并返回之后就会直接跳转到 __restore ，
            // 此时栈顶是一个我们构造出来第一次进入用户态执行的 Trap 上下文
            sp: kstack_ptr,
            s: [0; 12],
        }
    }
}


```

#### os/src/task/switch.rs

```rust
use super::TaskContext;
use core::arch::global_asm;

global_asm!(include_str!("switch.S"));

extern "C" {
    pub fn __switch(
        current_task_cx_ptr: *mut TaskContext,
        next_task_cx_ptr: *const TaskContext
    );
    // 我们会将这段汇编代码中的全局符号 __switch 解释为一个 Rust 函数
    // 我们会调用该函数来完成切换功能而不是直接跳转到符号 __switch 的地址。
    // 因此在调用前后 Rust 编译器会自动帮助我们插入保存/恢复调用者保存寄存器的汇编代码
}

```



#### os/src/task/switch.S

```asm
.altmacro
.macro SAVE_SN n
    sd s\n, (\n+2)*8(a0)
.endm
.macro LOAD_SN n
    ld s\n, (\n+2)*8(a1)
.endm
    .section .text
    .globl __switch
__switch:
    # __switch(
    #     current_task_cx_ptr: *mut TaskContext,
    #     next_task_cx_ptr: *const TaskContext
    # )
    
    # 函数原型中的两个参数分别是当前 A 任务上下文指针 next_task_cx_ptr 
    # 和即将被切换到的 B 任务上下文指针 next_task_cx_ptr ，
    # 从 RISC-V 调用规范 可以知道它们分别通过寄存器 a0/a1 传入
    
    # save kernel stack of current task
    # 保存当前任务的栈段
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    # 保存当前 任务上下文
    sd ra, 0(a0)
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # restore ra & s0~s11 of next execution
    # 恢复下一个任务的上下文
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    # 恢复上一个任务的 栈段
    ld sp, 8(a1)
    ret


```

#### os/src/task.rs

```rust
use super::TaskContext;

#[derive(Copy, Clone)]
// 通过 #[derive(...)] 可以让编译器为你的类型提供一些 Trait 的默认实现。 
// 实现了 Clone Trait 之后就可以调用 clone 函数完成拷贝；
// 实现了 PartialEq Trait 之后就可以使用 == 运算符比较该类型的两个实例，
// 		从逻辑上说只有 两个相等的应用执行状态才会被判为相等，而事实上也确实如此。
// Copy 是一个标记 Trait，决定该类型在按值传参/赋值的时候采用移动语义还是复制语义。
pub struct TaskControlBlock {
    // 任务控制块
    pub task_status: TaskStatus,
    // 任务 运行状态
    pub task_cx: TaskContext,
    // 任务上下文
}

#[derive(Copy, Clone, PartialEq)]
pub enum TaskStatus {
    UnInit,  // 未初始化
    Ready,   // 准备运行
    Running, // 正在运行
    Exited,  // 已退出
}
```

#### os/src/task/mod.rs

``` rust
mod context;
mod switch;
mod task;

use crate::config::MAX_APP_NUM;
use crate::loader::{get_num_app, init_app_cx};
use lazy_static::*;
use switch::__switch;
use task::{TaskControlBlock, TaskStatus};
use crate::sync::UPSafeCell;

pub use context::TaskContext;

// 全局的任务管理器
pub struct TaskManager {
    num_app: usize,
    // 任务管理器管理的应用的数目
    // 它在 TaskManager 初始化之后就不会发生变化
    inner: UPSafeCell<TaskManagerInner>,
    // 而包裹在 TaskManagerInner 内的任务控制块数组 tasks 
    // 以及表示 CPU 正在执行的应用编号 current_task 会在执行应用的过程中发生变化：
    // 每个应用的运行状态都会发生变化，而 CPU 执行的应用也在不断切换。
    // 因此我们需要将 TaskManagerInner 包裹在 UPSafeCell 内
    // 以获取其内部可变性以及单核上安全的运行时借用检查能力
}

struct TaskManagerInner {
    tasks: [TaskControlBlock; MAX_APP_NUM],
    current_task: usize,
    // 只能通过它知道 CPU正在执行哪个应用，而不能推测出其他应用的任何信息
}

lazy_static! {
    pub static ref TASK_MANAGER: TaskManager = {
        let num_app = get_num_app();
        // 调用 loader 子模块提供的 get_num_app 接口获取链接到内核的应用总数
        let mut tasks = [
            TaskControlBlock {
                task_cx: TaskContext::zero_init(),
                task_status: TaskStatus::UnInit
            };
            MAX_APP_NUM
        ];
        // 创建一个初始化的 tasks 数组，其中的每个任务控制块的运行状态都是 UnInit ：表示尚未初始化
        for i in 0..num_app {
            tasks[i].task_cx = TaskContext::goto_restore(init_app_cx(i));
            // 对于每个任务，我们先调用 init_app_cx 构造该任务的 Trap 上下文
            // （包括应用入口地址和用户栈指针）并将其压入到内核栈顶
            // 接着调用 TaskContext::goto_restore 来构造每个任务保存在任务控制块中的任务上下文。
            // 它设置任务上下文中的内核栈指针将任务上下文的 ra 寄存器设置为 __restore 的入口地址
            // 这样，在 __switch 从它上面恢复并返回之后就会直接跳转到 __restore ，
            // 此时栈顶是一个我们构造出来第一次进入用户态执行的 Trap 上下文
            tasks[i].task_status = TaskStatus::Ready;
        }
        // 依次对每个任务控制块进行初始化，将其运行状态设置为 Ready ：表示可以运行，
        // 并初始化它的 任务上下文
        TaskManager {
            num_app,
            inner: unsafe { UPSafeCell::new(TaskManagerInner {
                tasks,
                current_task: 0,
            })},
        }
        // 创建 TaskManager 实例并返回
    };
}
//注意我们无需和第二章一样将 TaskManager 标记为 Sync ，
// 因为编译器可以根据 TaskManager 字段的情况自动推导出 TaskManager 是 Sync 的

impl TaskManager {
    fn run_first_task(&self) -> ! {
        let mut inner = self.inner.exclusive_access();
        let task0 = &mut inner.tasks[0];
        // 我们取出即将最先执行的编号为 0 的应用的任务上下文指针 next_task_cx_ptr 并希望能够切换过去。注意 __switch 有两个参数分别表示当前应用和即将切换到的应用的任务上下文指针，其第一个参数存在的意义是记录当前应用的任务上下文被保存在哪里，也就是当前应用内核栈的栈顶，这样之后才能继续执行该应用。但在 run_first_task 的时候，我们并没有执行任何应用， __switch 前半部分的保存仅仅是在启动栈上保存了一些之后不会用到的数据，自然也无需记录启动栈栈顶的位置。

// 因此，我们显式在启动栈上分配了一个名为 _unused 的任务上下文，并将它的地址作为第一个参数传给 __switch ，这样保存一些寄存器之后的启动栈栈顶的位置将会保存在此变量中。然而无论是此变量还是启动栈我们之后均不会涉及到，一旦应用开始运行，我们就开始在应用的用户栈和内核栈之间开始切换了。这里声明此变量的意义仅仅是为了避免覆盖到其他数据
        task0.task_status = TaskStatus::Running;
        let next_task_cx_ptr = &task0.task_cx as *const TaskContext;
        // 将编号为0 的任务 标记为下一个要切换到的任务
        drop(inner);
        let mut _unused = TaskContext::zero_init();
        // 初始化创建一个不会用的的 任务上下文 作为当前任务上下文
        // 由于之后切换时 找不到它的 ID，所有不用当心它对后续程序运行的影响
        // before this, we should drop local variables that must be dropped manually
        unsafe {
            __switch(
                &mut _unused as *mut TaskContext,
                next_task_cx_ptr,
            );
        }
        panic!("unreachable in run_first_task!");
    }

    fn mark_current_suspended(&self) {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].task_status = TaskStatus::Ready;
        // 其中，首先获得里层 TaskManagerInner 的可变引用，
        // 通过 sync/up.rs 里 UPSafeCell结构体的方法 exclusive_access 获取
    	// 然后根据其中记录的当前正在执行的应用 ID 对应在任务控制块数组 tasks 中修改状态
    }

    fn mark_current_exited(&self) {
        let mut inner = self.inner.exclusive_access();
        let current = inner.current_task;
        inner.tasks[current].task_status = TaskStatus::Exited;
        // 其中，首先获得里层 TaskManagerInner 的可变引用，
        // 通过 sync/up.rs 里 UPSafeCell结构体的方法 exclusive_access 获取
    	// 然后根据其中记录的当前正在执行的应用 ID 对应在任务控制块数组 tasks 中修改状态
    }

    fn find_next_task(&self) -> Option<usize> {
        let inner = self.inner.exclusive_access();
        let current = inner.current_task;
        (current + 1..current + self.num_app + 1)
            .map(|id| id % self.num_app)
            .find(|id| {
                inner.tasks[*id].task_status == TaskStatus::Ready
            })
        // TaskManagerInner 的 tasks 是一个固定的任务控制块组成的表，长度为 num_app ，
        // 可以用下标 0~num_app-1 来访问得到每个应用的控制状态。
        // 我们的任务就是找到 current_task 后面第一个状态为 Ready 的应用。
        // 因此从 current_task + 1 开始循环一圈，需要首先对 num_app 取模得到实际的下标，
        // 然后检查它的运行状态
    }

    fn run_next_task(&self) {
        if let Some(next) = self.find_next_task() {
            // 它会调用 find_next_task 方法尝试寻找一个运行状态为 Ready 的应用并返回其 ID
            let mut inner = self.inner.exclusive_access();
            // 获取全局任务管理器的 struct TaskManagerInner {
            // tasks: [TaskControlBlock; MAX_APP_NUM],
            // current_task: usize,
            // }
            let current = inner.current_task;
            // 获取当前任务的 ID
            inner.tasks[next].task_status = TaskStatus::Running;
            // 将下一个任务的状态标注为 Running
            inner.current_task = next;
            // 将当前任务 标注为下一个任务的 ID
            let current_task_cx_ptr = &mut inner.tasks[current].task_cx as *mut TaskContext;
            // 获取存放当前任务上下文的 地址
            let next_task_cx_ptr = &inner.tasks[next].task_cx as *const TaskContext;
            // 获取存放下一个任务上下文的 地址
            drop(inner);
            // 在实际切换之前我们需要手动 drop 掉我们获取到的 TaskManagerInner 
            // 的来自 UPSafeCell 的借用标记。
            // 因为一般情况下它是在函数退出之后才会被自动释放，从而 TASK_MANAGER 
            // 的 inner 字段得以回归到未被借用的状态，
            // 之后可以再借用。如果不手动 drop 的话，编译器会在 __switch 返回时，
            // 也就是当前应用被切换回来的时候才 drop，这期间我们都不能修改 TaskManagerInner ，
            // 甚至不能读（因为之前是可变借用），会导致内核 panic 报错退出。
            // 正因如此，我们需要在 __switch 前提早手动 drop 掉 inner
            // before this, we should drop local variables that must be dropped manually
            unsafe {
                __switch(
                    current_task_cx_ptr,
                    next_task_cx_ptr,
                );
            }
            // 如果能够找到下一个可运行的应用的话，
            // 我们就可以分别拿到当前应用 current_task_cx_ptr 
            // 和即将被切换到的应用 next_task_cx_ptr 的任务上下文指针，
            // 然后调用 __switch 接口进行切换
            // go back to user mode
        } else {
            // 找不到就 panic，此时 panic 后就会退出
            panic!("All applications completed!");
        }
    }
}

pub fn run_first_task() {
    TASK_MANAGER.run_first_task();
    // 它调用了全局任务管理器 TASK_MANAGER 的 run_first_task 方法。
}

fn run_next_task() {
    TASK_MANAGER.run_next_task();
    // 它调用了全局任务管理器 TASK_MANAGER 的 run_next_task 方法。
}

fn mark_current_suspended() {
    TASK_MANAGER.mark_current_suspended();
    // 它调用了全局任务管理器 TASK_MANAGER 的 mark_current_suspended 方法。
}

fn mark_current_exited() {
    TASK_MANAGER.mark_current_exited();
    // 它调用了全局任务管理器 TASK_MANAGER 的 mark_current_exited 方法。
}

pub fn suspend_current_and_run_next() {
    mark_current_suspended();
    // 先修改当前应用的运行状态
    run_next_task();
    // 然后尝试切换到下一个应用
}

pub fn exit_current_and_run_next() {
    mark_current_exited();
    // 先修改当前应用的运行状态
    run_next_task();
    // 然后尝试切换到下一个应用
}
```

#### os/src/trap/context.rs

```rust
use riscv::register::sstatus::{Sstatus, self, SPP};

#[repr(C)]
pub struct TrapContext {
    pub x: [usize; 32],
    pub sstatus: Sstatus,
    pub sepc: usize,
}

impl TrapContext {
    pub fn set_sp(&mut self, sp: usize) { self.x[2] = sp; }
    pub fn app_init_context(entry: usize, sp: usize) -> Self {
        let mut sstatus = sstatus::read();
        sstatus.set_spp(SPP::User);
        let mut cx = Self {
            x: [0; 32],
            sstatus,
            sepc: entry,
        };
        cx.set_sp(sp);
        cx
    }
}

```

#### os/src/trap/mod.rs

```rust
mod context;

use riscv::register::{
    mtvec::TrapMode,
    stvec,
    scause::{
        self,
        Trap,
        Exception,
        Interrupt,
    },
    stval,
    sie,
};

use crate::syscall::syscall;
// use crate::batch::run_next_app;
use core::arch::global_asm;
use crate::task::{
    exit_current_and_run_next,
    suspend_current_and_run_next,
};
use crate::timer::set_next_trigger;
global_asm!(include_str!("trap.S"));

pub fn init(){
    extern "C" {fn __alltraps();}

    unsafe{
        stvec::write(__alltraps as usize, TrapMode::Direct);
    }
}

#[no_mangle]
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext{
    let scause = scause::read();
    let stval = stval::read();
    match scause.cause(){
        Trap::Exception(Exception::UserEnvCall) => {
            cx.sepc += 4;
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;
        }
        Trap::Exception(Exception::StoreFault) |
        Trap::Exception(Exception::StorePageFault) => {
            println!("[kernel] PageFault in application, bad addr = {:#x}, bad instruction = {:#x}, kernel killed it.", stval, cx.sepc);
            exit_current_and_run_next();
        }
        Trap::Exception(Exception::IllegalInstruction) => {
            println!("[kernel] IllegalInstruction in application, kernel killed it.");
            exit_current_and_run_next();
        }
        Trap::Interrupt(Interrupt::SupervisorTimer) => {
            set_next_trigger();
            suspend_current_and_run_next();
        }
        // 当发现触发了一个 S 特权级时钟中断的时候，首先重新设置一个 10ms 的计时器，
        // 然后调用上一小节提到的 suspend_current_and_run_next 函数暂停当前应用并切换到下一个
        _ => {
            panic!("Unsupported trap {:?}, stval = {:#x}!", scause.cause(), stval);
        }
    }
    cx
}

pub use context::TrapContext;

// use riscv::register::sie;

pub fn enable_timer_interrupt() {
    unsafe { sie::set_stimer(); }
}
```

#### os/src/trap/trap.S

```assembly
.altmacro
.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm
.macro LOAD_GP n
    ld x\n, \n*8(sp)
.endm
    .section .text
    .globl __alltraps
    .globl __restore
    .align 2
__alltraps:
    csrrw sp, sscratch, sp
    # now sp->kernel stack, sscratch->user stack
    # allocate a TrapContext on kernel stack
    addi sp, sp, -34*8
    # save general-purpose registers
    sd x1, 1*8(sp)
    # skip sp(x2), we will save it later
    sd x3, 3*8(sp)
    # skip tp(x4), application does not use it
    # save x5~x31
    .set n, 5
    .rept 27
        SAVE_GP %n
        .set n, n+1
    .endr
    # we can use t0/t1/t2 freely, because they were saved on kernel stack
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # read user stack from sscratch and save it on the kernel stack
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # set input argument of trap_handler(cx: &mut TrapContext)
    mv a0, sp
    call trap_handler

__restore:
    # now sp->kernel stack(after allocated), sscratch->user stack
    # restore sstatus/sepc
    # 它 不再需要 在开头 mv sp, a0 了。因为在 __switch 之后，
    # sp 就已经正确指向了我们需要的 Trap 上下文地址
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # restore general-purpuse registers except sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # release TrapContext on kernel stack
    addi sp, sp, 34*8
    # now sp->kernel stack, sscratch->user stack
    csrrw sp, sscratch, sp
    sret

```



#### os/src/loader.rs

```rust
use crate::trap::TrapContext;
use crate::config::*;
use core::arch::asm;

#[repr(align(4096))]
#[derive(Copy, Clone)]
struct KernelStack {
    data: [u8; KERNEL_STACK_SIZE],
}

#[repr(align(4096))]
#[derive(Copy, Clone)]
struct UserStack {
    data: [u8; USER_STACK_SIZE],
}

static KERNEL_STACK: [KernelStack; MAX_APP_NUM] = [
    KernelStack { data: [0; KERNEL_STACK_SIZE], };
    MAX_APP_NUM
];

static USER_STACK: [UserStack; MAX_APP_NUM] = [
    UserStack { data: [0; USER_STACK_SIZE], };
    MAX_APP_NUM
];

impl KernelStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + KERNEL_STACK_SIZE
    }
    pub fn push_context(&self, trap_cx: TrapContext) -> usize {
        let trap_cx_ptr = (self.get_sp() - core::mem::size_of::<TrapContext>()) as *mut TrapContext;
        unsafe { *trap_cx_ptr = trap_cx; }
        trap_cx_ptr as usize
    }
}

impl UserStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + USER_STACK_SIZE
    }
}

fn get_base_i(app_id: usize) -> usize {
    APP_BASE_ADDRESS + app_id * APP_SIZE_LIMIT
}
// 获取第 i 个应用的起始地址

pub fn get_num_app() -> usize {
    extern "C" { fn _num_app(); }
    unsafe { (_num_app as usize as *const usize).read_volatile() }
}
// 获取 _num_app, 即应用的个数

pub fn load_apps() {
    extern "C" { fn _num_app(); }
    let num_app_ptr = _num_app as usize as *const usize;
    let num_app = get_num_app();
    let app_start = unsafe {
        core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1)
    };
    // clear i-cache first
    unsafe { asm!("fence.i"); }
    // load apps
    for i in 0..num_app {
        let base_i = get_base_i(i);
        // 获取第 i 个应用的起始地址
        (base_i..base_i + APP_SIZE_LIMIT).for_each(|addr| unsafe {
            (addr as *mut u8).write_volatile(0)
        });
        // 将用于存放第 i 个应用的地址区间 用 0 填充
        let src = unsafe {
            core::slice::from_raw_parts(app_start[i] as *const u8, app_start[i + 1] - app_start[i])
        };
        // 获取第 i 个应用在 .data 段中的起始地址 和 长度
        let dst = unsafe {
            core::slice::from_raw_parts_mut(base_i as *mut u8, src.len())
        };
        // 将刚才用 0 填充的起始地址 和 应用的长度封装起来
        dst.copy_from_slice(src);
        // 将第 i 个应用 从 .data 段加载到 上面对应的地址空间
    }
}

pub fn init_app_cx(app_id: usize) -> usize {
    // 构造该任务的 Trap 上下文（包括应用入口地址和用户栈指针）并将其压入到内核栈顶
    KERNEL_STACK[app_id].push_context(
        TrapContext::app_init_context(get_base_i(app_id), USER_STACK[app_id].get_sp()),
    )
}

```

#### os/src/main.rs

``` rust
#![no_std]
#![no_main]
#![feature(panic_info_message)]

#[macro_use]
mod console;
mod lang_items;
mod sbi;
mod syscall;
mod trap;
// mod batch;
mod loader;
mod config;
mod task;
mod sync;
mod timer;


use core::arch::global_asm;
global_asm!(include_str!("entry.asm"));
global_asm!(include_str!("link_app.S"));

fn clear_bss(){
	extern "C" {
		fn sbss();
		fn ebss();
	}
	// (sbss as usize..ebss as usize).for_each(|a| {
	// 	unsafe { (a as *mut u8).write_volatile(0)}
	// });

	unsafe {
		core::slice::from_raw_parts_mut(
			sbss as usize as *mut u8,
			ebss as usize - sbss as usize,
		).fill(0);
	}
}

#[no_mangle]
pub fn rust_main() -> ! {
	clear_bss();
//	loop {}
	// println!("Hello, world");
	// println!("I am zk");
	// println!("\x1b[31mhello world\x1b[0m");
	// extern "C" {
	// 	fn stext();
	// 	fn etext();
	// }
	// println!(".text location: {}", stext as usize);
	// panic!("Shutdown machine!");
	println!("[kernel] Hello, world!");
	// println!("test {} a {} text.", 3, 5);
	trap::init();
// 	batch::init();
// 	batch::run_next_app();
	loader::load_apps();
	trap::enable_timer_interrupt();
    // 为了避免 S 特权级时钟中断被屏蔽，我们需要在执行第一个应用之前进行一些初始化设置
    // 设置了 sie.stie 使得 S 特权级时钟中断不会被屏蔽
	timer::set_next_trigger();
    // 设置第一个 10ms 的计时器
	task::run_first_task();
	panic!("Unreachable in rust_main!");
}



```



#### os/Makefile

```makefile
#Building
TARGET := riscv64gc-unknown-none-elf
MODE := release
KERNEL_ELF := target/$(TARGET)/$(MODE)/os
KERNEL_BIN := $(KERNEL_ELF).bin
DISASM_TMP := target/$(TARGET)/$(MODE)/asm

#BOARD
BOARD ?= qemu
SBI ?= rustsbi
BOOTLOADER := ../bootloader/$(SBI)-$(BOARD).bin
K210_BOOTLOADER_SIZE := 131072

#KERNEL ENTRY
ifeq ($(BOARD), qemu)
	KERNEL_ENTRY_PA := 0x80200000
else ifeq ($(BOARD), k210)
	KERNEL_ENTRY_PA := 0x80020000
endif

#Run K210
K210-SERIALPORT = /dev/ttyUSB0
K210-BURNER = ../tools/kflash.py

#Binutils
OBJDUMP := rust-objdump --arch-name=riscv64
OBJCOPY := rust-objcopy --binary-architecture=riscv64

#Disassembly
DISASM ?= -x

build: env switch-check $(KERNEL_BIN)

switch-check:
ifeq ($(BOARD), qemu)
	(which last-qemu) || (rm -f last-k210 && touch last-qemu && make clean)
else ifeq ($(BOARD), k210)
	(which last-k210) || (rm -f last-qemu && touch last-k210 && make clean)
endif

env:
	(rustup target list | grep "riscv64gc-unknown-none-elf (installed)") || rustup target add $(TARGET)
	cargo install cargo-binutils --vers =0.3.3
	rustup component add rust-src
	rustup component add llvm-tools-preview

$(KERNEL_BIN): kernel
	@$(OBJCOPY) $(KERNEL_ELF) --strip-all -O binary $@

kernel:
	@cd ../user && make build
	@echo Platform: $(BOARD)
	@cp src/linker-$(BOARD).ld src/linker.ld
	@cargo build --release --features "board_$(BOARD)"
	# 这里多了 --features "board_$(BOARD)"
	@rm src/linker.ld

clean:
	@cargo clean

disasm: kernel
	@$(OBJDUMP) $(DISASM) $(KERNEL_ELF) | less

disasm-vim: kernel
	@$(OBJDUMP) $(DISASM) $(KERNEL_ELF) > $(DISASM_TMP)
	@vim $(DISASM_TMP)
	@rm $(DISASM_TMP)

run: run-inner

run-inner: build
ifeq ($(BOARD), qemu)
	@qemu-system-riscv64 \
		-machine virt \
		-nographic \
		-bios $(BOOTLOADER) \
		-device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA)
else 
	(which $(K210-BURNER)) || (cd .. && git clone https://github.com/sipeed/kflash.py.git && mv kflash.py tools)
	@cp $(BOOTLOADER) $(BOOTLOADER).copy
	@dd if=$(KERNEL_BIN) of=$(BOOTLOADER).copy bs=$(K210_BOOTLOADER_SIZE) seek=1
	@mv $(BOOTLOADER).copy $(KERNEL_BIN)
	@sudo chmod 777 $(K210-SERIALPORT) 
	python3 $(K210-BURNER) -p $(K210-SERIALPORT) -b 1500000 $(KERNEL_BIN)
	python3 -m serial.tools.miniterm --eol LF --dtr 0 --rts 0 --filter direct $(K210-SERIALPORT) 115200
endif

debug: build
	@tmux new-session -d \
		"qemu-system-riscv64 -machine virt -nographic -bios $(BOOTLOADER) -device loader,file=$(KERNEL_BIN),addr=$(KERNEL_ENTRY_PA) -s -S" && \
		tmux split-window -h "riscv64-unknown-elf-gdb -ex 'file $(KERNEL_ELF)' -ex 'set arch riscv64:riscv64' -ex 'target remote localhost:1234'" && \
		tmux -2 attach-session -d
	
.PHONY: build env kernel clean disasm disasm-vim run-inner switch-check
```



### 在 user 文件夹下

#### user/.cargo/config

```
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-args=-Tsrc/linker.ld",
]

```

#### user/Cargo.toml

```
[package]
name = "user_lib"
version = "0.1.0"
authors = ["Yifan Wu <shinbokuow@163.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]


```



#### user/build.py

```python
import os

base_address = 0x80400000
step = 0x20000
linker = 'src/linker.ld'

app_id = 0
apps = os.listdir('src/bin')
# 得到一个列表， 里面是 文件名
apps.sort()
for app in apps:
    app = app[:app.find('.')]
    # app 变成不带后缀的文件名
    lines = []
    lines_before = []
    with open(linker, 'r') as f:
        for line in f.readlines():
            lines_before.append(line)
            line = line.replace(hex(base_address), hex(base_address+step*app_id))
            lines.append(line)
    with open(linker, 'w+') as f:
        f.writelines(lines)
    # 找到 src/linker.ld 中的 BASE_ADDRESS = 0x80400000; 
    # 这一行，并将后面的地址替换为和当前应用对应的一个地址
    os.system('cargo build --bin %s --release' % app)
    # 使用 --bin 参数来只构建某一个应用
    # 那就是对每个应用分别进行编译 改变其 起始地址
    print('[build.py] application %s start with address %s' %(app, hex(base_address+step*app_id)))
    with open(linker, 'w+') as f:
        f.writelines(lines_before)
    # 将 linker.ld 复原， 现在的 base_address = 0x80400000+step*app_id
    # 恢复为 0x80400000
    app_id = app_id + 1

```

#### user/src/console.rs

```rust
use core::fmt::{self, Write};
use super::write;

struct Stdout;

const STDOUT: usize = 1;

impl Write for Stdout {
    fn write_str(&mut self, s: &str) -> fmt::Result {
        write(STDOUT, s.as_bytes());
        Ok(())
    }
}

pub fn print(args: fmt::Arguments) {
    Stdout.write_fmt(args).unwrap();
}

#[macro_export]
macro_rules! print {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!($fmt $(, $($arg)+)?));
    }
}

#[macro_export]
macro_rules! println {
    ($fmt: literal $(, $($arg: tt)+)?) => {
        $crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
    }
}
```

#### user/src/lang_items.rs

```rust
#[panic_handler]
fn panic_handler(panic_info: &core::panic::PanicInfo) -> ! {
    let err = panic_info.message().unwrap();
    if let Some(location) = panic_info.location() {
        println!("Panicked at {}:{}, {}", location.file(), location.line(), err);
    } else {
        println!("Panicked: {}", err);
    }
    loop {}
}
```

#### user/src/lib.rs

```rust
#![no_std]
#![feature(panic_info_message)]
#![feature(linkage)]

#[macro_use]
pub mod console;
mod syscall;
mod lang_items;

#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start() -> ! {
    clear_bss();
    exit(main());
    panic!("unreachable after sys_exit!");
}

#[linkage = "weak"]
#[no_mangle]
fn main() -> i32 {
    panic!("Cannot find main");
}

fn clear_bss(){
    extern "C" {
        fn start_bss();
        fn end_bss();
    }
    (start_bss as usize..end_bss as usize).for_each(|addr|{
        unsafe{ (addr as *mut u8).write_volatile(0);}
    });
}

use syscall::*;

pub fn write(fd: usize, buf: &[u8]) -> isize {sys_write(fd, buf)}
pub fn exit(exit_code: i32) -> isize {sys_exit(exit_code)}
pub fn yield_() -> isize { sys_yield() }
pub fn get_time() -> isize { sys_get_time() }
```

#### user/src/linker.ld

```
OUTPUT_ARCH(riscv)
ENTRY(_start)

BASE_ADDRESS = 0x80400000;

SECTIONS
{
    . = BASE_ADDRESS;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    .bss : {
        start_bss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
        end_bss = .;
    }
    /DISCARD/ : {
        *(.eh_frame)
        *(.debug*)
    }
}
```



#### user/src/syscall.rs

```rust
use core::arch::asm;

const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;
const SYSCALL_YIELD: usize = 124;
const SYSCALL_GET_TIME: usize = 169;

fn syscall(id: usize, args: [usize; 3]) -> isize{
    let mut ret: isize;
    unsafe{
        asm!(
            "ecall",
            inlateout("x10") args[0] => ret,
            in("x11") args[1],
            in("x12") args[2],
            in("x17") id
        );
    }
    ret
}

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize{
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

pub fn sys_exit(exit_code: i32) -> isize {
    syscall(SYSCALL_EXIT, [exit_code as usize, 0, 0])
}

pub fn sys_yield() -> isize {
    syscall(SYSCALL_YIELD, [0, 0, 0])
}

pub fn sys_get_time() -> isize {
    syscall(SYSCALL_GET_TIME, [0, 0, 0])
}
```

#### user/src/bin/00power_3.rs

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

const LEN: usize = 100;

#[no_mangle]
fn main() -> i32 {
    let p = 3u64;
    let m = 998244353u64;
    let iter: usize = 200000;
    let mut s = [0u64; LEN];
    let mut cur = 0usize;
    s[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        s[next] = s[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_3 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, s[cur], m);
    println!("Test power_3 OK!");
    0
}
```

#### user/src/bin/01power_5.rs

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

const LEN: usize = 100;

#[no_mangle]
fn main() -> i32 {
    let p = 5u64;
    let m = 998244353u64;
    let iter: usize = 140000;
    let mut s = [0u64; LEN];
    let mut cur = 0usize;
    s[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        s[next] = s[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_5 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, s[cur], m);
    println!("Test power_5 OK!");
    0
}

```



#### user/src/bin/02power_7.rs

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

const LEN: usize = 100;

#[no_mangle]
fn main() -> i32 {
    let p = 7u64;
    let m = 998244353u64;
    let iter: usize = 160000;
    let mut s = [0u64; LEN];
    let mut cur = 0usize;
    s[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        s[next] = s[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_7 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, s[cur], m);
    println!("Test power_7 OK!");
    0
}

```

#### user/src/bin/03sleep.rs

``` rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

use user_lib::{get_time, yield_};

#[no_mangle]
fn main() -> i32 {
    let current_timer = get_time();
    let wait_for = current_timer + 3000;
    while get_time() < wait_for {
        yield_();
    }
    println!("Test sleep OK!");
    0
}
```



#### user/Makefile

```makefile
TARGET := riscv64gc-unknown-none-elf
MODE := release
APP_DIR := src/bin
TARGET_DIR := target/$(TARGET)/$(MODE)
APPS := $(wildcard $(APP_DIR)/*.rs)
ELFS := $(patsubst $(APP_DIR)/%.rs, $(TARGET_DIR)/%, $(APPS))
BINS := $(patsubst $(APP_DIR)/%.rs, $(TARGET_DIR)/%.bin, $(APPS))

OBJDUMP := rust-objdump --arch-name=riscv64
OBJCOPY := rust-objcopy --binary-architecture=riscv64

elf: $(APPS)
	@python3 build.py

binary: elf
	$(foreach elf, $(ELFS), $(OBJCOPY) $(elf) --strip-all -O binary $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.bin, $(elf));)

build: binary

clean:
	@cargo clean

.PHONY: elf binary build clean
```



## 执行流程

1. - os/src/main.rs
        - 导入 entry.asm

     - 设置函数栈， 跳转到rust_main
        - 导入 link_app.S
            - 这里全是数据段， 可执行文件在部分数据段中
        - 调用 clear_bass 函数， 清除 除 .bss.stack 以外的 .bss 段
        - 调用 println! 打印 [kernel] Hello, world! , 此时处于内核态
        - 调用 trap::init

2. os/src/trap/mod.rs

   - 我们需要修改stvec 寄存器来指向正确的Trap处理入口点
       将stvec设置为Direct模式指向它的地址

       **注意：由于前面有导入 trap.S， 所有这里知道 Trap的处理入口**

       **即 __alltraps**

   - 调用 loader::load_apps

3. os/src/loader.rs

   - 调用 get_num_app 获取应用个数
   - 获取第一个程序的入口地址
   - 刷新指令缓冲区
   - 调用 get_base_i 获取 每个应用程序的起始地址
   - 将每个应用起始地址到 应用空间限制范围的地址区间 用 0 填充
   - 获取每个应用在 .data 段的起始地址和大小 和 在将会放置该程序的起始地址和程序大小
   - 调用 copy_from_slice 将程序从 .data 段复制到 相应的地址区间
   - 至此，程序加载完毕

4. os/src/main.rs

   - 调用 trap::enable_timer_interrupt

5. os/src/trap/mod.rs

   - 调用 sie::set_stimer 使得 S 特权级时钟中断不会被屏蔽

 6. os/src/main.rs

     - 调用 timer::set_next_trigger

7. os/src/timer.rs

   - 调用 get_time 获得当前时间， 即 mtime 的值
   - 估算出 10 ms 后 mtime 的值
   - 再将估算的值赋值给 mtimecmp

8. os/src/main.rs

   - 调用 task 模块的 run_first_task

  9. os/src/task/mod.rs

     - 首先，静态加载一个全局的任务管理器

         TaskManager 结构 的实例 TASK_MANAGER,包含 num_app 和 inner

         inner 包括 数组 tasks 和 current_task

         数组 tasks 是 TaskControlBlock 结构的数组

         TaskControlBlock 结构体 包括 任务状态 task_status 和 任务上下文 TaskContext

         TaskContext 包含 那14个寄存器

     - 对任务数组进行初始化，将每个任务都进行初始化

     - 对任务上下文调用 zero_init 进行初始化，即将这 14 个寄存器都初始化为 0

     - 将任务的状态 设置为 UnInit, 即未初始化

     - 对于每个任务，先调用 init_app_cx, 构造该任务的 Trap 上下文

     - 在 os/src/loader.rs 

         - 获取每个应用的 起始地址 和 栈针
         - 初始化它的 Trap 上下文
         - 在 os/src/trap/context.rs
             - 获取当前的 sstatus 并存放在结构中
             - 将这个 sstatus 结构的 SPP置位 User
             - 将通用寄存器 全置为 0
             - 将 sepc 设为传入的应用的地址 entry
             - 将 Trap上下文的 sp 置为 传入的 内核栈地址 sp
             - 返回 Trap 上下文
         - 将 Trap 上下文 push 到内核栈中
         - 返回一个 执行改上下文的 指针

     - 将指向 Trap 上下文的 指针 传入到 goto_restoren 函数中

     - 用它初始化任务上下文 TaskContext

     - 将任务上下文返回给每个任务

     - 将每个任务都置为 Ready

     - 最后，获得一个任务管理器，它包含 任务数，任务数组，当前任务的编号

     - 这样，静态加载器的任务完成

     - 调用 run_first_task

         - 将第一个任务的状态置为 Running
         - 创建一个指针 指向 第一个任务的任务上下文
         - 构造一个 只有任务上下文结构而无真正内容的 任务上下文
         - 将 这两个上下文的 指针送给 切换函数 _switch
         - _switch 函数在将他们送到 汇编函数 _switch 中
         - 保存当前任务的 栈段， 保存当前 任务上下文， 恢复下一个任务上下文， 恢复下一个任务的 栈段
         - ret 返回到 下一个任务，进入到用户态

 10. user/src/bin/00power_3.rs

     - 正常运行
     - 达到时间后，中断触发， 进入 Trap

     - 进入到 __alltraps, 保存完上下文后，进入到 trap_handler
     - 判断出是 时钟中断，先设置下一个中断的时间
     - 再悬置当前任务，跳转到下一个任务

  11. 当应用完成了， 打印 Test power_3 OK!

      - user/src/lib.rs 调用 exit，exit 调用 sys_exit
      - user/src/syscall.rs 中 sys_exit 进行 系统调用 syscall , 引发 ecall
      - 跳转到 __alltraps
      - 跳转到 os/src/syscall/mod.rs 中的 sys_exit
      - os/src/syscall/process.rs 中的 sys_exit
          - 先打印 [kernel] Application exited with code 0
          - 然后调用 exit_current_and_run_next

  12. 再次返回到 os/src/task/mod.rs

      - 先将 当前任务的状态标记为 Exited
      - 再调用 run_next_task

13. user/src/bin/sleep.rs

    - 单出现 yield_ 时，和上一节一样

  14. 如果所有的任务都完成了，就 panic 打印出 All applications completed!

      - 打印 panic 的位置

      - 调用 shutdown 关机

      

