

os实现分时多任务系统与抢占式调用

## 第三章代码树

分时多任务的为例

```
1 ./os/src
 2 Rust        25 Files    1415 Lines
 3 Assembly     3 Files      88 Lines
 4 
 5 ├── bootloader
 6 │   ├── rustsbi-k210.bin
 7 │   └── rustsbi-qemu.bin
 8 ├── LICENSE
 9 ├── os
10 │   ├── build.rs
11 │   ├── Cargo.lock
12 │   ├── Cargo.toml
13 │   ├── Makefile
14 │   └── src
15 │       ├── config.rs(修改：新增一些内存管理的相关配置)
16 │       ├── console.rs
17 │       ├── entry.asm
18 │       ├── lang_items.rs
19 │       ├── link_app.S
20 │       ├── linker-k210.ld(修改：将跳板页引入内存布局)
21 │       ├── linker-qemu.ld(修改：将跳板页引入内存布局)
22 │       ├── loader.rs(修改：仅保留获取应用数量和数据的功能)
23 │       ├── main.rs(修改)
24 │       ├── mm(新增：内存管理的 mm 子模块)
25 │       │   ├── address.rs(物理/虚拟 地址/页号的 Rust 抽象)
26 │       │   ├── frame_allocator.rs(物理页帧分配器)
27 │       │   ├── heap_allocator.rs(内核动态内存分配器)
28 │       │   ├── memory_set.rs(引入地址空间 MemorySet 及逻辑段 MemoryArea 等)
29 │       │   ├── mod.rs(定义了 mm 模块初始化方法 init)
30 │       │   └── page_table.rs(多级页表抽象 PageTable 以及其他内容)
31 │       ├── sbi.rs
32 │       ├── sync
33 │       │   ├── mod.rs
34 │       │   └── up.rs
35 │       ├── syscall
36 │       │   ├── fs.rs(修改：基于地址空间的 sys_write 实现)
37 │       │   ├── mod.rs
38 │       │   └── process.rs
39 │       ├── task
40 │       │   ├── context.rs(修改：构造一个跳转到不同位置的初始任务上下文)
41 │       │   ├── mod.rs(修改，详见文档)
42 │       │   ├── switch.rs
43 │       │   ├── switch.S
44 │       │   └── task.rs(修改，详见文档)
45 │       ├── timer.rs
46 │       └── trap
47 │           ├── context.rs(修改：在 Trap 上下文中加入了更多内容)
48 │           ├── mod.rs(修改：基于地址空间修改了 Trap 机制，详见文档)
49 │           └── trap.S(修改：基于地址空间修改了 Trap 上下文保存与恢复汇编代码)
50 ├── README.md
51 ├── rust-toolchain
52 ├── tools
53 │   ├── kflash.py
54 │   ├── LICENSE
55 │   ├── package.json
56 │   ├── README.rst
57 │   └── setup.py
58 └── user
59 
    ├── build.py(移除)
60     ├── Cargo.toml
61     ├── Makefile
62     └── src
63         ├── bin
64         │   ├── 00power_3.rs
65         │   ├── 01power_5.rs
66         │   ├── 02power_7.rs
67         │   └── 03sleep.rs
68         ├── console.rs
69         ├── lang_items.rs
70         ├── lib.rs
71         ├── linker.ld(修改：将所有应用放在各自地址空间中固定的位置)
72         └── syscall.rs
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
authors = ["Yifan Wu <shinbokuow@163.com>"]
edition = "2018"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
riscv = { git = "https://github.com/rcore-os/riscv", features = ["inline-asm"] }
lazy_static = { version = "1.4.0", features = ["spin_no_std"] }
buddy_system_allocator = "0.6"
# 需将我们的动态内存分配器类型实例化为一个全局变量，并使用 #[global_allocator] 语义项标记即可。
# 由于该分配器的实现比较复杂，我们这里直接使用一个已有的伙伴分配器实现
bitflags = "1.2.1"
xmas-elf = "0.7.0"

[features]
board_qemu = []
board_k210 = []
```

#### os/src/config.rs

``` rust
pub const USER_STACK_SIZE: usize = 4096 * 2;
pub const KERNEL_STACK_SIZE: usize = 4096 * 2;
pub const KERNEL_HEAP_SIZE: usize = 0x30_0000;
pub const MEMORY_END: usize = 0x80800000;
// 我们硬编码整块物理内存的终止物理地址为 0x80800000 。 
// 而 之前 提到过物理内存的起始物理地址为 0x80000000 ，这意味着我们将可用内存大小设置为 8 MB
pub const PAGE_SIZE: usize = 0x1000;
pub const PAGE_SIZE_BITS: usize = 0xc;

pub const TRAMPOLINE: usize = usize::MAX - PAGE_SIZE + 1;
pub const TRAP_CONTEXT: usize = TRAMPOLINE - PAGE_SIZE;
/// Return (bottom, top) of a kernel stack in kernel space.
pub fn kernel_stack_position(app_id: usize) -> (usize, usize) {
    let top = TRAMPOLINE - app_id * (KERNEL_STACK_SIZE + PAGE_SIZE);
    let bottom = top - KERNEL_STACK_SIZE;
    (bottom, top)
}

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
        . = ALIGN(4K);
        strampoline = .;
        *(.text.trampoline);
        . = ALIGN(4K);
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
    sbss_with_stack = .;
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

#### os/src/mm/address.rs

Virtual Address (39bits)

38                         12    11                              0

Virtual Page Number	Page Offset(页内偏移)

VPN：27

Physical Address (56bits)

55                             12   11                                0

Physical Page Number	Page Offsett(页内偏移 )

PPN：44

```rust
// 页表中的页表项的索引其实是虚拟地址中的虚拟页号，页表项的重要内容是物理地址的物理页帧号。
// 为了能够灵活地在虚拟地址、物理地址、虚拟页号、物理页号之间进行各种转换
// 这里实现了各种转换函数

use crate::config::{PAGE_SIZE, PAGE_SIZE_BITS};
use super::PageTableEntry;
use core::fmt::{self, Debug, Formatter};

const PA_WIDTH_SV39: usize = 56;
const VA_WIDTH_SV39: usize = 39;
const PPN_WIDTH_SV39: usize = PA_WIDTH_SV39 - PAGE_SIZE_BITS;
const VPN_WIDTH_SV39: usize = VA_WIDTH_SV39 - PAGE_SIZE_BITS;

// 在分页内存管理中，地址转换的核心任务在于如何维护虚拟页号到物理页号的映射——也就是页表。
// 不过在具体实现它之前，我们先将地址和页号的概念抽象为 Rust 中的类型，
// 借助 Rust 的类型安全特性来确保它们被正确实现
/// Definitions
#[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
pub struct PhysAddr(pub usize);

#[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
pub struct VirtAddr(pub usize);

#[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
pub struct PhysPageNum(pub usize);

#[derive(Copy, Clone, Ord, PartialOrd, Eq, PartialEq)]
pub struct VirtPageNum(pub usize);
// 上面分别给出了物理地址、虚拟地址、物理页号、虚拟页号的 Rust 类型声明，
// 它们都是 Rust 的元组式结构体，可以看成 usize 的一种简单包装。
// 我们刻意将它们各自抽象出不同的类型而不是都使用与RISC-V 64硬件直接对应的 usize 基本类型，
// 就是为了在 Rust 编译器的帮助下，通过多种方便且安全的 类型转换 (Type Convertion) 来构建页表
/// Debugging

impl Debug for VirtAddr {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        f.write_fmt(format_args!("VA:{:#x}", self.0))
    }
}
impl Debug for VirtPageNum {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        f.write_fmt(format_args!("VPN:{:#x}", self.0))
    }
}
impl Debug for PhysAddr {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        f.write_fmt(format_args!("PA:{:#x}", self.0))
    }
}
impl Debug for PhysPageNum {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        f.write_fmt(format_args!("PPN:{:#x}", self.0))
    }
}

/// T: {PhysAddr, VirtAddr, PhysPageNum, VirtPageNum}
/// T -> usize: T.0
/// usize -> T: usize.into()

// 一般而言，当我们为类型 U 实现了 From<T> Trait 之后，
// 可以使用 U::from(_: T) 来从一个 T 类型的实例来构造一个 U 类型的实例；
// 而当我们为类型 U 实现了 Into<T> Trait 之后，对于一个 U 类型的实例 u ，
// 可以使用 u.into() 来将其转化为一个类型为 T 的实例。

// 当我们为 U 实现了 From<T> 之后，Rust 会自动为 T 实现 Into<U> Trait，
// 因为它们两个本来就是在做相同的事情。因此我们只需相互实现 From 就可以相互 From/Into 了

// 这些类型本身可以和 usize 之间互相转换
impl From<usize> for PhysAddr {
    fn from(v: usize) -> Self { Self(v & ( (1 << PA_WIDTH_SV39) - 1 )) }
                                     // 和 56 个 1 相与
}
impl From<usize> for PhysPageNum {
    fn from(v: usize) -> Self { Self(v & ( (1 << PPN_WIDTH_SV39) - 1 )) }
}
impl From<usize> for VirtAddr {
    fn from(v: usize) -> Self { Self(v & ( (1 << VA_WIDTH_SV39) - 1 )) }
                                     // 和 39 个 1 相与
}
impl From<usize> for VirtPageNum {
    fn from(v: usize) -> Self { Self(v & ( (1 << VPN_WIDTH_SV39) - 1 )) }
}
impl From<PhysAddr> for usize {
    fn from(v: PhysAddr) -> Self { v.0 }
}
impl From<PhysPageNum> for usize {
    fn from(v: PhysPageNum) -> Self { v.0 }
}
impl From<VirtAddr> for usize {
    fn from(v: VirtAddr) -> Self { v.0 }
}
impl From<VirtPageNum> for usize {
    fn from(v: VirtPageNum) -> Self { v.0 }
}

impl VirtAddr {
    // 从物理页号到物理地址的转换只需左移  位即可，
    // 但是物理地址需要保证它与页面大小对齐才能通过右移转换为物理页号。
    // 对于不对齐的情况，物理地址不能通过 From/Into 转换为物理页号，
    // 而是需要通过它自己的 floor 或 ceil 方法来进行下取整或上取整的转换
    pub fn floor(&self) -> VirtPageNum { VirtPageNum(self.0 / PAGE_SIZE) }
    pub fn ceil(&self) -> VirtPageNum  { VirtPageNum((self.0 - 1 + PAGE_SIZE) / PAGE_SIZE) }
    pub fn page_offset(&self) -> usize { self.0 & (PAGE_SIZE - 1) }
    pub fn aligned(&self) -> bool { self.page_offset() == 0 }
}
impl From<VirtAddr> for VirtPageNum {
    fn from(v: VirtAddr) -> Self {
        assert_eq!(v.page_offset(), 0);
        v.floor()
    }
}
impl From<VirtPageNum> for VirtAddr {
    fn from(v: VirtPageNum) -> Self { Self(v.0 << PAGE_SIZE_BITS) }
                                                  // 12
}
impl PhysAddr {
    pub fn floor(&self) -> PhysPageNum { PhysPageNum(self.0 / PAGE_SIZE) }
    // 整除
    pub fn ceil(&self) -> PhysPageNum { PhysPageNum((self.0 - 1 + PAGE_SIZE) / PAGE_SIZE) }
                                                                  // 4096
    pub fn page_offset(&self) -> usize { self.0 & (PAGE_SIZE - 1) }
                                               // 与  12    个    1 相与
    pub fn aligned(&self) -> bool { self.page_offset() == 0 }
}
// 地址和页号之间可以相互转换
impl From<PhysAddr> for PhysPageNum {
    fn from(v: PhysAddr) -> Self {
        assert_eq!(v.page_offset(), 0);
        // 但是物理地址需要保证它与页面大小对齐才能通过右移转换为物理页号
        // 即需要保证这个地址是一个页的开头，即末尾12位为 0
        v.floor()
    }
}

// 从物理页号到物理地址的转换只需左移12位即可，
impl From<PhysPageNum> for PhysAddr {
    fn from(v: PhysPageNum) -> Self { Self(v.0 << PAGE_SIZE_BITS) }
                                                  // 12
}

