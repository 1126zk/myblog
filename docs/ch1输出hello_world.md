# 输出hello world

# 内核第一条指令



## 第一章代码树

```
./os/src
Rust        4 Files   119 Lines
Assembly    1 Files    11 Lines

├── bootloader(内核依赖的运行在 M 特权级的 SBI 实现，本项目中我们使用 RustSBI)
│   ├── rustsbi-k210.bin(可运行在 k210 真实硬件平台上的预编译二进制版本)
│   └── rustsbi-qemu.bin(可运行在 qemu 虚拟机上的预编译二进制版本)
├── LICENSE
├── os(我们的内核实现放在 os 目录下)
│   ├── Cargo.toml(内核实现的一些配置文件)
│   ├── Makefile
│   └── src(所有内核的源代码放在 os/src 目录下)
│       ├── console.rs(将打印字符的 SBI 接口进一步封装实现更加强大的格式化输出)
│       ├── entry.asm(设置内核执行环境的的一段汇编代码)
│       ├── lang_items.rs(需要我们提供给 Rust 编译器的一些语义项，目前包含内核 panic 时的处理逻辑)
│       ├── linker-k210.ld(控制内核内存布局的链接脚本以使内核运行在 k210 真实硬件平台上)
│       ├── linker-qemu.ld(控制内核内存布局的链接脚本以使内核运行在 qemu 虚拟机上)
│       ├── main.rs(内核主函数)
│       └── sbi.rs(调用底层 SBI 实现提供的 SBI 接口)
├── README.md
├── rust-toolchain(控制整个项目的工具链版本)
└── tools(自动下载的将内核烧写到 k210 开发板上的工具)
   ├── kflash.py
   ├── LICENSE
   ├── package.json
   ├── README.rst
   └── setup.py
```



## 本节文件解析

### bootloader 文件夹下

- rustsbi-qemu.bin

### os 文件夹下

- ./cargo/config

```
# 配置文件没有注释， 默认 # 后面接注释

[build]
target = "riscv64gc-unknown-none-elf"

# 指定目标平台，不然编译时要带上参数
# cargo run --target riscv64gc-unknown-none-elf

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-arg=-Tsrc/linker.ld", "-Cforce-frame-pointers=yes"
]

# 用于指定链接文件的地址
# -T后面接文件路径
# -Cforce-frame-pointers=yes 估计是llvm中对指针的优化或其他底层操作

```

- os/Cargo.toml

```
[package]
name = "os"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

# 此时还不需要配置什么
```

### os/src 文件夹下

- os/src/entry.asm

```asm
	.section .text.entry
	.global _start
_start:
	la sp, boot_stack_top
	call rust_main
	
	# 设置栈顶位置后将 控制权转交到 Rust入口

	.section .bss.stack
# 关于 .bss.stack
#前面我们提到过 .bss 段一般放置需要被初始化为零的数据。
#然而栈并不需要在使用前被初始化为零，因为在函数调用的时候我们会插入栈帧覆盖已有的数据。
#我们尝试将其放置到全局数据 .data 段中但最后未能成功，因此才决定将其放置到 .bss 段中。
#全局符号 sbss 和 ebss 分别指向 .bss 段除 .bss.stack 以外的起始和终止地址，
#我们在使用这部分数据之前需要将它们初始化为零
	.global boot_stack
boot_stack:
	.space 4096 * 16
	#预留了一块大小为 4096 * 16 字节也就是64KB的空间用作接下来要运行的程序的栈空间
	.global boot_stack_top
boot_stack_top:

## 注意：
# 我们基本上说明了函数调用是如何基于栈来实现的。
#不过我们可以暂时先忽略掉这些细节，因为我们现在只是需要在初始化阶段完成栈的设置，
#也就是设置好栈指针 sp 寄存器，编译器会帮我们自动完成后面的函数调用相关机制的代码生成。
#麻烦的是， sp 的值也不能随便设置，至少我们需要保证它指向合法的物理内存，
#而且不能与程序的其他代码、数据段相交，因为在函数调用的过程中，栈区域里面的内容会被修改

# 这段话讲了为什么要先设置好sp寄存器
# 同时注意，这是riscv汇编的函数调用过程，关于函数调用约定是存在的
# 即什么寄存器该保留在栈中，函数调用的栈帧 就在这个栈中
# 只不过rust编译器知道了目前平台是 riscv，riscv的函数调用规范它知道
# 它会在内部对寄存器的使用做出调整， 编译器自动生成函数调用机制相关的代码
# 简化了我们开发的流程
```

