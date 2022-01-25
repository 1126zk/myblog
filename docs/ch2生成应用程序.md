# 生成能被内核识别的应用程序

## 第二章代码树

```
./os/src
Rust        13 Files   372 Lines
Assembly     2 Files    58 Lines

├── bootloader
│   ├── rustsbi-k210.bin
│   └── rustsbi-qemu.bin
├── LICENSE
├── os
│   ├── build.rs(新增：生成 link_app.S 将应用作为一个数据段链接到内核)
│   ├── Cargo.toml
│   ├── Makefile(修改：构建内核之前先构建应用)
│   └── src
│       ├── batch.rs(新增：实现了一个简单的批处理系统)
│       ├── console.rs
│       ├── entry.asm
│       ├── lang_items.rs
│       ├── link_app.S(构建产物，由 os/build.rs 输出)
│       ├── linker-k210.ld
│       ├── linker-qemu.ld
│       ├── main.rs(修改：主函数中需要初始化 Trap 处理并加载和执行应用)
│       ├── sbi.rs
│       ├── sync(新增：同步子模块 sync ，目前唯一功能是提供 UPSafeCell)
│       │   ├── mod.rs
│       │   └── up.rs(包含 UPSafeCell，它可以帮助我们以更 Rust 的方式使用全局变量)
│       ├── syscall(新增：系统调用子模块 syscall)
│       │   ├── fs.rs(包含文件 I/O 相关的 syscall)
│       │   ├── mod.rs(提供 syscall 方法根据 syscall ID 进行分发处理)
│       │   └── process.rs(包含任务处理相关的 syscall)
│       └── trap(新增：Trap 相关子模块 trap)
│           ├── context.rs(包含 Trap 上下文 TrapContext)
│           ├── mod.rs(包含 Trap 处理入口 trap_handler)
│           └── trap.S(包含 Trap 上下文保存与恢复的汇编代码)
├── README.md
├── rust-toolchain
├── tools
│   ├── kflash.py
│   ├── LICENSE
│   ├── package.json
│   ├── README.rst
│   └── setup.py
└── user(新增：应用测例保存在 user 目录下)
   ├── Cargo.toml
   ├── Makefile
   └── src
      ├── bin(基于用户库 user_lib 开发的应用，每个应用放在一个源文件中)
      │   ├── 00hello_world.rs
      │   ├── 01store_fault.rs
      │   ├── 02power.rs
      │   ├── 03priv_inst.rs
      │   └── 04priv_csr.rs
      ├── console.rs
      ├── lang_items.rs
      ├── lib.rs(用户库 user_lib)
      ├── linker.ld(应用的链接脚本)
      └── syscall.rs(包含 syscall 方法生成实际用于系统调用的汇编指令，
                     各个具体的 syscall 都是通过 syscall 来实现的)
```

## 本节文件解析

本节实现的是能够被内核识别并逐个加载运行的应用程序

要保证应用程序在用户态正常运行，能够被qemu-riscv64用户态模拟器 识别并执行

所以本节的代码都在user目录下

#### user/.cargo/config

```
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
    "-Clink-args=-Tsrc/linker.ld"
]
```

#### user/Cargo.toml

```
[package]
name = "user_lib"
version = "0.1.0"
edition = "2021"

# 将库的名字设置为： user_lib

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]
riscv = { git = "https://github.com/rcore-os/riscv", features = ["inline-asm"] } 

```

#### user/src/linker.ld

```
OUTPUT_ARCH(riscv)
ENTRY(_start)

# 在lib.rs 中定义了用户库的入库点 _start
# 通过rust的宏 #[link_section = ".text.entry"]
# 将_start的那段代码编译后的汇编代码放在一个名为 .text.entry 的段中
BASE_ADDRESS = 0x80400000;

# 程序的起始地址为 0x80400000

SECTIONS{
    . = BASE_ADDRESS;
    .text : {
        *(.text.entry)
        # 将 _start 所在的 .text.entry 段放在整个程序的开头
        # 即放在 0x80400000处
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
        # 进入用户库入口之后，要手动清空需要零初始化的 .bss 段
        # 由于内核目前还没有这个能力，只能在用户库中完成
        # 这里的start_bss 、 end_bss 便于初始化
    }
    /DISCARD/ : {
        *(.eh_frame)
        *(.debug*)
    }
}
```