impl VirtPageNum {
    // 可以取出虚拟页号的三级页索引，并按照从高到低的顺序返回。注意它里面包裹的 usize 可能有 27 位，
    // 也有可能有 64 - 12 = 52 位，但这里我们是用来在多级页表上进行遍历，因此只取出低 27 位。
    pub fn indexes(&self) -> [usize; 3] {
        let mut vpn = self.0;
        let mut idx = [0usize; 3];
        for i in (0..3).rev() {
            // 注意：这里是逆序存放
            // 先放 第三级地址，存放的是下一个目录的地址
            idx[i] = vpn & 511;
                           // 9 个 1
            vpn >>= 9;
        }
        idx
    }
}

// 我们构造可变引用来直接访问一个物理页号 PhysPageNum 对应的物理页帧，
// 不同的引用类型对应于物理页帧上的一种不同的内存布局
impl PhysPageNum {
    // get_pte_array 返回的是一个页表项定长数组的可变引用，代表多级页表中的一个节点
    pub fn get_pte_array(&self) -> &'static mut [PageTableEntry] {
        let pa: PhysAddr = self.clone().into();
        // 都是先把物理页号转为物理地址 PhysAddr ，然后再转成 usize 形式的物理地址
        unsafe {
            core::slice::from_raw_parts_mut(pa.0 as *mut PageTableEntry, 512)
            // 我们直接将它转为裸指针用来访问物理地址指向的物理内存。
            // 在分页机制开启前，这样做自然成立；而开启之后，虽然裸指针被视为一个虚拟地址，
            // 但是上面已经提到，基于恒等映射，虚拟地址会映射到一个相同的物理地址，
            // 因此在也是成立的。注意，我们在返回值类型上附加了静态生命周期泛型 'static ，
            // 这是为了绕过 Rust 编译器的借用检查，实质上可以将返回的类型也看成一个裸指针，
            // 因为它也只是标识数据存放的位置以及类型。但与裸指针不同的是，
            // 无需通过 unsafe 的解引用访问它指向的数据，而是可以像一个正常的可变引用一样直接访问
        }
    }
    // get_bytes_array 返回的是一个字节数组的可变引用，可以以字节为粒度对物理页帧上的数据进行访问，
    // 前面进行数据清零就用到了这个方法
    pub fn get_bytes_array(&self) -> &'static mut [u8] {
        // 我们在返回值类型上附加了静态生命周期泛型 'static ，这是为了绕过 Rust 编译器的借用检查，
        // 实质上可以将返回的类型也看成一个裸指针，因为它也只是标识数据存放的位置以及类型。
        // 但与裸指针不同的是，无需通过 unsafe 的解引用访问它指向的数据，
        // 而是可以像一个正常的可变引用一样直接访问
        let pa: PhysAddr = self.clone().into();
        // 先将物理页号 转换为 物理地址
        unsafe {
            core::slice::from_raw_parts_mut(pa.0 as *mut u8, 4096)
            // 再转成 usize 形式的物理地址，直接将它转为裸指针用来访问物理地址指向的物理内存
        }
    }
    // get_mut 是个泛型函数，可以获取一个恰好放在一个物理页帧开头的类型为 T 的数据的可变引用
    pub fn get_mut<T>(&self) -> &'static mut T {
        let pa: PhysAddr = self.clone().into();
        // 先将物理页号 转换为 物理地址
        unsafe {
            (pa.0 as *mut T).as_mut().unwrap()
        }
    }
}

pub trait StepByOne {
    fn step(&mut self);
}
impl StepByOne for VirtPageNum {
    fn step(&mut self) {
        self.0 += 1;
    }
}

#[derive(Copy, Clone)]
pub struct SimpleRange<T> where
    T: StepByOne + Copy + PartialEq + PartialOrd + Debug, {
    l: T,
    r: T,
}
impl<T> SimpleRange<T> where
    T: StepByOne + Copy + PartialEq + PartialOrd + Debug, {
    pub fn new(start: T, end: T) -> Self {
        assert!(start <= end, "start {:?} > end {:?}!", start, end);
        Self { l: start, r: end }
    }
    pub fn get_start(&self) -> T { self.l }
    pub fn get_end(&self) -> T { self.r }
}
impl<T> IntoIterator for SimpleRange<T> where
    T: StepByOne + Copy + PartialEq + PartialOrd + Debug, {
    type Item = T;
    type IntoIter = SimpleRangeIterator<T>;
    fn into_iter(self) -> Self::IntoIter {
        SimpleRangeIterator::new(self.l, self.r)
    }
}
pub struct SimpleRangeIterator<T> where
    T: StepByOne + Copy + PartialEq + PartialOrd + Debug, {
    current: T,
    end: T,
}
impl<T> SimpleRangeIterator<T> where
    T: StepByOne + Copy + PartialEq + PartialOrd + Debug, {
    pub fn new(l: T, r: T) -> Self {
        Self { current: l, end: r, }
    }
}
impl<T> Iterator for SimpleRangeIterator<T> where
    T: StepByOne + Copy + PartialEq + PartialOrd + Debug, {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        if self.current == self.end {
            None
        } else {
            let t = self.current;
            self.current.step();
            Some(t)
        }
    }
}
pub type VPNRange = SimpleRange<VirtPageNum>;

```



#### os/src/mm/frame_allocator.rs

```rust
// 操作系统如果要建立页表（构建虚实地址映射关系），首先要能管理整个系统的物理内存，
// 这就需要知道整个计算机系统的物理内存空间的范围，物理内存中哪些区域是空闲可用的，
// 哪些区域放置内核/应用的代码和数据。操作系统内核能够以物理页帧为单位分配和回收物理内存
use super::{PhysAddr, PhysPageNum};
use alloc::vec::Vec;
use crate::sync::UPSafeCell;
use crate::config::MEMORY_END;
use lazy_static::*;
use core::fmt::{self, Debug, Formatter};

// 我们声明一个 FrameAllocator Trait 来描述一个物理页帧管理器需要提供哪些功能
trait FrameAllocator {
    fn new() -> Self;
    fn alloc(&mut self) -> Option<PhysPageNum>;
    fn dealloc(&mut self, ppn: PhysPageNum);
}

pub struct FrameTracker {
    pub ppn: PhysPageNum,
}

impl FrameTracker {
    pub fn new(ppn: PhysPageNum) -> Self {
        // page cleaning
        let bytes_array = ppn.get_bytes_array();
        // ppn 是一个结构体，元组结构体，里面是个 usize 类型的整数
        // 它有个 get_bytes_array 方法，返回一个字节数组的可变引用，
        // 返回一个指针，指向它所对应的物理页的地址
        // 以以字节为粒度对物理页帧上的数据进行访问
        
        for i in bytes_array {
            *i = 0;
        }
        // 我们将分配来的物理页帧的物理页号作为参数传给 FrameTracker 的 new 方法来创建一个 FrameTracker 实例。
        // 由于这个物理页帧之前可能被分配过并用做其他用途，我们在这里直接将这个物理页帧上的所有字节清零
        Self { ppn }
    }
}

impl Debug for FrameTracker {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        f.write_fmt(format_args!("FrameTracker:PPN={:#x}", self.ppn.0))
    }
}

impl Drop for FrameTracker {
    fn drop(&mut self) {
        frame_dealloc(self.ppn);
    }
    // 当一个 FrameTracker 生命周期结束被编译器回收的时候，我们需要将它控制的物理页帧回收掉
    // 这里我们只需为 FrameTracker 实现 Drop Trait 即可。当一个 FrameTracker 实例被回收的时候，
    // 它的 drop 方法会自动被编译器调用，
    // 通过之前实现的 frame_dealloc 我们就将它控制的物理页帧回收以供后续使用了
}


// 实现一种最简单的栈式物理页帧管理策略 StackFrameAllocator
pub struct StackFrameAllocator {
    current: usize,
    end: usize,
    // 物理页号区间 [ current , end ) 此前均 从未 被分配出去过
    recycled: Vec<usize>,
    // 向量 recycled 以后入先出的方式保存了被回收的物理页号
}

impl StackFrameAllocator {
    pub fn init(&mut self, l: PhysPageNum, r: PhysPageNum) {
        self.current = l.0;
        self.end = r.0;
    }
    // 而在它真正被使用起来之前，需要调用 init 方法将自身的 [current, end) 初始化为可用物理页号区间
}
impl FrameAllocator for StackFrameAllocator {
    fn new() -> Self {
        Self {
            current: 0,
            end: 0,
            recycled: Vec::new(),
        }
    }
    // 初始化非常简单。在通过 FrameAllocator 的 new 方法创建实例的时候，只需将区间两端均设为  ，然后创建一个新的向量
    fn alloc(&mut self) -> Option<PhysPageNum> {
        if let Some(ppn) = self.recycled.pop() {
            // 在分配 alloc 的时候，首先会检查栈 recycled 内有没有之前回收的物理页号，
            // 如果有的话直接弹出栈顶并返回
            Some(ppn.into())
        } else {
            if self.current == self.end {
                None
                // 极端情况下可能出现内存耗尽分配失败的情况：即 recycled 为空且 current == end 
                // 为了涵盖这种情况， alloc 的返回值被 Option 包裹，我们返回 None 即可
            } else {
                self.current += 1;
                Some((self.current - 1).into())
                // 否则的话我们只能从之前从未分配过的物理页号区间 [ current , end ) 上进行分配，
                // 我们分配它的左端点 current ，同时将管理器内部维护的 current 加 1 代表 current 已被分配了。
                // 在即将返回的时候，我们使用 into 方法将 usize 转换成了物理页号 PhysPageNum
            }
        }
    }
    fn dealloc(&mut self, ppn: PhysPageNum)
                               // 元组类型
    // 在回收 dealloc 的时候，我们需要检查回收页面的合法性，然后将其压入 recycled 栈中
        let ppn = ppn.0;
        // validity check
    // 该页面之前一定被分配出去过，因此它的物理页号一定  current ；
	// 该页面没有正处在回收状态，即它的物理页号不能在栈 recycled 中找到
        if ppn >= self.current || self.recycled
            .iter()
            .find(|&v| {*v == ppn})
            .is_some() {
            panic!("Frame ppn={:#x} has not been allocated!", ppn);
            // 我们通过 recycled.iter() 获取栈上内容的迭代器，
            // 然后通过迭代器的 find 方法试图寻找一个与输入物理页号相同的元素。
            // 其返回值是一个 Option ，如果找到了就会是一个 Option::Some ，
            // 这种情况说明我们内核其他部分实现有误，直接报错退出
        }
        // recycle
        self.recycled.push(ppn);
    }
}

type FrameAllocatorImpl = StackFrameAllocator;

lazy_static! {
    // 这里我们使用 UPSafeCell<T> 来包裹栈式物理页帧分配器。每次对该分配器进行操作之前，
    // 我们都需要先通过 FRAME_ALLOCATOR.exclusive_access() 拿到分配器的可变借用
    pub static ref FRAME_ALLOCATOR: UPSafeCell<FrameAllocatorImpl> = unsafe {
        	      // 它的本质是一个指针，一个引用， 指向的是 FrameAllocatorImpl 类型
        		  //                             即 StackFrameAllocator 类型
        UPSafeCell::new(FrameAllocatorImpl::new())
        // 创建 StackFrameAllocator 的实例，并创建它的引用 再 返回出去
    };
}

pub fn init_frame_allocator() {
    extern "C" {
        fn ekernel();
    }
    // 在正式分配物理页帧之前，我们需要将物理页帧全局管理器 FRAME_ALLOCATOR 初始化
    FRAME_ALLOCATOR
        .exclusive_access()
    // 拿到分配器的 可变借用
        .init(PhysAddr::from(ekernel as usize).ceil(), PhysAddr::from(MEMORY_END).floor());
    // 这里我们调用物理地址 PhysAddr 的 floor/ceil 方法分别下/上取整获得可用的物理页号区间
}

// 然后是公开给其他内核模块调用的分配/回收物理页帧的接口
pub fn frame_alloc() -> Option<FrameTracker> {
    FRAME_ALLOCATOR
        .exclusive_access()
        .alloc()
        .map(|ppn| FrameTracker::new(ppn))
    // 可以发现， frame_alloc 的返回值类型并不是 FrameAllocator 要求的物理页号 PhysPageNum ，
    // 而是将其进一步包装为一个 FrameTracker 。这里借用了 RAII 的思想，
    // 将一个物理页帧的生命周期绑定到一个 FrameTracker 变量上，
    // 当一个 FrameTracker 被创建的时候，我们需要从 FRAME_ALLOCATOR 中分配一个物理页帧
}