- os/src/linker.ld

```ld
# 链接脚本

OUTPUT_ARCH(riscv)
# 指定目标文件所在的平台
ENTRY(_start)
# 指定入口地址
# 在程序中执行的第一条指令称为入口点。 您可以使用 ENTRY 链接器脚本命令来设置入口点。 参数是符号名称
BASE_ADDRESS = 0x80200000;
# 类似一个全局变量
# 指令内核从 0x80200000 处开始存放

SECTIONS
{
    . = BASE_ADDRESS;
    skernel = .;

    stext = .;
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
    # 内核从 0x80200000 开始，
    # 最开时存放的是 代码段，
    # 而且是指定的 .text.entry 段
    # 即 entry.asm 的 第一个代码段
    # 其他的代码段也存放在此处

    . = ALIGN(4K);
    etext = .;
    srodata = .;
    .rodata : {
        *(.rodata .rodata.*)
        *(.srodata .srodata.*)
    }
    
    # 此处存放所有的 rodata 段

    . = ALIGN(4K);
    erodata = .;
    sdata = .;
    .data : {
        *(.data .data.*)
        *(.sdata .sdata.*)
    }
    
    # 此处存放所有 data 段

    . = ALIGN(4K);
    edata = .;
    .bss : {
        *(.bss.stack)
        sbss = .;
        *(.bss .bss.*)
        *(.sbss .sbss.*)
    }
	
	# 此处存放所有 bss 段
	# 注意，有个 特殊的栈段也存放在此处
	# 即 .bss.stack 段
	
    . = ALIGN(4K);
    ebss = .;
    ekernel = .;

    /DISCARD/ : {
        *(.eh_frame)
    }
}

```

- os/src/sbi.rs

```rust
#![allow(unused)]
//并非所有的系统调用都会被用到，所以allow(unused)
const SBI_SET_TIMER:usize = 0;
const SBI_CONSOLE_PUTCHAR:usize = 1;
const SBI_CONSOLE_GETCHAR:usize = 2;
const SBI_CLEAR_IPI:usize = 3;
const SBI_SEND_IPI:usize = 4;
const SBI_REMOTE_FENCE_I:usize = 5;
const SBI_REMOTE_SFENCE_VMA:usize = 6;
const SBI_REMOTE_SFENCE_VMA_ASID:usize = 7;
const SBI_SHUTDOWN:usize = 8;

// rust-sbi提供了系统调用，
/*
const LEGACY_SET_TIMER: usize = 0x0;
const LEGACY_CONSOLE_PUTCHAR: usize = 0x01;
const LEGACY_CONSOLE_GETCHAR: usize = 0x02;
// const LEGACY_CLEAR_IPI: usize = 0x03;
const LEGACY_SEND_IPI: usize = 0x04;
// const LEGACY_REMOTE_FENCE_I: usize = 0x05;
// const LEGACY_REMOTE_SFENCE_VMA: usize = 0x06;
// const LEGACY_REMOTE_SFENCE_VMA_ASID: usize = 0x07;
const LEGACY_SHUTDOWN: usize = 0x08;
这是 rustsbi中 src/ecall.rs 的源码
*/
// 这里定义的 8 个系统调用 和 rustsbi提供的是一致
// 本质就是通过 riscv汇编 的ecall指令调用 rustsbi提供的系统调用

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

//这里本质就是对rustsbi提供的系统调用的一层封装

pub fn console_putchar(c:usize){
	sbi_call(SBI_CONSOLE_PUTCHAR, c, 0, 0);
}

pub fn shutdown() -> ! {
	sbi_call(SBI_SHUTDOWN, 0, 0, 0);
	panic!("It should shutdown!");
}

//进一步进行封装，简化调用的形式，只需要必要的参数
//同时也可以添加一些额外的信息
```