#### user/src/syscall.rs

```rust
// 在子模块 syscall 中我们作为应用程序来通过 ecall 调用批处理系统提供的接口，
//由于应用程序运行在用户态（即 U 模式）， 
//ecall 指令会触发 名为 Environment call from U-mode 的异常，
//并 Trap 进入 S 模式执行批处理系统针对这个异常特别提供的服务代码

//我们知道系统调用实际上是汇编指令级的二进制接口，因此这里给出的只是使用 Rust 语言描述的 API 版本。
//在实际调用的时候，我们需要按照 RISC-V 调用规范（即ABI格式）在合适的寄存器中放置系统调用的参数，
//然后执行 ecall 指令触发 Trap。在 Trap 回到 U 模式的应用程序代码之后，
//会从 ecall 的下一条指令继续执行，同时我们能够按照调用规范在合适的寄存器中读取返回值

//在 RISC-V 调用规范中，和函数调用的 ABI 情形类似，约定寄存器 a0~a6 保存系统调用的参数， 
//a0~a1 保存系统调用的返回值。有些许不同的是寄存器 a7 用来传递 syscall ID，
//这是因为所有的 syscall 都是通过 ecall 指令触发的，
//除了各输入参数之外我们还额外需要一个寄存器来保存要请求哪个系统调用。
//由于这超出了 Rust 语言的表达能力，
//我们需要在代码中使用内嵌汇编来完成参数/返回值绑定和 ecall 指令的插入

use core::arch::asm;

const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

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
    //在第一章中，我们曾经使用 global_asm! 宏来嵌入全局汇编代码，
    //而这里的 asm! 宏可以将汇编代码嵌入到局部的函数上下文中。
    //相比 global_asm! ， asm! 宏可以获取上下文中的变量信息并允许嵌入的汇编代码对这些变量进行操作。
    //由于编译器的能力不足以判定插入汇编代码这个行为的安全性，
    //所以我们需要将其包裹在 unsafe 块中自己来对它负责
    ret
}

// 就是调用rustsbi提供的系统调用

pub fn sys_write(fd: usize, buffer: &[u8]) -> isize{
    syscall(SYSCALL_WRITE, [fd, buffer.as_ptr() as usize, buffer.len()])
}

pub fn sys_exit(exit_code: i32) -> isize {
    syscall(SYSCALL_EXIT, [exit_code as usize, 0, 0])
}
//再进行封装，便于应用程序调用


/*
/// 功能：将内存中缓冲区中的数据写入文件。
/// 参数：`fd` 表示待写入文件的文件描述符；
///      `buf` 表示内存中缓冲区的起始地址；
///      `len` 表示内存中缓冲区的长度。
/// 返回值：返回成功写入的长度。
/// syscall ID：64
fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize;

/// 功能：退出应用程序并将返回值告知批处理系统。
/// 参数：`xstate` 表示应用程序的返回值。
/// 返回值：该系统调用不应该返回。
/// syscall ID：93
fn sys_exit(xstate: usize) -> !;
*/
```

#### user/src/console.rs

```rust
use core::fmt::{self, Write};
use super::write;

struct Stdout;

const STDOUT: usize = 1;

impl Write for Stdout{
    fn write_str(&mut self, s: &str) -> fmt::Result{
        write(STDOUT, s.as_bytes());
        Ok(())
    }
    //我们把 console 子模块中 Stdout::write_str 改成基于 write 的实现，
    //且传入的 fd 参数设置为 1，它代表标准输出， 也就是输出到屏幕
}

pub fn print(args: fmt::Arguments){
    Stdout.write_fmt(args).unwrap();
}

#[macro_export]
macro_rules! print{
    ($fmt: literal $(, $($args: tt)+)?) => {
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

#### user/src/lang_items.rs

```rust
use crate::println;