fn frame_dealloc(ppn: PhysPageNum) {
    FRAME_ALLOCATOR
        .exclusive_access()
        .dealloc(ppn);
}

#[allow(unused)]
pub fn frame_allocator_test() {
    let mut v: Vec<FrameTracker> = Vec::new();
    for i in 0..5 {
        let frame = frame_alloc().unwrap();
        println!("{:?}", frame);
        v.push(frame);
    }
    v.clear();
    for i in 0..5 {
        let frame = frame_alloc().unwrap();
        println!("{:?}", frame);
        v.push(frame);
    }
    drop(v);
    println!("frame_allocator_test passed!");
}
```



#### os/src/mm/heap_allocator.rs

```rust
use buddy_system_allocator::LockedHeap;
use crate::config::KERNEL_HEAP_SIZE;

#[global_allocator]
static HEAP_ALLOCATOR: LockedHeap = LockedHeap::empty();
// 我们直接将 buddy_system_allocator 中提供的 LockedHeap 实例化成一个全局变量，
// 并使用 alloc 要求的 #[global_allocator] 语义项进行标记。
// 注意 LockedHeap 已经实现了 GlobalAlloc 要求的抽象接口了
// // alloc::alloc::GlobalAlloc
// pub unsafe fn alloc(&self, layout: Layout) -> *mut u8;
// pub unsafe fn dealloc(&self, ptr: *mut u8, layout: Layout);
// 它们类似 C 语言中的 malloc/free ，分别代表堆空间的分配和回收，
// 也同样使用一个裸指针（也就是地址）作为分配的返回值和回收的参数。两个接口中都有一个alloc::alloc::Layout 类型的参数， 
// 它指出了分配的需求，分为两部分，分别是所需空间的大小 size ，以及返回地址的对齐要求 align 。
// 这个对齐要求必须是一个 2 的幂次，单位为字节数，限制返回的地址必须是 align 的倍数

#[alloc_error_handler]
pub fn handle_alloc_error(layout: core::alloc::Layout) -> ! {
    panic!("Heap allocation error, layout = {:?}", layout);
}
// 我们还需要处理动态内存分配失败的情形

static mut HEAP_SPACE: [u8; KERNEL_HEAP_SIZE] = [0; KERNEL_HEAP_SIZE];
// 这块内存是一个 static mut 且被零初始化的字节数组，位于内核的 .bss 段中

pub fn init_heap() {
    // 在使用任何 alloc 中提供的堆数据结构之前，我们需要先调用 init_heap 函数来给我们的全局分配器一块内存用于分配
    unsafe {
        HEAP_ALLOCATOR
            .lock()
            .init(HEAP_SPACE.as_ptr() as usize, KERNEL_HEAP_SIZE);
        // LockedHeap 也是一个被互斥锁 Mutex<T> 保护的类型，
        // 在对它任何进行任何操作之前都要先获取锁以避免其他线程同时对它进行操作导致数据竞争。
        // 然后，调用 init 方法告知它能够用来分配的空间的起始地址和大小即可。
    }
}

#[allow(unused)]
pub fn heap_test() {
    use alloc::boxed::Box;
    use alloc::vec::Vec;
    extern "C" {
        fn sbss();
        fn ebss();
    }
    let bss_range = sbss as usize..ebss as usize;
    let a = Box::new(5);
    assert_eq!(*a, 5);
    assert!(bss_range.contains(&(a.as_ref() as *const _ as usize)));
    drop(a);
    let mut v: Vec<usize> = Vec::new();
    for i in 0..500 {
        v.push(i);
    }
    for i in 0..500 {
        assert_eq!(v[i], i);
    }
    assert!(bss_range.contains(&(v.as_ptr() as usize)));
    drop(v);
    println!("heap_test passed!");
}
// 其中分别使用智能指针 Box<T> 和向量 Vec<T> 在堆上分配数据并管理它们，
// 通过 as_ref 和 as_ptr 方法可以分别看到它们指向的数据的位置，能够确认它们的确在位于 .bss 段的堆上
```



#### os/src/mm/memory_set.rs

```rust
// 一旦使能分页机制，CPU 访问到的地址都是虚拟地址了，那么内核中也将基于虚地址进行虚存访问。
// 所以在给应用添加虚拟地址空间前，内核自己也会建立一个页表，
// 把整块物理内存通过简单的恒等映射（即虚拟地址映射到对等的物理地址）映射到内核虚拟地址空间中。
// 后续的应用在执行前，也需要操作系统帮助它建立一个虚拟地址空间

// 在一个虚拟地址空间中，有代码段，数据段等不同属性且不一定连续的子空间，
// 它们通过一个重要的数据结构 MapArea 来表示和管理
use super::{PageTable, PageTableEntry, PTEFlags};
use super::{VirtPageNum, VirtAddr, PhysPageNum, PhysAddr};
use super::{FrameTracker, frame_alloc};
use super::{VPNRange, StepByOne};
use alloc::collections::BTreeMap;
use alloc::vec::Vec;
use riscv::register::satp;
use alloc::sync::Arc;
use lazy_static::*;
use crate::sync::UPSafeCell;
use crate::config::{
    MEMORY_END,
    PAGE_SIZE,
    TRAMPOLINE,
    TRAP_CONTEXT,
    USER_STACK_SIZE
};
use core::arch::asm;

extern "C" {
    fn stext();
    fn etext();
    fn srodata();
    fn erodata();
    fn sdata();
    fn edata();
    fn sbss_with_stack();
    fn ebss();
    fn ekernel();
    fn strampoline();
}

// 创建内核地址空间的全局实例
lazy_static! {
    pub static ref KERNEL_SPACE: Arc<UPSafeCell<MemorySet>> = Arc::new(unsafe {
        UPSafeCell::new(MemorySet::new_kernel()
    )});
    // 从之前对于 lazy_static! 宏的介绍可知， KERNEL_SPACE 在运行期间它第一次被用到时才会实际进行初始化，
    // 而它所 占据的空间则是编译期被放在全局数据段中。这里使用 Arc<UPSafeCell<T>> 组合是因为我们既需要 Arc<T> 提供的共享 引用，
    // 也需要 UPSafeCell<T> 提供的内部可变引用访问
    
    // 首先，我们引用 KERNEL_SPACE ，这是它第一次被使用，就在此时它会被初始化，
    // 调用 MemorySet::new_kernel 创建一个内核地址空间并使用 Arc<Mutex<T>> 包裹起来；
	// 接着使用 .exclusive_access() 获取一个可变引用 &mut MemorySet 。
    // 需要注意的是这里发生了两次隐式类型转换：
	// 我们知道 exclusive_access 是 UPSafeCell<T> 的方法而不是 Arc<T> 的方法，
    // 由于 Arc<T> 实现了 Deref Trait ，当 exclusive_access 需要一个 &UPSafeCell<T> 类型的参数的时候，
    // 编译器会自动将传入的 Arc<UPSafeCell<T>> 转换为 &UPSafeCell<T> 这样就实现了类型匹配；
	// 事实上 UPSafeCell<T>::exclusive_access 返回的是一个 RefMut<'_, T> ，
    // 这同样是 RAII 的思想，当这个类型生命周期结束后互斥锁就会被释放。
    // 而该类型实现了 DerefMut Trait，因此当一个函数接受类型为 &mut T 的参数却被传入一个类型为 &mut RefMut<'_, T> 的参数的时候，
    // 编译器会自动进行类型转换使参数匹配。
}

// 地址空间 是一系列有关联的不一定连续的逻辑段，
// 这种关联一般是指这些逻辑段组成的虚拟内存空间与一个运行的程序（目前把一个运行的程序称为任务，
// 后续会称为进程）绑定，即这个运行的程序对代码和数据的直接访问范围限制在它关联的虚拟地址空间之内。
// 这样我们就有任务的地址空间，内核的地址空间等说法了。地址空间使用 MemorySet 类型来表示
pub struct MemorySet {
    page_table: PageTable,
    // 该地址空间的多级页表 page_table
    // PageTable 下挂着所有多级页表的节点所在的物理页帧
    areas: Vec<MapArea>,
    // 一个逻辑段 MapArea 的向量 areas
    // 每个 MapArea 下则挂着对应逻辑段中的数据所在的物理页帧，这两部分合在一起构成了一个地址空间所需的所有物理页帧。
    // 这同样是一种 RAII 风格，当一个地址空间 MemorySet 生命周期结束后，这些物理页帧都会被回收
}

impl MemorySet {
    // 新建一个空的地址空间
    pub fn new_bare() -> Self {
        Self {
            page_table: PageTable::new(),
            areas: Vec::new(),
        }
    }
    
    pub fn token(&self) -> usize {
        self.page_table.token()
    }
    /// Assume that no conflicts.
    // 调用 push ，可以在当前地址空间插入一个 Framed 方式映射到物理内存的逻辑段。
    // 注意该方法的调用者要保证同一地址空间内的任意两个逻辑段不能存在交集，
    // 从后面即将分别介绍的内核和应用的地址空间布局可以看出这一要求得到了保证
    pub fn insert_framed_area(&mut self, start_va: VirtAddr, end_va: VirtAddr, permission: MapPermission) {
        self.push(MapArea::new(
            start_va,
            end_va,
            MapType::Framed,
            permission,
        ), None);
    }
    
