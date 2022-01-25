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

#### ./cargo/config

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

#### os/Cargo.toml

```
[package]
name = "os"
version = "0.1.0"
edition = "2021"

# See more keys and their definitions at https://doc.rust-lang.org/cargo/reference/manifest.html

[dependencies]

# 此时还不需要配置什么
```

#### os/src/entry.asm

```
     .section .text.entry
     .globl _start
 _start:
     li x1, 100

# 这就是本节所说的 内核第一条指令
# 就是一条单纯的 riscv 的汇编代码
```

#### os/src/linker.ld

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

#### os/src/lang_items.rs

```rust
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
loop {}
}

//此时只是重写了 panic
//但未做任何处理
```

<font color="red">待解决的问题</font>	

<font color="red">我们尝试直接调用它，看会出现什么情况</font>	

<font color="red">它能否被调用，是否被调用</font>	

<font color="red">单当执行完 0x8020000 的 li ra, 100 后再次执行 si，后面都是unimp，但为什么还可以执行</font>	

<font color="red">跳转后的代码是否为 panic的，为什么一直si执行到 0x800007956后执行，会调到0x80200000</font>	

 <font color="red">再次在0x80007956处设断点后执行一圈，就跳到0x00000000处了，并且不能执行了</font>

------------------------------------------------------------------------ ==== ------------------------------------------------------------------------

#### os/src/main.rs

```rust
#![no_std]
#![no_main]

mod lang_items;

use core::arch::global_asm;
global_asm!(include_str!("entry.asm"));

// 由于这是个rust项目，我们并没有直接用汇编器编译 汇编代码
// 所以 我们是以内联汇编的形式将汇编代码导入到rust文件中，再编译
// 所以项目的入口还是 main.rs
// 只不过代码的执行如何不是main函数了
// 而是指定的_start

//fn main() {
//    println!("Hello, world!");
//}
```



## 执行流程

main.rs 中内联的 entry.esm 的一条riscv的汇编指令