- os/src/console.rs

```rust
use crate::sbi::console_putchar;
use core::fmt::{self, Write};

struct Stdout;
//结构体 Stdout 不包含任何字段，因此它被称为类单元结构体

impl Write for Stdout{
    // core::fmt::Write trait 包含一个用来实现 println! 宏很好用的 write_fmt 方法，
    //为此我们准备为结构体 Stdout 实现 Write trait 。
	fn write_str(&mut self, s:&str) -> fmt::Result{
        //在 Write trait 中， write_str 方法必须实现，因此我们需要为 Stdout 实现这一方法，
        //它并不难实现，只需遍历传入的 &str 中的每个字符并调用 console_putchar 就能将传入的整个字符串打印到屏幕上。
		for c in s.chars(){
			console_putchar(c as usize);
		}
		Ok(())
	}
}

//在此之后 Stdout 便可调用 Write trait 提供的 write_fmt 方法并进而实现 print 函数。
pub fn print(args: fmt::Arguments){
	Stdout.write_fmt(args).unwrap();
}
//https://doc.rust-lang.org/core/fmt/trait.Write.html#method.write_fmt

//在声明宏（Declarative macros） print! 和 println! 中会调用 print 函数完成输出。
#[macro_export]
macro_rules! print{
	($fmt: literal $(, $($arg: tt)+)?) => {
		$crate::console::print(format_args!($fmt $(, $($arg)+)?));
	}
}

#[macro_export]
macro_rules! println{
	($fmt: literal $(, $($arg: tt)+)?) => {
		$crate::console::print(format_args!(concat!($fmt, "\n") $(, $($arg)+)?));
	}
}

```

<font color="red">trait的实现以及宏的声明等后续学完rust后再回过头来看</font>

------------------------------------------------------------------------- == -------------------------------------------------------------------------------

- os/src/lang_items.rs

```rust
use core::panic::PanicInfo;
use crate::sbi::shutdown;

#[panic_handler]
fn panic(info: &PanicInfo) -> ! {
	if let Some(location) = info.location() {
		println!(
			"Panicked at {}:{} {}",
			location.file(),
			location.line(),
			info.message().unwrap()
		);
	} else {
		println!("Panicked: {}", info.message().unwrap());
	}
    //我们需要在 main.rs 开头加上 #![feature(panic_info_message)] 
    //才能通过 PanicInfo::message 获取报错信息
	shutdown()
}

```

- os/src/main.rs

```rust
#![no_std]
#![no_main]
#![feature(panic_info_message)]
#[macro_use]

mod console;
mod lang_items;
mod sbi;

use core::arch::global_asm;
global_asm!(include_str!("entry.asm"));

fn clear_bss(){
	extern "C" {
		fn sbss();
		fn ebss();
        //extern “C” 可以引用一个外部的 C 函数接口（这意味着调用它的时候要遵从目标平台的 C 语言调用规范）。
        //但我们这里只是引用位置标志并将其转成 usize 获取它的地址。由此可以知道 .bss 段两端的地址
	}
    // 通过这种方法，我们可以获取 linker.ld 文件中段的地址
	(sbss as usize..ebss as usize).for_each(|a| {
		unsafe { (a as *mut u8).write_volatile(0)}
	});
    // Rust 的迭代器和闭包，后续再了解
}

#[no_mangle]
pub fn rust_main() -> ! {
	clear_bss();
	println!("Hello, world");
	panic!("Shutdown machine!");
}



```

<font color="red">关于PanicInfo的使用还有待了解</font>

<font color="red">为什么panic!("Shutdown machine!") 可以打印出来，但shut down() 的panic!("It should shutdown!")没有被打印了</font>

<font color="red">既然已经要关机了，为什么还要写句话：panic!("It should shutdown!")</font>

## 执行流程

1. 进入entr.asm，设置函数栈，跳转到 rust_main
2. 清零除.bss.stack以外的.bss段
3. 调用println! -> print -> Stdout.write_fmt
4. 调用 painc! -> println! -> shutdown -> sbi_call(SBI_SHUTDOWN, 0, 0, 0)