    // push 方法可以在当前地址空间插入一个新的逻辑段 map_area ，如果它是以 Framed 方式映射到物理内存，
    // 还可以可选地在那些被映射到的物理页帧上写入一些初始化数据 data
    fn push(&mut self, mut map_area: MapArea, data: Option<&[u8]>) {
        map_area.map(&mut self.page_table);
        if let Some(data) = data {
            map_area.copy_data(&mut self.page_table, data);
        }
        self.areas.push(map_area);
    }
    /// Mention that trampoline is not collected by areas.
    fn map_trampoline(&mut self) {
        self.page_table.map(
            VirtAddr::from(TRAMPOLINE).into(),
            PhysAddr::from(strampoline as usize).into(),
            PTEFlags::R | PTEFlags::X,
        );
    }
    /// Without kernel stacks.
    // 生成内核的地址空间
    // new_kernel 将映射跳板和地址空间中最低  中的内核逻辑段。
    // 我们从 os/src/linker.ld 中引用了很多表示各个段位置的符号，而后在 new_kernel 中，
    // 我们从低地址到高地址依次创建 5 个逻辑段并通过 push 方法将它们插入到内核地址空间中，
    // 上面我们已经详细介绍过这 5 个逻辑段。跳板是通过 map_trampoline 方法来映射的
    pub fn new_kernel() -> Self {
        let mut memory_set = Self::new_bare();
        // map trampoline
        memory_set.map_trampoline();
        // map kernel sections
        println!(".text [{:#x}, {:#x})", stext as usize, etext as usize);
        println!(".rodata [{:#x}, {:#x})", srodata as usize, erodata as usize);
        println!(".data [{:#x}, {:#x})", sdata as usize, edata as usize);
        println!(".bss [{:#x}, {:#x})", sbss_with_stack as usize, ebss as usize);
        println!("mapping .text section");
        memory_set.push(MapArea::new(
            (stext as usize).into(),
            (etext as usize).into(),
            MapType::Identical,
            MapPermission::R | MapPermission::X,
        ), None);
        println!("mapping .rodata section");
        memory_set.push(MapArea::new(
            (srodata as usize).into(),
            (erodata as usize).into(),
            MapType::Identical,
            MapPermission::R,
        ), None);
        println!("mapping .data section");
        memory_set.push(MapArea::new(
            (sdata as usize).into(),
            (edata as usize).into(),
            MapType::Identical,
            MapPermission::R | MapPermission::W,
        ), None);
        println!("mapping .bss section");
        memory_set.push(MapArea::new(
            (sbss_with_stack as usize).into(),
            (ebss as usize).into(),
            MapType::Identical,
            MapPermission::R | MapPermission::W,
        ), None);
        println!("mapping physical memory");
        memory_set.push(MapArea::new(
            (ekernel as usize).into(),
            MEMORY_END.into(),
            MapType::Identical,
            MapPermission::R | MapPermission::W,
        ), None);
        memory_set
    }
    /// Include sections in elf and trampoline and TrapContext and user stack,
    /// also returns user_sp and entry point.
    // 应用的 ELF 格式可执行文件解析出各数据段并对应生成应用的地址空间
    // 在创建应用地址空间的时候，我们需要对 get_app_data 得到的 ELF 格式数据进行解析，
    // 找到各个逻辑段所在位置和访问限制并插入进来，最终得到一个完整的应用地址空间
    pub fn from_elf(elf_data: &[u8]) -> (Self, usize, usize) {
        let mut memory_set = Self::new_bare();
        // map trampoline
        memory_set.map_trampoline();
        // 我们将跳板插入到应用地址空间
        // map program headers of elf, with U flag
        let elf = xmas_elf::ElfFile::new(elf_data).unwrap();
        // 我们使用外部 crate xmas_elf 来解析传入的应用 ELF 数据并可以轻松取出各个部分
        // 我们取出 ELF 的魔数来判断它是不是一个合法的 ELF
        let elf_header = elf.header;
        let magic = elf_header.pt1.magic;
        assert_eq!(magic, [0x7f, 0x45, 0x4c, 0x46], "invalid elf!");
        let ph_count = elf_header.pt2.ph_count();
        // 我们可以直接得到 program header 的数目，
        let mut max_end_vpn = VirtPageNum(0);
        // 然后遍历所有的 program header 并将合适的区域加入到应用地址空间中
        for i in 0..ph_count {
            let ph = elf.program_header(i).unwrap();
            if ph.get_type().unwrap() == xmas_elf::program::Type::Load {
            // program header 的类型是 LOAD ，这表明它有被内核加载的必要，
            // 此时不必理会其他类型的 program header 
                let start_va: VirtAddr = (ph.virtual_addr() as usize).into();
                let end_va: VirtAddr = ((ph.virtual_addr() + ph.mem_size()) as usize).into();
                // 计算这一区域在应用地址空间中的位置
                let mut map_perm = MapPermission::U;
                let ph_flags = ph.flags();
                // 确认这一区域访问方式的限制并将其转换为 MapPermission 类型
                if ph_flags.is_read() { map_perm |= MapPermission::R; }
                if ph_flags.is_write() { map_perm |= MapPermission::W; }
                if ph_flags.is_execute() { map_perm |= MapPermission::X; }
                let map_area = MapArea::new(
                // 创建逻辑段 map_area
                    start_va,
                    end_va,
                    MapType::Framed,
                    map_perm,
                );
                max_end_vpn = map_area.vpn_range.get_end();
                memory_set.push(
                // push 到应用地址空间
                    map_area,
                    Some(&elf.input[ph.offset() as usize..(ph.offset() + ph.file_size()) as usize])
                );
                // 在 push 的时候我们需要完成数据拷贝，当前 program header 数据被存放的位置可以通过 ph.offset() 和 ph.file_size() 来找到。 
                // 注意当存在一部分零初始化的时候， ph.file_size() 将会小于 ph.mem_size() ，
                // 因为这些零出于缩减可执行文件大小的原因不应该实际出现在 ELF 数据中
            }
        }
        // map user stack with U flags
        // 处理用户栈。注意在前面加载各个 program header 的时候，
        // 我们就已经维护了 max_end_vpn 记录目前涉及到的最大的虚拟页号，
        // 只需紧接着在它上面再放置一个保护页面和用户栈即可
        let max_end_va: VirtAddr = max_end_vpn.into();
        let mut user_stack_bottom: usize = max_end_va.into();
        // guard page
        user_stack_bottom += PAGE_SIZE;
        let user_stack_top = user_stack_bottom + USER_STACK_SIZE;
        memory_set.push(MapArea::new(
            user_stack_bottom.into(),
            user_stack_top.into(),
            MapType::Framed,
            MapPermission::R | MapPermission::W | MapPermission::U,
        ), None);
        // map TrapContext
        // 应用地址空间中映射次高页面来存放 Trap 上下文
        memory_set.push(MapArea::new(
            TRAP_CONTEXT.into(),
            TRAMPOLINE.into(),
            MapType::Framed,
            MapPermission::R | MapPermission::W,
        ), None);
        (memory_set, user_stack_top, elf.header.pt2.entry_point() as usize)
        // 返回应用地址空间 memory_set ，
        // 也同时返回用户栈虚拟地址 user_stack_top 以及从解析 ELF 得到的该应用入口点地址，
        // 它们将被我们用来创建应用的任务控制块
    }
    pub fn activate(&self) {
        let satp = self.page_table.token();
        unsafe {
            satp::write(satp);
            asm!("sfence.vma");
        }
        // 我们将这个值写入当前 CPU 的 satp CSR ，从这一刻开始 SV39 分页模式就被启用了，
        // 而且 MMU 会使用内核地址空间的多级页表进行地址转换
    }
    pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
        self.page_table.translate(vpn)
    }
}

// 我们以逻辑段 MapArea 为单位描述一段连续地址的虚拟内存。所谓逻辑段，
// 就是指地址区间中的一段实际可用（即 MMU 通过查多级页表可以正确完成地址转换）的地址连续的虚拟地址区间，
// 该区间内包含的所有虚拟页面都以一种相同的方式映射到物理页帧，具有可读/可写/可执行等属性
pub struct MapArea {
    vpn_range: VPNRange,
    // 其中 VPNRange 描述一段虚拟页号的连续区间，表示该逻辑段在地址区间中的位置和长度。
    // 它是一个迭代器，可以使用 Rust 的语法糖 for-loop 进行迭代
    data_frames: BTreeMap<VirtPageNum, FrameTracker>,
    map_type: MapType,
    map_perm: MapPermission,
    // MapPermission 表示控制该逻辑段的访问方式，它是页表项标志位 PTEFlags 的一个子集，
    // 仅保留 U/R/W/X 四个标志位，因为其他的标志位仅与硬件的地址转换机制细节相关，这样的设计能避免引入错误的标志位
}

impl MapArea {
    // 新建一个逻辑段结构体，
    // 注意传入的起始/终止虚拟地址会分别被下取整/上取整为虚拟页号并传入迭代器 vpn_range 中
    pub fn new(
        start_va: VirtAddr,
        end_va: VirtAddr,
        map_type: MapType,
        map_perm: MapPermission
    ) -> Self {
        let start_vpn: VirtPageNum = start_va.floor();
        let end_vpn: VirtPageNum = end_va.ceil();
        Self {
            vpn_range: VPNRange::new(start_vpn, end_vpn),
            data_frames: BTreeMap::new(),
            map_type,
            map_perm,
        }
    }
    
    // 对逻辑段中的单个虚拟页面进行映射/解映射的方法
    // 在虚拟页号 vpn 已经确定的情况下，它需要知道要将一个怎么样的页表项插入多级页表。
    // 页表项的标志位来源于当前逻辑段的类型为 MapPermission 的统一配置，只需将其转换为 PTEFlags ；
    // 而页表项的物理页号则取决于当前逻辑段映射到物理内存的方式:
    	// 当以恒等映射 Identical 方式映射的时候，物理页号就等于虚拟页号；
		// 当以 Framed 方式映射时，需要分配一个物理页帧让当前的虚拟页面可以映射过去，
    	// 此时页表项中的物理页号自然就是 这个被分配的物理页帧的物理页号。
   	 	// 此时还需要将这个物理页帧挂在逻辑段的 data_frames 字段下。
	// 当确定了页表项的标志位和物理页号之后，即可调用多级页表 PageTable 的 map 接口来插入键值对
    pub fn map_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
        let ppn: PhysPageNum;
        match self.map_type {
            MapType::Identical => {
                ppn = PhysPageNum(vpn.0);
            }
            MapType::Framed => {
                let frame = frame_alloc().unwrap();
                ppn = frame.ppn;
                self.data_frames.insert(vpn, frame);
            }
        }
        let pte_flags = PTEFlags::from_bits(self.map_perm.bits).unwrap();
        page_table.map(vpn, ppn, pte_flags);
    }
    
    // 基本上就是调用 PageTable 的 unmap 接口删除以传入的虚拟页号为键的键值对即可。
    // 然而，当以 Framed 映射的时候，不要忘记同时将虚拟页面被映射到的物理页帧 FrameTracker 从 data_frames 中移除，
    // 这样这个物理页帧才能立即被回收以备后续分配
    #[allow(unused)]
    pub fn unmap_one(&mut self, page_table: &mut PageTable, vpn: VirtPageNum) {
        match self.map_type {
            MapType::Framed => {
                self.data_frames.remove(&vpn);
            }
            _ => {}
        }
        page_table.unmap(vpn);
    }
    
    // 可以将当前逻辑段到物理内存的映射从传入的该逻辑段所属的地址空间的多级页表中加入或删除。
    // 可以看到它们的实现是遍历逻辑段中的所有虚拟页面，
    // 并以每个虚拟页面为单位依次在多级页表中进行键值对的插入或删除，
    // 分别对应 MapArea 的 map_one 和 unmap_one 方法
    pub fn map(&mut self, page_table: &mut PageTable) {
        for vpn in self.vpn_range {
            self.map_one(page_table, vpn);
        }
    }
    #[allow(unused)]
    pub fn unmap(&mut self, page_table: &mut PageTable) {
        for vpn in self.vpn_range {
            self.unmap_one(page_table, vpn);
        }
    }
    /// data: start-aligned but maybe with shorter length
    /// assume that all frames were cleared before
    // 将切片 data 中的数据拷贝到当前逻辑段实际被内核放置在的各物理页帧上，
    // 从而在地址空间中通过该逻辑段就能访问这些数据。调用它的时候需要满足：
    // 切片 data 中的数据大小不超过当前逻辑段的总大小，且切片中的数据会被对齐到逻辑段的开头，
    // 然后逐页拷贝到实际的物理页帧
    pub fn copy_data(&mut self, page_table: &mut PageTable, data: &[u8]) {
        assert_eq!(self.map_type, MapType::Framed);
        let mut start: usize = 0;
        let mut current_vpn = self.vpn_range.get_start();
        let len = data.len();
        loop {
            let src = &data[start..len.min(start + PAGE_SIZE)];
            let dst = &mut page_table
                .translate(current_vpn)
                .unwrap()
                .ppn()
                .get_bytes_array()[..src.len()];
            dst.copy_from_slice(src);
            start += PAGE_SIZE;
            if start >= len {
                break;
            }
            current_vpn.step();
        }
        // 循环会遍历每一个需要拷贝数据的虚拟页面，在数据拷贝完成后会通过调用 step 方法
    }
}

// MapType 描述该逻辑段内的所有虚拟页面映射到物理页帧的同一种方式，它是一个枚举类型，
// 在内核当前的实现中支持两种方式
#[derive(Copy, Clone, PartialEq, Debug)]
pub enum MapType {
    Identical,
    // Identical 表示恒等映射方式
    // 恒等映射方式主要是用在启用多级页表之后，
    // 内核仍能够在虚存地址空间中访问一个特定的物理地址指向的物理内存
    Framed,
    // Framed 则表示对于每个虚拟页面都有一个新分配的物理页帧与之对应，
    // 虚地址与物理地址的映射关系是相对随机的
    // 当逻辑段采用 MapType::Framed 方式映射到物理内存的时候， 
    // data_frames 是一个保存了该逻辑段内的每个虚拟页面和它被映射到的物理页帧 FrameTracker 的一个键值对容器 BTreeMap 中，
    // 这些物理页帧被用来存放实际内存数据而不是作为多级页表中的中间节点。和之前的 PageTable 一样，
    // 这也用到了 RAII 的思想，将这些物理页帧的生命周期绑定到它所在的逻辑段 MapArea 下，
    // 当逻辑段被回收之后这些之前分配的物理页帧也会自动地同时被回收
}


bitflags! {
    pub struct MapPermission: u8 {
        const R = 1 << 1;
        const W = 1 << 2;
        const X = 1 << 3;
        const U = 1 << 4;
    }
    // 仅保留 U/R/W/X 四个标志位，因为其他的标志位仅与硬件的地址转换机制细节相关，
    // 这样的设计能避免引入错误的标志位
}