#[panic_handler]
fn panic_handler(panic_info: &core::panic::PanicInfo) -> ! {
    let err = panic_info.message().unwrap();
    if let Some(location) = panic_info.location() {
        println!("Panicked at {}:{}, {}", location.file(), location.line(), err);
    } else{
        println!("Panicked: {}", err);
    }
    loop {}
}

// 除了没有shutdown，没有本质区别
```

#### user/src/lib.rs

```rust
#![no_std]
#![feature(panic_info_message)]
// 为了获取panic的信息
#![feature(linkage)]
// 为了支持弱连接

pub mod console;
mod syscall;
mod lang_items;

#[no_mangle]
#[link_section = ".text.entry"]
//使用 Rust 的宏将 _start 这段代码编译后的汇编代码中放在一个名为 .text.entry 的代码段中，
//方便我们在后续链接的时候调整它的位置使得它能够作为用户库的入口
pub extern "C" fn _start() -> ! {
    clear_bss();
    //手动清空需要零初始化的 .bss 段
    exit(main());
    panic!("unreachable after sys_exit!");
}

//我们使用 Rust 的宏将其函数符号 main 标志为弱链接。这样在最后链接的时候，
//虽然在 lib.rs 和 bin 目录下的某个应用程序都有 main 符号，
//但由于 lib.rs 中的 main 符号是弱链接，链接器会使用 bin 目录下的应用主逻辑作为 main 。
//这里我们主要是进行某种程度上的保护，如果在 bin 目录下找不到任何 main ，
//那么编译也能够通过，但会在运行时报错。
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

// 进一步封装，从而更加接近Linux等平台的实际系统调用接口
```

#### user/src/bin/00hello_world.rs

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;
// 引入外部库
// 库名 以Cargo。toml 文件规定为主

#[no_mangle]
fn main() -> i32{
    println!("Hello, world!");
    0
}
```

> Hello, world!

#### user/src/bin/01store_fault.rs

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

#[no_mangle]
fn main() -> i32{
    println!("Into Test store_fault, we will insert an invalid store operation...");
    println!("Kernel should kill this application!");
    unsafe{(0x0 as *mut u8).write_volatile(0);}
    // 访问一个非法的物理地址
    0
}
```

> Into Test store_fault, we will insert an invalid store operation...
> Kernel should kill this application!
> Segmentation fault (core dumped)

#### user/src/bin/02priv_inst.rs

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

use core::arch::asm;

#[no_mangle]
fn main() -> i32{
    println!("Try to execute privileged instruction in U Mode");
    println!("Kernel should kill this application!");
    unsafe{
        asm!("sret");
        // 在用户态执行内核态的特权指令 sret
    }
    0
}
```

> Try to execute privileged instruction in U Mode
> Kernel should kill this application!
> Illegal instruction (core dumped)

#### user/src/bin/03priv_csr.rs

```rust
#![no_std]
#![no_main]

#[macro_use]
extern crate user_lib;

use riscv::register::sstatus::{self, SPP};

#[no_mangle]
fn main() -> i32{
    println!("Try to access privileged CSR in U Mode");
    println!("Kernel should kill this application!");
    unsafe{
        sstatus::set_spp(SPP::User);
    }
    // 在用户态修改内核态CSR sstatus
    0
}
```

> Try to access privileged CSR in U Mode
> Kernel should kill this application!
> Illegal instruction (core dumped)

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

elf:
	@cargo build --release
	@echo $(APPS)
	@echo $(ELFS)
	@echo $(BINS)

binary: elf
	$(foreach elf, $(ELFS), $(OBJCOPY) $(elf) --strip-all -O binary $(patsubst $(TARGET_DIR)/%, $(TARGET_DIR)/%.bin, $(elf));)

build: binary

```



<font color="red">去掉库函数的引用会怎样</font>

因为 #![no_std]

println! 都得自己实现，所以不能去掉



## 执行流程

_start -> clear_bss -> main -> exit