// 检查内核地址空间的多级页表是否被正确设置
#[allow(unused)]
pub fn remap_test() {
    let mut kernel_space = KERNEL_SPACE.exclusive_access();
    let mid_text: VirtAddr = ((stext as usize + etext as usize) / 2).into();
    let mid_rodata: VirtAddr = ((srodata as usize + erodata as usize) / 2).into();
    let mid_data: VirtAddr = ((sdata as usize + edata as usize) / 2).into();
    assert_eq!(
        kernel_space.page_table.translate(mid_text.floor()).unwrap().writable(),
        false
    );
    assert_eq!(
        kernel_space.page_table.translate(mid_rodata.floor()).unwrap().writable(),
        false,
    );
    assert_eq!(
        kernel_space.page_table.translate(mid_data.floor()).unwrap().executable(),
        false,
    );
    println!("remap_test passed!");
}

```



#### os/src/mm/mod.rs

```rust
mod heap_allocator;
mod address;
mod frame_allocator;
mod page_table;
mod memory_set;

use page_table::{PageTable, PTEFlags};
use address::{VPNRange, StepByOne};
pub use address::{PhysAddr, VirtAddr, PhysPageNum, VirtPageNum};
pub use frame_allocator::{FrameTracker, frame_alloc};
pub use page_table::{PageTableEntry, translated_byte_buffer};
pub use memory_set::{MemorySet, KERNEL_SPACE, MapPermission};
pub use memory_set::remap_test;

pub fn init() {
    heap_allocator::init_heap();
    // 我们最先进行了全局动态内存分配器的初始化
    // 因为接下来马上就要用到 Rust 的堆数据结构
    frame_allocator::init_frame_allocator();
    // 接下来我们初始化物理页帧管理器（内含堆数据结构 Vec<T> ）使能可用物理页帧的分配和回收能力
    KERNEL_SPACE.exclusive_access().activate();
    // 最后我们创建内核地址空间并让 CPU 开启分页模式， MMU 在地址转换的时候使用内核的多级页表
}

```



#### os/src/mm/page_table.rs

```rust
// 页表中的页表项的索引其实是虚拟地址中的虚拟页号，页表项的重要内容是物理地址的物理页帧号。
// 为了能够灵活地在虚拟地址、物理地址、虚拟页号、物理页号之间进行各种转换

// 查页表以访问不同地址空间的数据 。在内核地址空间中执行的内核代码常常需要读写应用的地址空间中的数据，
// 这无法简单的通过一次访存来解决，而是需要手动查用户态应用的地址空间的页表，
// 知道用户态应用的虚地址对应的物理地址后，转换成对应的内核态的虚地址，才能访问应用地址空间中的数据。
// 如果访问应用地址空间中的数据跨了多个页，还需要注意处理地址的边界条件
use super::{frame_alloc, PhysPageNum, FrameTracker, VirtPageNum, VirtAddr, StepByOne};
use alloc::vec::Vec;
use alloc::vec;
use bitflags::*;

// bitflags 是一个 Rust 中常用来比特标志位的 crate 。它提供了一个 bitflags! 宏，
// 如上面的代码段所展示的那样，可以将一个 u8 封装成一个标志位的集合类型，支持一些常见的集合运算
bitflags! {
    pub struct PTEFlags: u8 {
        const V = 1 << 0;
        const R = 1 << 1; 
        const W = 1 << 2;
        const X = 1 << 3;
        const U = 1 << 4;
        const G = 1 << 5;
        const A = 1 << 6;
        const D = 1 << 7;
    }
}

// 实现页表项 PageTableEntry
// 64位
// |Reserved PPN[2] PPN[1] PPN[0] RSW D A G U X W R V|

#[derive(Copy, Clone)]
// 让编译器自动为 PageTableEntry 实现 Copy/Clone Trait，来让这个类型以值语义赋值/传参的时候不会发生所有权转移，
// 而是拷贝一份新的副本。从这一点来说 PageTableEntry 就和 usize 一样，
// 因为它也只是后者的一层简单包装，并解释了 usize 各个比特段的含义
#[repr(C)]
// 页表项 63-54 (Reserved)	53-10 (44位物理页号)	9-8 (RSW) 7-0 (8位标志位)
pub struct PageTableEntry {
    pub bits: usize,
}

impl PageTableEntry {
    pub fn new(ppn: PhysPageNum, flags: PTEFlags) -> Self {
        PageTableEntry {
            bits: ppn.0 << 10 | flags.bits as usize,
        }
    }
    // 我们可以从一个物理页号 PhysPageNum 和一个页表项标志位 PTEFlags 生成一个页表项 PageTableEntry 实例
    pub fn empty() -> Self {
        PageTableEntry {
            bits: 0,
        }
    }
    // 我们也可以通过 empty 方法生成一个全零的页表项，注意这隐含着该页表项的 V 标志位为 0 ，
    // 因此它是不合法的
    pub fn ppn(&self) -> PhysPageNum {
        (self.bits >> 10 & ((1usize << 44) - 1)).into()
                           // 和 44 个 1 相与，相当于去掉reserved 部分
    }
    // 从一个页表项将 ppn 取出
    pub fn flags(&self) -> PTEFlags {
        PTEFlags::from_bits(self.bits as u8).unwrap()
    }
    // 从一个页表项将 flags 8个标志位取出
    pub fn is_valid(&self) -> bool {
        (self.flags() & PTEFlags::V) != PTEFlags::empty()
    }
    pub fn readable(&self) -> bool {
        (self.flags() & PTEFlags::R) != PTEFlags::empty()
    }
    pub fn writable(&self) -> bool {
        (self.flags() & PTEFlags::W) != PTEFlags::empty()
    }
    pub fn executable(&self) -> bool {
        (self.flags() & PTEFlags::X) != PTEFlags::empty()
    }
    // 一些辅助函数(Helper Function)，可以快速判断一个页表项的 V/R/W/X 标志位是否为 1 
}

//          页表
pub struct PageTable {
    root_ppn: PhysPageNum,
    frames: Vec<FrameTracker>,
    // 每个应用的地址空间都对应一个不同的多级页表，
    // 这也就意味这不同页表的起始地址（即页表根节点的地址）是不一样的。
    // 因此 PageTable``要保存它根节点的物理页号 ``root_ppn 作为页表唯一的区分标志。
    // 此外，向量 frames 以 FrameTracker 的形式保存了页表所有的节点（包括根节点）所在的物理页帧。
    // 这与物理页帧管理模块的测试程序是一个思路，即将这些 FrameTracker 的生命周期进一步绑定到 PageTable 下面。
    // 当 PageTable 生命周期结束后，向量 frames 里面的那些 FrameTracker 也会被回收，
    // 也就意味着存放多级页表节点的那些物理页帧被回收了
}

/// Assume that it won't oom when creating/mapping.
impl PageTable {
    // 当我们通过 new 方法新建一个 PageTable 的时候，它只需有一个根节点。
    // 为此我们需要分配一个物理页帧 FrameTracker 并挂在向量 frames 下，然后更新根节点的物理页号 root_ppn
    pub fn new() -> Self {
        let frame = frame_alloc().unwrap();
        PageTable {
            root_ppn: frame.ppn,
            frames: vec![frame],
        }
    }
    /// Temporarily used to get arguments from user space.
    pub fn from_token(satp: usize) -> Self {
        Self {
            root_ppn: PhysPageNum::from(satp & ((1usize << 44) - 1)),
            frames: Vec::new(),
        }
    }
    // // from_token 可以临时创建一个专用来手动查页表的 PageTable ，
    // 它仅有一个从传入的 satp token 中得到的多级页表根节点的物理页号，它的 frames 字段为空，
    // 也即不实际控制任何资源
    
    // 当遇到需要查一个特定页表（非当前正处在的地址空间的页表时），
    // 便可先通过 PageTable::from_token 新建一个页表，再调用它的 translate 方法查页表
    
    // 在多级页表找到一个虚拟页号对应的页表项的可变引用。
    // 如果在遍历的过程中发现有节点尚未创建则会新建一个节点
    fn find_pte_create(&mut self, vpn: VirtPageNum) -> Option<&mut PageTableEntry> {
        let idxs = vpn.indexes();
        // 取出三级索引
        let mut ppn = self.root_ppn;
        let mut result: Option<&mut PageTableEntry> = None;
        for i in 0..3 {
            let pte = &mut ppn.get_pte_array()[idxs[i]];
           // 分别取出 第三级索引 第二级索引 第一级索引(即物理页号)
            if i == 2 {
                // 找到 物理页号
                result = Some(pte);
                // 返回物理页号
                break;
            }
            if !pte.is_valid() {
                let frame = frame_alloc().unwrap();
                // create 创建一个物理页帧，返回他的物理页号
                *pte = PageTableEntry::new(frame.ppn, PTEFlags::V);
                // 生成这个物理页号对应的物理页表项
                self.frames.push(frame);
                // 将这个物理页帧存放在 全局管理器的 Vector中
            }
            ppn = pte.ppn();
        }
        // 变量 ppn 表示当前节点的物理页号，最开始指向多级页表的根节点。
        // 随后每次循环通过 get_pte_array 将取出当前节点的页表项数组，
        // 并根据当前级页索引找到对应的页表项。如果当前节点是一个叶节点，
        // 那么直接返回这个页表项的可变引用；否则尝试向下走。走不下去的话就新建一个节点，
        // 更新作为下级节点指针的页表项，并将新分配的物理页帧移动到向量 frames 中方便后续的自动回收。
        // 注意在更新页表项的时候，不仅要更新物理页号，还要将标志位 V 置 1，
        // 不然硬件在查多级页表的时候，会认为这个页表项不合法，从而触发 Page Fault 而不能向下走
        result
    }
    fn find_pte(&self, vpn: VirtPageNum) -> Option<&PageTableEntry> {
        let idxs = vpn.indexes();
        let mut ppn = self.root_ppn;
        let mut result: Option<&PageTableEntry> = None;
        for i in 0..3 {
            let pte = &ppn.get_pte_array()[idxs[i]];
            if i == 2 {
                result = Some(pte);
                break;
            }
            if !pte.is_valid() {
                return None;
            }
            ppn = pte.ppn();
        }
        // PageTable::find_pte 与 find_pte_create 的不同在于当找不到合法叶子节点的时候不会新建叶子节点而是直接返回 None 即查找失败。
        // 因此，它不会尝试对页表本身进行修改，但是注意它返回的参数类型是页表项的可变引用，
        // 也即它允许我们修改页表项。从 find_pte 的实现还可以看出，即使找到的页表项不合法，
        // 还是会将其返回回去而不是返回 None 。这说明在目前的实现中，页表和页表项是相对解耦合的
        result
    }
    
    // 多级页表并不是被创建出来之后就不再变化的，
    // 为了 MMU 能够通过地址转换正确找到应用地址空间中的数据实际被内核放在内存中位置，
    // 操作系统需要动态维护一个虚拟页号到页表项的映射，支持插入/删除键值对
    
    // 接下来介绍建立和拆除虚实地址映射关系的 map 和 unmap 方法是如何实现的。
    // 它们都依赖于一个很重要的过程，即在多级页表中找到一个虚拟地址对应的页表项。找到之后，
    // 只要修改页表项的内容即可完成键值对的插入和删除。在寻找页表项的时候，
    // 可能出现页表的中间级节点还未被创建的情况，这个时候我们需要手动分配一个物理页帧来存放这个节点，
    // 并将这个节点接入到当前的多级页表的某级中
    
    // 只需根据虚拟页号找到页表项，然后修改或者直接清空其内容即可
    #[allow(unused)]
    pub fn map(&mut self, vpn: VirtPageNum, ppn: PhysPageNum, flags: PTEFlags) {
        let pte = self.find_pte_create(vpn).unwrap();
        // 查找一个虚拟页号
        assert!(!pte.is_valid(), "vpn {:?} is mapped before mapping", vpn);
        *pte = PageTableEntry::new(ppn, flags | PTEFlags::V);
        // 创建一个页表项， 存放这个虚拟页号 所映射的 物理页号
    }
    // 通过 map 方法来在多级页表中插入一个键值对，
    // 注意这里将物理页号 ppn 和页表项标志位 flags 作为不同的参数传入
    #[allow(unused)]
    pub fn unmap(&mut self, vpn: VirtPageNum) {
        let pte = self.find_pte_create(vpn).unwrap();
        //  创建一个虚拟页表项
        assert!(pte.is_valid(), "vpn {:?} is invalid before unmapping", vpn);
        *pte = PageTableEntry::empty();
        // 将页表项的所有位 置0
    }
    // 通过 unmap 方法来删除一个键值对，在调用时仅需给出作为索引的虚拟页号即可
    pub fn translate(&self, vpn: VirtPageNum) -> Option<PageTableEntry> {
        self.find_pte(vpn)
            .map(|pte| {pte.clone()})
    }
    //  调用 find_pte 来实现，如果能够找到页表项，那么它会将页表项拷贝一份并返回，否则就返回一个 None
    
    pub fn token(&self) -> usize {
        8usize << 60 | self.root_ppn.0
    }
    // PageTable::token 会按照 satp CSR 格式要求 构造一个无符号 64 位无符号整数，
    // 使得其分页模式为 SV39 ，且将当前多级页表的根节点所在的物理页号填充进去
}

pub fn translated_byte_buffer(token: usize, ptr: *const u8, len: usize) -> Vec<&'static mut [u8]> {
    let page_table = PageTable::from_token(token);
    let mut start = ptr as usize;
    let end = start + len;
    let mut v = Vec::new();
    while start < end {
        let start_va = VirtAddr::from(start);
        let mut vpn = start_va.floor();
        let ppn = page_table
            .translate(vpn)
            .unwrap()
            .ppn();
        vpn.step();
        let mut end_va: VirtAddr = vpn.into();
        end_va = end_va.min(VirtAddr::from(end));
        if end_va.page_offset() == 0 {
            v.push(&mut ppn.get_bytes_array()[start_va.page_offset()..]);
        } else {
            v.push(&mut ppn.get_bytes_array()[start_va.page_offset()..end_va.page_offset()]);
        }
        start = end_va.into();
    }
    v
}
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
// 查页表以访问不同地址空间的数据 。在内核地址空间中执行的内核代码常常需要读写应用的地址空间中的数据，
// 这无法简单的通过一次访存来解决，而是需要手动查用户态应用的地址空间的页表，
// 知道用户态应用的虚地址对应的物理地址后，转换成对应的内核态的虚地址，才能访问应用地址空间中的数据。
// 如果访问应用地址空间中的数据跨了多个页，还需要注意处理地址的边界条件

use crate::mm::translated_byte_buffer;
use crate::task::current_user_token;

const FD_STDOUT: usize = 1;

pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize {
    match fd {
        FD_STDOUT => {
            let buffers = translated_byte_buffer(current_user_token(), buf, len);
            for buffer in buffers {
                print!("{}", core::str::from_utf8(buffer).unwrap());
            }
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

pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize {
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
use crate::timer::get_time_ms;

pub fn sys_exit(exit_code: i32) -> ! {
    println!("[kernel] Application exited with code {}", exit_code);
    exit_current_and_run_next();
    panic!("Unreachable in sys_exit!");
}

pub fn sys_yield() -> isize {
    suspend_current_and_run_next();
    0
}

pub fn sys_get_time() -> isize {
    get_time_ms() as isize
}
```

#### os/src/task/context.rs

```rust
use crate::trap::trap_return;

#[repr(C)]
pub struct TaskContext {
    ra: usize,
    sp: usize,
    s: [usize; 12],
}

impl TaskContext {
    pub fn zero_init() -> Self {
        Self {
            ra: 0,
            sp: 0,
            s: [0; 12],
        }
    }
    pub fn goto_trap_return(kstack_ptr: usize) -> Self {
        Self {
            ra: trap_return as usize,
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
    # save kernel stack of current task
    sd sp, 8(a0)
    # save ra & s0~s11 of current execution
    sd ra, 0(a0)
    .set n, 0
    .rept 12
        SAVE_SN %n
        .set n, n + 1
    .endr
    # restore ra & s0~s11 of next execution
    ld ra, 0(a1)
    .set n, 0
    .rept 12
        LOAD_SN %n
        .set n, n + 1
    .endr
    # restore kernel stack of next task
    ld sp, 8(a1)
    ret


```

#### os/src/task.rs

```rust
// 操作系统需要扩展任务控制块 TaskControlBlock 的管理范围，
// 使得操作系统能管理拥有独立页表和单一虚拟地址空间的应用程序的运行
use crate::mm::{MemorySet, MapPermission, PhysPageNum, KERNEL_SPACE, VirtAddr};
use crate::trap::{TrapContext, trap_handler};
use crate::config::{TRAP_CONTEXT, kernel_stack_position};
use super::TaskContext;

pub struct TaskControlBlock {
    pub task_status: TaskStatus,
    pub task_cx: TaskContext,
    pub memory_set: MemorySet,
    pub trap_cx_ppn: PhysPageNum,
    pub base_size: usize,
}

impl TaskControlBlock {
    pub fn get_trap_cx(&self) -> &'static mut TrapContext {
        self.trap_cx_ppn.get_mut()
    }
    pub fn get_user_token(&self) -> usize {
        self.memory_set.token()
    }
    pub fn new(elf_data: &[u8], app_id: usize) -> Self {
        // memory_set with elf program headers/trampoline/trap context/user stack
        let (memory_set, user_sp, entry_point) = MemorySet::from_elf(elf_data);
        // 解析传入的 ELF 格式数据构造应用的地址空间 memory_set 并获得其他信息
        let trap_cx_ppn = memory_set
            .translate(VirtAddr::from(TRAP_CONTEXT).into())
            .unwrap()
            .ppn();
        // 从地址空间 memory_set 中查多级页表找到应用地址空间中的 Trap 上下文实际被放在哪个物理页帧
        let task_status = TaskStatus::Ready;
        // map a kernel-stack in kernel space
        let (kernel_stack_bottom, kernel_stack_top) = kernel_stack_position(app_id);
        KERNEL_SPACE
            .exclusive_access()
            .insert_framed_area(
                kernel_stack_bottom.into(),
                kernel_stack_top.into(),
                MapPermission::R | MapPermission::W,
            );
        // 根据传入的应用 ID app_id 调用在 config 子模块中定义的 kernel_stack_position 找到 应用的内核栈预计放在内核地址空间 KERNEL_SPACE 中的哪个位置
        // 并通过 insert_framed_area 实际将这个逻辑段 加入到内核地址空间中
        let task_control_block = Self {
            task_status,
            task_cx: TaskContext::goto_trap_return(kernel_stack_top),
            // 在应用的内核栈顶压入一个跳转到 trap_return 而不是 __restore 的任务上下文，
        // 这主要是为了能够支持对该应用的启动并顺利切换到用户地址空间执行。
        // 在构造方式上，只是将 ra 寄存器的值设置为 trap_return 的地址。 
        // trap_return 是后面要介绍的新版的 Trap 处理的一部分
            memory_set,
            trap_cx_ppn,
            base_size: user_sp,
            // 用上面的信息来创建并返回任务控制块实例 task_control_block
        };
        
        // prepare TrapContext in user space
        let trap_cx = task_control_block.get_trap_cx();
        *trap_cx = TrapContext::app_init_context(
            entry_point,
            user_sp,
            KERNEL_SPACE.exclusive_access().token(),
            kernel_stack_top,
            trap_handler as usize,
        );
        task_control_block
    }
}

#[derive(Copy, Clone, PartialEq)]
pub enum TaskStatus {
    Ready,
    Running,
    Exited,
}
```

#### os/src/task/mod.rs

``` rust
mod context;
mod switch;
mod task;

use crate::loader::{get_num_app, get_app_data};
use crate::trap::TrapContext;
use crate::sync::UPSafeCell;
use lazy_static::*;
use switch::__switch;
use task::{TaskControlBlock, TaskStatus};
use alloc::vec::Vec;

pub use context::TaskContext;

pub struct TaskManager {
    num_app: usize,
    inner: UPSafeCell<TaskManagerInner>,
}

struct TaskManagerInner {
    tasks: Vec<TaskControlBlock>,
    current_task: usize,
}

lazy_static! {
    pub static ref TASK_MANAGER: TaskManager = {
        println!("init TASK_MANAGER");
        let num_app = get_num_app();
        println!("num_app = {}", num_app);
        let mut tasks: Vec<TaskControlBlock> = Vec::new();
        for i in 0..num_app {
            tasks.push(TaskControlBlock::new(
                get_app_data(i),
                i,
            ));
        }
        TaskManager {
            num_app,
            inner: unsafe { UPSafeCell::new(TaskManagerInner {
                tasks,
                current_task: 0,
            })},
        }
    };
}

impl TaskManager {
    fn run_first_task(&self) -> ! {
        let mut inner = self.inner.exclusive_access();
        let next_task = &mut inner.tasks[0];
        next_task.task_status = TaskStatus::Running;
        let next_task_cx_ptr = &next_task.task_cx as *const TaskContext;
        drop(inner);
        let mut _unused = TaskContext::zero_init();
        // before this, we should drop local variables that must be dropped manually
        unsafe {
            __switch(
                &mut _unused as *mut _,
                next_task_cx_ptr,
            );
        }
        panic!("unreachable in run_first_task!");
    }

    fn mark_current_suspended(&self) {
        let mut inner = self.inner.exclusive_access();
        let cur = inner.current_task;
        inner.tasks[cur].task_status = TaskStatus::Ready;
    }

    fn mark_current_exited(&self) {
        let mut inner = self.inner.exclusive_access();
        let cur = inner.current_task;
        inner.tasks[cur].task_status = TaskStatus::Exited;
    }

    fn find_next_task(&self) -> Option<usize> {
        let inner = self.inner.exclusive_access();
        let current = inner.current_task;
        (current + 1..current + self.num_app + 1)
            .map(|id| id % self.num_app)
            .find(|id| {
                inner.tasks[*id].task_status == TaskStatus::Ready
            })
    }

    fn get_current_token(&self) -> usize {
        let inner = self.inner.exclusive_access();
        inner.tasks[inner.current_task].get_user_token()
    }

    fn get_current_trap_cx(&self) -> &mut TrapContext {
        let inner = self.inner.exclusive_access();
        inner.tasks[inner.current_task].get_trap_cx()
    }

    fn run_next_task(&self) {
        if let Some(next) = self.find_next_task() {
            let mut inner = self.inner.exclusive_access();
            let current = inner.current_task;
            inner.tasks[next].task_status = TaskStatus::Running;
            inner.current_task = next;
            let current_task_cx_ptr = &mut inner.tasks[current].task_cx as *mut TaskContext;
            let next_task_cx_ptr = &inner.tasks[next].task_cx as *const TaskContext;
            drop(inner);
            // before this, we should drop local variables that must be dropped manually
            unsafe {
                __switch(
                    current_task_cx_ptr,
                    next_task_cx_ptr,
                );
            }
            // go back to user mode
        } else {
            panic!("All applications completed!");
        }
    }
}

pub fn run_first_task() {
    TASK_MANAGER.run_first_task();
}

fn run_next_task() {
    TASK_MANAGER.run_next_task();
}

fn mark_current_suspended() {
    TASK_MANAGER.mark_current_suspended();
}

fn mark_current_exited() {
    TASK_MANAGER.mark_current_exited();
}

pub fn suspend_current_and_run_next() {
    mark_current_suspended();
    run_next_task();
}

pub fn exit_current_and_run_next() {
    mark_current_exited();
    run_next_task();
}

pub fn current_user_token() -> usize {
    TASK_MANAGER.get_current_token()
}

pub fn current_trap_cx() -> &'static mut TrapContext {
    TASK_MANAGER.get_current_trap_cx()
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
    pub kernel_satp: usize,
    // 表示内核地址空间的 token ，即内核页表的起始物理地址
    pub kernel_sp: usize,
    // 表示当前应用在内核地址空间中的内核栈栈顶的虚拟地址
    pub trap_handler: usize,
    // 表示内核中 trap handler 入口点的虚拟地址
}

impl TrapContext {
    pub fn set_sp(&mut self, sp: usize) { self.x[2] = sp; }
    pub fn app_init_context(
        entry: usize,
        sp: usize,
        kernel_satp: usize,
        kernel_sp: usize,
        trap_handler: usize,
    ) -> Self {
        let mut sstatus = sstatus::read();
        sstatus.set_spp(SPP::User);
        let mut cx = Self {
            x: [0; 32],
            sstatus,
            sepc: entry,
            kernel_satp,
            kernel_sp,
            trap_handler,
        };
        cx.set_sp(sp);
        cx
    }
}

```

#### os/src/trap/mod.rs

```rust
// 还有就是需要对来自用户态和内核态的异常/中断分别进行处理

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
use crate::task::{
    exit_current_and_run_next,
    suspend_current_and_run_next,
    current_user_token,
    current_trap_cx,
};
use crate::timer::set_next_trigger;
use crate::config::{TRAP_CONTEXT, TRAMPOLINE};
use core::arch::{global_asm, asm};

global_asm!(include_str!("trap.S"));

pub fn init() {
    set_kernel_trap_entry();
}

fn set_kernel_trap_entry() {
    unsafe {
        stvec::write(trap_from_kernel as usize, TrapMode::Direct);
    }
}

fn set_user_trap_entry() {
    unsafe {
        stvec::write(TRAMPOLINE as usize, TrapMode::Direct);
    }
}

pub fn enable_timer_interrupt() {
    unsafe { sie::set_stimer(); }
}

#[no_mangle]
pub fn trap_handler() -> ! {
    set_kernel_trap_entry();
    let cx = current_trap_cx();
    let scause = scause::read();
    let stval = stval::read();
    match scause.cause() {
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
        _ => {
            panic!("Unsupported trap {:?}, stval = {:#x}!", scause.cause(), stval);
        }
    }
    trap_return();
}

#[no_mangle]
pub fn trap_return() -> ! {
    set_user_trap_entry();
    let trap_cx_ptr = TRAP_CONTEXT;
    let user_satp = current_user_token();
    extern "C" {
        fn __alltraps();
        fn __restore();
    }
    let restore_va = __restore as usize - __alltraps as usize + TRAMPOLINE;
    unsafe {
        asm!(
            "fence.i",
            "jr {restore_va}",
            restore_va = in(reg) restore_va,
            in("a0") trap_cx_ptr,
            in("a1") user_satp,
            options(noreturn)
        );
    }
}

#[no_mangle]
pub fn trap_from_kernel() -> ! {
    panic!("a trap from kernel!");
}

pub use context::{TrapContext};

```

#### os/src/trap/trap.S

```assembly
#  由于代表应用程序运行的任务和管理应用的操作系统各自有独立的页表和虚拟地址空间，
# 所以在操作系统的设计实现上需要考虑两个挑战。第一个挑战是 页表切换 。
# 于系统调用、中断或异常导致的应用程序和操作系统之间的上下文切换不像以前那么简单了，
# 因为在这些处理过程中需要切换页表，

.altmacro
.macro SAVE_GP n
    sd x\n, \n*8(sp)
.endm
.macro LOAD_GP n
    ld x\n, \n*8(sp)
.endm
    .section .text.trampoline
    .globl __alltraps
    .globl __restore
    .align 2
__alltraps:
    csrrw sp, sscratch, sp
    # now sp->*TrapContext in user space, sscratch->user stack
    # save other general purpose registers
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
    # we can use t0/t1/t2 freely, because they have been saved in TrapContext
    csrr t0, sstatus
    csrr t1, sepc
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # read user stack from sscratch and save it in TrapContext
    csrr t2, sscratch
    sd t2, 2*8(sp)
    # 应用地址空间中完成了保存 Trap 上下文的工作
    
    # load kernel_satp into t0
    ld t0, 34*8(sp)
    # 将内核地址空间的 token 载入到 t0 寄存器中
    
    # load trap_handler into t1
    ld t1, 36*8(sp)
    # 将 trap handler 入口点的虚拟地址载入到 t1 寄存器中
    
    # move to kernel_sp
    ld sp, 35*8(sp)
    # 直接将 sp 修改为应用内核栈顶的地址
    
    # switch to kernel space
    csrw satp, t0
    sfence.vma
    # 将 satp 修改为内核地址空间的 token 并使用 sfence.vma 刷新快表，
    # 这就切换到了内核地址空间
    # jump to trap_handler
    jr t1
    # 通过 jr 指令跳转到 t1 寄存器所保存的trap handler 入口点的地址

__restore:
    # a0: *TrapContext in user space(Constant); a1: user space token
    # switch to user space
    csrw satp, a1
    sfence.vma
    # 切换回应用地址空间
    
    csrw sscratch, a0
    # 将传入的 Trap 上下文位置保存在 sscratch 寄存器中，
    # 这样 __alltraps 中才能基于它将 Trap 上下文保存到正确的位置
    mv sp, a0
    # 将 sp 修改为 Trap 上下文的位置，后面基于它恢复各通用寄存器和 CSR
    
    # now sp points to TrapContext in user space, start restoring based on it
    # restore sstatus/sepc
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    # restore general purpose registers except x0/sp/tp
    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
    # back to user stack
    ld sp, 2*8(sp)
    sret
    # 通过 sret 指令返回用户态

```



#### os/src/loader.rs

```rust
// 在 os/src/build.rs 中，我们不再将丢弃了所有符号的应用二进制镜像链接进内核，
// 因为在应用二进制镜像中，内存布局中各个逻辑段的置和访问限制等信息都被裁剪掉了。
// 我们直接使用保存了逻辑段信息的 ELF 格式的应用可执行文件。这样 loader 子模块的设计实现也变得精简

// 获取链接到内核内的应用的数目
pub fn get_num_app() -> usize {
    extern "C" { fn _num_app(); }
    unsafe { (_num_app as usize as *const usize).read_volatile() }
}

// get_app_data 则根据传入的应用编号取出对应应用的 ELF 格式可执行文件数据。
// 它们和之前一样仍是基于 build.rs 生成的 link_app.S 给出的符号来确定其位置，
// 并实际放在内核的数据段中。 loader 模块中原有的内核和用户栈则分别作为逻辑段放在内核和用户地址空间中，
// 我们无需再去专门为其定义一种类型
pub fn get_app_data(app_id: usize) -> &'static [u8] {
    extern "C" { fn _num_app(); }
    let num_app_ptr = _num_app as usize as *const usize;
    let num_app = get_num_app();
    let app_start = unsafe {
        core::slice::from_raw_parts(num_app_ptr.add(1), num_app + 1)
    };
    assert!(app_id < num_app);
    unsafe {
        core::slice::from_raw_parts(
            app_start[app_id] as *const u8,
            app_start[app_id + 1] - app_start[app_id]
        )
    }
}

```

#### os/src/main.rs

``` rust
#![no_std]
#![no_main]
#![feature(panic_info_message)]
#![feature(alloc_error_handler)]

extern crate alloc;
// alloc 库需要我们提供给它一个 全局的动态内存分配器 ，它会利用该分配器来管理堆空间，
// 从而使得与堆相关的智能指针或容器数据结构可以正常工作
// 需要引入 alloc 库的依赖，由于它算是 Rust 内置的 crate ，我们并不是在 Cargo.toml 中进行引入，
// 而是在 main.rs 中声明即可
#[macro_use]
extern crate bitflags;

#[macro_use]
mod console;
mod lang_items;
mod sbi;
mod syscall;
mod trap;
mod loader;
mod config;
mod task;
mod timer;
mod sync;
mod mm;

use core::arch::global_asm;

global_asm!(include_str!("entry.asm"));
global_asm!(include_str!("link_app.S"));

fn clear_bss() {
    extern "C" {
        fn sbss();
        fn ebss();
    }
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
    println!("[kernel] Hello, world!");
    mm::init();
    println!("[kernel] back to world!");
    mm::remap_test();
    trap::init();
    //trap::enable_interrupt();
    trap::enable_timer_interrupt();
    timer::set_next_trigger();
    task::run_first_task();
    panic!("Unreachable in rust_main!");
}

```



#### os/Makefile

```makefile
# Building
TARGET := riscv64gc-unknown-none-elf
MODE := release
KERNEL_ELF := target/$(TARGET)/$(MODE)/os
KERNEL_BIN := $(KERNEL_ELF).bin
DISASM_TMP := target/$(TARGET)/$(MODE)/asm

# BOARD
BOARD ?= qemu
SBI ?= rustsbi
BOOTLOADER := ../bootloader/$(SBI)-$(BOARD).bin
K210_BOOTLOADER_SIZE := 131072

# KERNEL ENTRY
ifeq ($(BOARD), qemu)
	KERNEL_ENTRY_PA := 0x80200000
else ifeq ($(BOARD), k210)
	KERNEL_ENTRY_PA := 0x80020000
endif

# Run K210
K210-SERIALPORT	= /dev/ttyUSB0
K210-BURNER = ../tools/kflash.py

# Binutils
OBJDUMP := rust-objdump --arch-name=riscv64
OBJCOPY := rust-objcopy --binary-architecture=riscv64

# Disassembly
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
ifeq ($(BOARD),qemu)
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
		tmux split-window -h "riscv64-unknown-elf-gdb -ex 'file $(KERNEL_ELF)' -ex 'set arch riscv:rv64' -ex 'target remote localhost:1234'" && \
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
#![feature(linkage)]
#![feature(panic_info_message)]

#[macro_use]
pub mod console;
mod syscall;
mod lang_items;

#[no_mangle]
#[link_section = ".text.entry"]
pub extern "C" fn _start() -> ! {
    exit(main());
    panic!("unreachable after sys_exit!");
}

#[linkage = "weak"]
#[no_mangle]
fn main() -> i32 {
    panic!("Cannot find main!");
}


use syscall::*;

pub fn write(fd: usize, buf: &[u8]) -> isize { sys_write(fd, buf) }
pub fn exit(exit_code: i32) -> isize { sys_exit(exit_code) }
pub fn yield_() -> isize { sys_yield() }
pub fn get_time() -> isize { sys_get_time() }
```

#### user/src/linker.ld

```

OUTPUT_ARCH(riscv)
ENTRY(_start)

BASE_ADDRESS = 0x10000;
# 假定操作系统能够通过分页机制把不同应用的相同虚地址映射到不同的物理地址上
# 这样我们写应用就不用考虑应用的物理地址布局的问题，能够以一种更加统一的方式编写应用程序，
# 可以忽略掉一些不必要的细节

SECTIONS
{
    . = BASE_ADDRESS;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
    . = ALIGN(4K);
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    . = ALIGN(4K);
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    .bss : {
        *(.bss .bss.*)
        *(.sbss .sbss.*)
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

static mut S: [u64; LEN] = [0u64; LEN];

#[no_mangle]
unsafe fn main() -> i32 {
    let p = 3u64;
    let m = 998244353u64;
    let iter: usize = 300000;
    let mut cur = 0usize;
    S[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        S[next] = S[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_3 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, S[cur], m);
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

static mut S: [u64; LEN] = [0u64; LEN];

#[no_mangle]
unsafe fn main() -> i32 {
    let p = 5u64;
    let m = 998244353u64;
    let iter: usize = 210000;
    let mut cur = 0usize;
    S[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        S[next] = S[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_5 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, S[cur], m);
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

static mut S: [u64; LEN] = [0u64; LEN];

#[no_mangle]
unsafe fn main() -> i32 {
    let p = 7u64;
    let m = 998244353u64;
    let iter: usize = 240000;
    let mut cur = 0usize;
    S[cur] = 1;
    for i in 1..=iter {
        let next = if cur + 1 == LEN { 0 } else { cur + 1 };
        S[next] = S[cur] * p % m;
        cur = next;
        if i % 10000 == 0 {
            println!("power_7 [{}/{}]", i, iter);
        }
    }
    println!("{}^{} = {}(MOD {})", p, iter, S[cur], m);
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
	@cargo build --release

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
            - 调用 clear_bss 函数， 清除 除 .bss.stack 以外的 .bss 段
            - 调用 println! 打印 [kernel] Hello, world! , 此时处于内核态
            - 调用 mm::init

2. os/src/mm/mod.rs

    - 调用 init_heap
        - os/src/mm/heap_allocator.rs
        - 将一段数组的长度和大小作为参数 用于 HEAP_ALLOCATOR 管理
        - 堆管理的具体实现是由Rust的库实现
    - 调用 init_frame_allocator
        - os/src/mm/frame_allocator
        - 获取 ekernel 的地址
        - 将 ekernel 和 MEMORY_END 的地址取整转成物理地址后传入 FRAME_ALLOCATOR 
        - 跳转StackFrameAllocator的 init函数，即赋值给结构体StackFrameAllocator的current和end
    - 调用KERNEL_SPACE 的 activate
        - os/src/mm/memory_set.rs
        - 静态生成KERNEL_SPACE，它是MemorySet类型通过其new_kernel函数构造而成
        - new_kernel先调用 new_bare 函数获得一个MemorySet
            - new_bare函数调用PageTable 的new函数获得一个PageTable
                - PageTable 的new函数调用frame_alloc函数
                    - frame_alloc 函数先调用FRAME_ALLOCATOR函数的 alloc函数获得物理页
                        - alloc 函数先查看StackFrameAllocator的recycled这个Vec中有没有物理页可以返回
                        - 如果有就直接返回，没有就分配新的页，如果整个数组被分配完了，就返回None
                    - frame_alloc 函数调用FrameTracker结构的new函数将刚才得到的物理页表所对应的页清零
                    - 再将得到的物理页传到FRAME_ALLOCATOR的map函数，将Option<PhysPageNum>类型的 ppn 转成FrameTracker类型
                - 将获得的FrameTracker的ppn作为PageTable 的 root_ppn,并将这个FrameTracker存在在PageTable的frames这个Vec容器中
                - 返回这个PageTable
            - 生成一个空的areas的Vec容器
            - 返回这个MemorySet这个结构体
        - 调用获得的 MemorySet 的 map_trampoline 函数，
            - map_trampoline 调用 MemorySet的page_table的map函数
                - 先调用VirtAddr的from方法将usize的TRAMPOLIMNE转成VirtAddr
                - 再将VirtAddr类型转成VirtPageNum类型
                - 同理，将从linker.ld中获得的strampoline地址先转成物理地址，再转成物理页号
                - 最后给他们读写权限
            - 将获得的虚拟页号和物理页号以及标志位传入到map函数
            - map函数先调用PageTable的find_pte_create方法
                - find_pte_create方法中，先调用VirtPageNum 的indexes方法
                    - indexes方法中，先取出虚拟页号，分别取出三级页的索引，以数组的形式返回
                - 获取PageTable的root_ppn
                - 分别取出三级索引，如果找到了第一级索引（即物理页号），就将其对应的页表项直接返回
                - 如果某个索引走不通，就是不存在或不和法，就调用frame_alloc申请一个物理页，获取他的物理页号，再将这个物理页号传入到PageTableEntry的new函数中，生成一个物理页帧，再将物理页号存放到PageTable全局页管理器中
                - 将生成的新的物理页帧返回
            - 再调用PageTableEntry的new方法，通过得到的物理页号生成对应的物理页帧
            - 最后将通过虚拟页号得到的物理页帧指向传入到map函数的物理页帧
        - 打印出各个段的地址
        - 调用memory_set的push方法
            - 调用MapArea的new方法
                - 先将各个段的起始地址和结束地址转成usize类型，再转成虚拟地址
                - 将各个段的虚拟地址范围和映射类型以及映射的权限作为参数传入到new方法中
                - MapArea中的new方法将两个地址转成虚拟页号
                - 将两个虚拟页号作为区间传入到VPNRange的new方法中，而VPNRange的本质是SimpleRange类的别称，SimpleRange是一个泛型，只包含一个泛型的左值和右值，它的new方法就是将两个参数赋值给结构体的左右值
                - 生成一个空的键值对表BTreeMap(注意：它是有序的)
                - 将map_type 和 map_perm赋值
                - 返回生成的MapArea
        - 将生成的MapAreapush到开头生成的MemorySet的实例的 areas这Vec容器中
    - 返回生成的MemorySet 实例 memory_set
    - 调用KERNEL_SPACE 的 activate
        - 调用MemorySet结构体的成员PageTable的token方法
            - PageTable的Token方法，将root_ppn进行处理，返回一个usize
        - 将得到的Token写入到satp寄存器中，控制分页系统

    ### 到这完成分页

3. main函数中调用 mm::remap_test函数

4. main函数中调用 trap::init函数

    - os/src/trap/mod.rs
    - 调用 set_kernel_trap_entry
        - set_kernel_trap_entry 中 将 函数 trap_from_kernel 的地址和 trap的模式作为参数传入到stvec::write方法中，即设置 trap 的处理模式 和 对应处理函数的地址
        - 这里的 trap_from_kernel 函数只是仅仅调用一个 panic

5. main函数中调用 trap::enable_interrupt

    - os/src/mod.rs
    - enable_timer_interrupt 中

6. os/src/loader.rs

    - 调用 get_num_app 获取应用个数
    - 获取第一个程序的入口地址
    - 刷新指令缓冲区
    - 调用 get_base_i 获取 每个应用程序的起始地址
    - 将每个应用起始地址到 应用空间限制范围的地址区间 用 0 填充
    - 获取每个应用在 .data 段的起始地址和大小 和 在将会放置该程序的起始地址和程序大小
    - 调用 copy_from_slice 将程序从 .data 段复制到 相应的地址区间
    - 至此，程序加载完毕

7. os/src/main.rs

    - 调用 trap::enable_timer_interrupt

    - 调用 sie::set_stimer 使得 S 特权级时钟中断不会被屏蔽

 8. os/src/main.rs

    - 调用 timer::set_next_trigger
    - 设置时间片

    - 调用 get_time 获得当前时间， 即 mtime 的值
    - 估算出 10 ms 后 mtime 的值
    - 再将估算的值赋值给 mtimecmp

9. os/src/main.rs

    - 调用 task 模块的 run_first_task

  10. os/src/task/mod.rs

- 首先，静态加载一个全局的任务管理器

     TaskManager 结构 的实例 TASK_MANAGER,包含 num_app 和 inner

     inner 包括 数组 tasks 和 current_task

     数组 tasks 是 TaskControlBlock 结构的数组

     TaskControlBlock 结构体 包括 任务状态 task_status、 任务上下文 TaskContext

     (TaskContext 包含 那14个寄存器)、内存转换页 memory_set、trap的物理地址 trap_cx_ppn、和基地址base_size

 - 对任务数组进行初始化，对每个任务都形成一个任务控制块

      - os/src/tasks/tasks.rs
      - 将每个任务的地址和其编号传入到 TaskControlBlock 结构体的 new方法中，由于任务没有去除元数据，直接生成的ELF格式，所有该地址就相当于是elf_data的地址，
           - 在new中，先同过地址获取ELF结构的头的信息
           - 将TRAP_CONTEXT此时的虚拟地址转成物理地址
           - 将任务的状态置为 Ready
           - 调用config.rs中的kernel_stack_position通过app_id获取内核栈的上界和下界
           - 调用KERNEL_SPACE 的 insert_framed_area
                - 在 os/src/mm/memory_set.rs中
                - 将虚拟的起始地址和结束地址以及权限符号 传入到MapArea的new方法中
                - new方法中，将虚拟地址区间取整转成虚拟页号，再将这两个虚拟页号转成VPNRange这中迭代类型
                - 生成一个空的虚实页表映射的键值对
                - 最后返回这个映射表
           - 将这个映射表存放到MemorySet中的areas这个Vec容
      - 生成一个任务控制块
           - 调用TaskContext的goto_trap_return 函数
                - 将kernel_stack_top 传入到此函数中
                - 根据这个任务在内核栈中的位置，获取它的trap上下文
           - 根据任务状态、trap上下文、内存映射表、trap的物理页号、用户栈的起始地址生成一个任务控制块
      - 获得这个任务控制块中 trap物理页号的可变引用
      - 将获得的任务的入口地址、用户栈、物理页token、内核栈顶、trap_handler函数的地址传入到TrapContext的app_init_context 函数中
           - 在app_init_context函数中，生成一个trap上下文
           - 这个上下文包括 32的通用寄存器初始化为0，sstatus设为用户态、sepc指向ELF文件的入口地址、kernel_satp寄存器，kernel_sp内核栈、trap处理函数的入口，sp指向传入的用户的sp
      - 将这个生成的上下文传入到任务控制块中的任务上下文
      - 最后返回这任务控制块
      
- 静态的任务管理器 TASK_MANAGER 生成，包括任务数、任务数组 即各个任务的控制块 和 当前任务的编号

- 调用 run_first_task 函数，该函数调用任务管理器的 run_first_task 方法

     - 在run_first_task 方法中
     - 先获取任务管理器的锁，即使用权
     - 然后将next任务置为第0个任务的借用
     - 并将next任务的状态 置为 Running
     - 获取next任务的上下文的应用，给一个指针
     - 创建一个虚的任务上下文，将其全初始化为零，由于它不会真正用到，所有如何初始化都无影响
     - 我们的目的就是获取 任务上下文，此时已经获取，为了安全，将任务0的借用释放掉
     - 将构造的序虚任务上下文作为当前上下文的借用，next任务上下文的引用做为下一个任务上下文传入到 __switch 函数中，
     - 进行一番寄存器的操作后，切换到第一个任务的空间，执行第一个用户任务

## 对流程进行整理分析

### 先看结构和内存的示意图

### 堆管理

![HEAP_ALLOCATOR](../../../../Downloads/HEAP_ALLOCATOR.png)

地址空间

![Address](ch4地址空间.assets/Address.png)

### 内核内存管理

![内核内存管理](ch4地址空间.assets/内核内存管理.png)

1![内存图](ch4地址空间.assets/内存图.png)



### 任务管理

![任务管理 (ch4地址空间.assets/任务管理 (1).png)](../../../../Downloads/任务管理 (1).png)





1. 在物理内存上

    - sp -> boot_stack_top              .bss.stack

    - satp: 1000 0 ekernel.ceil

2. 进入到 内核地址空间         一切皆为虚拟地址

    - stvec -> trap_from_kernel   内核地址
    - sie::set_stimer 启动时钟中断
    - ra: trap_return 内核地址
    - s: [0; 12]
    - sp -> kernel_stack_top任务一      内核地址
    - restore_va = TRAMPOLINE + size 内核地址  ->  .text.trampoline 物理地址
    - jr restore_va
    - stvec: TRAMPOLINE 虚拟地址
    - trap_cx_ptr = TRAP_CONTEXT 虚拟地址
    - __restore
    - satp: 1000 0 app0.root_ppn

3. 进入到任务一的地址空间

    - sscratch: TRAP_CONTEXT 任务一的地址空间
    - sp -> TRAP_CONTEXT 任务一定的地址空间
    - sstatus: SPP:User
    - sepc : 任务一的 entrypoint
    - s: [0; 12]
    - sp -> user_sp = user_stack_top 任务一的地址空间
    - pc -> 任务一的 entrypoint
    - sret 
    - pc -> sepc

4. 进入到任务一

    - 执行 syscall 调用 ecall

    - stvec -> TRAMPOLINE 任务一的地址空间

    - __alltraps

    - sp -> 任务一的 TRAP_CONTEXT

    - sscratch -> User_stack_top   任务一的地址空间

        |   sscratch   |   -> 任务一的地址空间的 TRAP_CONTEXT

        |   sepc        |  -> ecall 后的一条指令

        |   sstatus    | 

        |   x31          |

        |     .             |
        |   x5            |

        |   x3            |

        |   x1            |

        |____ __ __|  任务一的 user_stack_top 

    - sp -> kernel_sp任务一   本身就是内核地址空间的地址

    - satp : 1000 0 ekernel.ceil

5. 进入到内核地址空间

    - jr trap_handler 本身就是内核地址空间的地址
    - stvec -> trap_from_kernel   内核地址空间的地址  意思是不接受内核的 Trap

6. stvec -> TRAMPOLINE

    - __restore
    - sapt -> 1000 0 app0.root_ppn

7. 进入到 任务一的地址空间

    - sscratch -> TRAP_CONTEXT
    - sp -> TRAM_CONTEXT
    - sstatus: SPP:User
    - sepc -> ecall 下一条指令
    - s[0; 12]
    - sp -> User_stack_top
    - pc -> ecall 的下一条指令

8. 进入到 任务一

9. 当任务切换时

    - 进入到内核地址空间

    - 将当前转态存到任务一的任务上下文  此时为内核地址空间，可以直接找到
    - 将任务二的 任务上下文加载到寄存器中
    - ret
    - pc -> trap_return

