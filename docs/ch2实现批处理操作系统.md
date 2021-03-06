

# 实现批处理操作系统

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

## 本节的任务

在批处理操作系统中，每当一个应用执行完毕，我们都需要将下一个要执行的应用的代码和数据加载到内存

如何找到应用所在的位置

我们将多个应用程序的代码和内核的代码通过编程技巧将其 绑定在一起，是一种静态绑定

基于静态编码留下的绑定信息，内核就可以找到每个应用程序文件二进制代码的起始地址和长度，并加载到内存中运行

## 本节文件解析

### os文件夹下

#### os/.cargo/config

```
[build]
target = "riscv64gc-unknown-none-elf"

[target.riscv64gc-unknown-none-elf]
rustflags = [
	"-Clink-arg=-Tsrc/linker.ld","-Cforce-frame-pointers=yes"
]

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
# 用于引用 Rust 的 riscv 库，方便操作CSR
lazy_static = {version = "1.4.0", features = ["spin_no_std"]}
# 调用外部库 lazy_static ， 用于获取 lazy_static! 宏
```

#### os/src/entry.asm

```asm
# 	.section .text.entry
# 	.global _start
# _start:
# 	li x1, 100

	.section .text.entry
	.global _start
_start:
	la sp, boot_stack_top
	call rust_main

	.section .bss.stack
	.global boot_stack
boot_stack:
	.space 4096 * 16
	.global boot_stack_top
boot_stack_top:
```



#### os/build.rs

```rust
// 用于生成 link_app.S

use std::io::{Result, Write};
use std::fs::{File, read_dir};

fn main(){
    println!("cargo:rerun-if-changed=../user/src/");
    println!("cargo:rerun-if-changed={}", TARGET_PATH);
    insert_app_data().unwrap();
}

static TARGET_PATH: &str = "../user/target/riscv64gc-unknown-none-elf/release/";

fn insert_app_data() -> Result<()> {
    let mut f = File::create("src/link_app.S").unwrap();
    // 创建 link_app.S 文件
    let mut apps: Vec<_> = read_dir("../user/src/bin")
    // 读取user/src/bin 目录下的文件， 获取文件名
        .unwrap()
        .into_iter()
        .map(|dir_entry|{
            let mut name_with_ext = dir_entry.unwrap().file_name().into_string().unwrap();
            name_with_ext.drain(name_with_ext.find('.').unwrap()..name_with_ext.len());
            name_with_ext
        }) 
        .collect();
    apps.sort();
    // 将文件排序

    writeln!(f, r#"
    .align 3
    .section .data
    .global _num_app
_num_app:
    .quad {}"#, apps.len())?;
    // 应用文件的个数

    for i in 0..apps.len() {
        writeln!(f, r#"    .quad app_{}_start"#, i)?;
    }
    writeln!(f, r#"    .quad app_{}_end"#, apps.len() - 1)?;

    for (idx, app) in apps.iter().enumerate(){
        println!("app_{}: {}", idx, app);
        writeln!(f, r#"
    .section .data
    .global app_{0}_start
    .global app_{0}_end
app_{0}_start:
    .incbin "{2}{1}.bin"
app_{0}_end:"#, idx, app, TARGET_PATH)?;
    }
    // 循环，将应用都放在 .data段中，并获取其 开始地址和结束地址
    // 按格式生成 link_app.S 
    Ok(())
}
```

#### os/src/link_app.S

```asm
#它一开始并不存在，而是在构建操作系统时自动生成的。当我们使用 make run 让系统运行的过程中，这个汇编代码 link_app.S 就生成了

    .align 3
    .section .data
    .global _num_app
_num_app:
	# .quad 用于定义一个 64位的数据
    .quad 5
    .quad app_0_start
    .quad app_1_start
    .quad app_2_start
    .quad app_3_start
    .quad app_4_start
    .quad app_4_end

    .section .data
    .global app_0_start
    .global app_0_end
app_0_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/00hello_world.bin"
app_0_end:

    .section .data
    .global app_1_start
    .global app_1_end
app_1_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/01store_fault.bin"
    # 前面我们从ELF格式可执行文件剥离元数据后放入到二进制镜像文件中
    # 这里直接将 这个镜像文件 放在内核的数据段 链接到内核中
app_1_end:

    .section .data
    .global app_2_start
    .global app_2_end
app_2_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/02power.bin"
app_2_end:

    .section .data
    .global app_3_start
    .global app_3_end
app_3_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/03priv_inst.bin"
app_3_end:

    .section .data
    .global app_4_start
    .global app_4_end
app_4_start:
    .incbin "../user/target/riscv64gc-unknown-none-elf/release/04priv_csr.bin"
app_4_end:

```

#### src/sync/up.rs

```rust
use core::cell::{RefCell, RefMut};

pub struct UPSafeCell<T>{
    inner: RefCell<T>,
}

unsafe impl<T> Sync for UPSafeCell<T> {}

impl<T> UPSafeCell<T> {
    pub unsafe fn new(value: T) -> Self{
        Self{ inner: RefCell::new(value)}
    }

    pub fn exclusive_access(&self) -> RefMut<'_, T> {
        self.inner.borrow_mut()
    }
}

// 关于所有权 和 安全 相关的rust编程技巧， 后面再关注
```

#### os/src/sync/mod.rs

```rust
mod up;

pub use up::UPSafeCell;
```



#### os/src/batch.rs

```rust
// 本节的核心代码

use lazy_static::*;
use crate::trap::TrapContext;
use crate::sync::UPSafeCell;
use core::arch::asm;

const USER_STACK_SIZE: usize = 4096 * 2;
const KERNEL_STACK_SIZE: usize = 4096 * 2;
// 常数指出 内核栈和用户栈的大小 分别为8KB
// 以全局变量的形式实例化在操作系统的 .bss 段中

const MAX_APP_NUM: usize = 16;
const APP_BASE__ADDRESS: usize = 0x80400000;
const APP_SIZE_LIMIT: usize = 0x20000;


#[repr(align(4096))]
struct KernelStack{
    data: [u8; KERNEL_STACK_SIZE],
}

#[repr(align(4096))]
struct UserStack{
    data: [u8; USER_STACK_SIZE],
}
// 表示用户栈和内核栈，都是字节数组的简单包装
// 用于保存在trap发生前 原控制流的寄存器状态


static KERNEL_STACK: KernelStack = KernelStack {data: [0; KERNEL_STACK_SIZE]};
static USER_STACK: UserStack = UserStack { data: [0; USER_STACK_SIZE]};

impl KernelStack{
    fn get_sp(&self) -> usize{
        self.data.as_ptr() as usize + KERNEL_STACK_SIZE
        // .as_ptr 获取字符串字面量的地址
    }
    pub fn push_context(&self, cx: TrapContext) -> &'static mut TrapContext {
        let cx_ptr = (self.get_sp() - core::mem::size_of::<TrapContext>()) as *mut TrapContext;
        unsafe { *cx_ptr = cx;}
        unsafe { cx_ptr.as_mut().unwrap() }
    }
}

impl UserStack {
    fn get_sp(&self) -> usize {
        self.data.as_ptr() as usize + USER_STACK_SIZE
    }
}
// 实现 get_sp 方法 来获取栈顶地址
// 由于在RISCV中栈是向下增长的，只需返回包裹的数组的结尾地址
// 用户栈和内核栈的切换， 只需要将 sp 寄存器的值修改为 get_sp 的返回值即可

// 应用管理器 本节的核心组件
// 能够找到并加载应用程序二进制码
struct AppManager{
    num_app: usize,
    // 保存应用数量
    current_app: usize,
    // 当前执行到了第几个应用
    app_start: [usize; MAX_APP_NUM+1],
    // 保存应用的各自的位置
}
// 根据应用程序位置信息，初始化好应用所需内存空间，并加载应用执行

impl AppManager {
    pub fn print_app_info(&self){
        println!("[kernel] num_app = {}", self.num_app);
        for i in 0..self.num_app {
            println!("[kernel] app_{} [{:#x}, {:#x})", i, self.app_start[i], self.app_start[i+1]);
        }
    }

    unsafe fn load_app(&self, app_id: usize){
        if app_id >= self.num_app{
            panic!("All applications completed!");
        }
        println!("[kernel] Loading app_{}", app_id);

        asm!("fence.i");
        // 汇编指令 fence.i 是用来清理 i-cache 的
        // 通常情况下， CPU 会认为程序的代码段不会发生变化，因此 i-cache 是一种只读缓存。
        //但在这里，我们将修改会被 CPU 取指的内存区域，这会使得 i-cache 中含有与内存中不一致的内容。
        //因此我们这里必须使用 fence.i 指令手动清空 i-cache ，让里面所有的内容全部失效，
        //才能够保证CPU访问内存数据和代码的正确性

        core::slice::from_raw_parts_mut(
            APP_BASE__ADDRESS as *mut u8,
            APP_SIZE_LIMIT
        ).fill(0);
        let app_src = core::slice::from_raw_parts(
            self.app_start[app_id] as *const u8,
            self.app_start[app_id + 1] - self.app_start[app_id]
        );
        let app_dst = core::slice::from_raw_parts_mut(
            APP_BASE__ADDRESS as *mut u8,
            app_src.len()
        );
        app_dst.copy_from_slice(app_src);
    }

    pub fn get_current_app(&self) -> usize {self.current_app}
    
    pub fn move_to_next_app(&mut self){
        self.current_app += 1;
    }
}

// 使用 lazy_static! 宏
// lazy_static! 宏提供了全局变量的运行时初始化功能
// 有关宏的具体细节， 后面再了解
lazy_static! {
    // 静态生成一个全局的 AppManager 的一个实例 APP_MANAGER
    static ref APP_MANAGER: UPSafeCell<AppManager> = unsafe {UPSafeCell::new({
        extern "C" {fn _num_app();}
        // 找到 link_app.S 中提供的符号 _num_app
        // 并从这里开始解析出应用数量以及各个应用的起始地址
        // link_app.S 中获取 _num_app 的地址，从而可以解析出
        // _num_app:
        // 		.quad 5
        // 		.quad app_0_start
        // 		.quad app_1_start
        // 		.quad app_2_start
        // 		.quad app_3_start
        // 		.quad app_4_start
        // 		.quad app_4_end
        let num_app_ptr = _num_app as usize as *const usize;
        let num_app = num_app_ptr.read_volatile();
        let mut app_start: [usize; MAX_APP_NUM + 1] = [0; MAX_APP_NUM + 1];
        let app_start_raw: &[usize] = core::slice::from_raw_parts(
            num_app_ptr.add(1), num_app + 1
        );
        app_start[..=num_app].copy_from_slice(app_start_raw);
        AppManager {
            num_app,
            current_app: 0,
            app_start,
        }
        // 初始化 AppManager 的全局实例 APP_MANAGER
    })};
}

// batch 子模块对外暴露的接口如下

pub fn init() {
    print_app_info();
}
// 调用 print_app_info 的时候第一次用到了全局变量 APP_MANAGER ，
//它也是在这个时候完成初始化

pub fn print_app_info() {
    APP_MANAGER.exclusive_access().print_app_info();
    // exclusive_access 是 os/src/sync/up.rs 模块提供的用于内存安全的一种函数
}

pub fn run_next_app() -> ! {
    let mut app_manager = APP_MANAGER.exclusive_access();
    let current_app = app_manager.get_current_app();
    unsafe {
        app_manager.load_app(current_app);
    }

    app_manager.move_to_next_app();
    drop(app_manager);

    extern "C" {fn __restore(cx_addr: usize);}
    unsafe {
        __restore(KERNEL_STACK.push_context(
            TrapContext::app_init_context(APP_BASE__ADDRESS, USER_STACK.get_sp())
        ) as *const _ as usize);
        // 在内核栈上压入一个trap 上下文
        // sepc 是应用程序入口地址 0x80400000
        // sp 寄存器指向用户栈
        // sstatus 的 SPP 字段被设置为 User
        // push_context 的返回值是 内核栈压入 Trap 上下文之后的栈顶
        // 它被作为 __restore 的参数， 这使得 __restore 函数中 sp 仍然可以指向内核栈的栈顶
        // 这之后就和执行一次普通的 __restore 函数调用一样了
    }
    panic!("Unreachable in batch::run_current_app!");
    // 这个不会被触发，在此之前会有一个panic，就直接 shutdown了
}
// 批处理操作系统的核心操作，即加载并运行下一个应用程序。
// 当批处理操作系统完成初始化或者一个应用程序运行结束或出错之后会调用该函数
```

#### os/src/trap/context.rs

```rust
use riscv::register::sstatus::{Sstatus, self, SPP};

// Trap 上下文，类似函数调用上下文，
// 即在Trap发生时需要保存的物理资源内容
#[repr(C)]
pub struct TrapContext{
    pub x: [usize; 32],
    // 所有通用寄存器
    // 找出哪些寄存器无需保存很难，甚至不可能，那就直接全部保存
    pub sstatus: Sstatus,
    pub sepc: usize,
    // 对于 CSR 而言，我们知道进入 Trap 的时候，
    //硬件会立即覆盖掉 scause/stval/sstatus/sepc 的全部或是其中一部分。scause/stval 的情况是：
    //它总是在 Trap 处理的第一时间就被使用或者是在其他地方保存下来了，因此它没有被修改并造成不良影响的风险。
    //而对于 sstatus/sepc 而言，它们会在 Trap 处理的全程有意义（在 Trap 控制流最后 sret 的时候还用到了它们），
    //而且确实会出现 Trap 嵌套的情况使得它们的值被覆盖掉。
    //所以我们需要将它们也一起保存下来，并在 sret 之前恢复原样
}

impl TrapContext{
    pub fn set_sp(&mut self, sp: usize) {
        self.x[2] = sp;
        // x2 = sp
    }
    pub fn app_init_context(entry: usize, sp: usize) -> Self{
        let mut sstatus = sstatus::read();
        sstatus.set_spp(SPP::User);
        let mut cx = Self{
            x: [0; 32],
            sstatus,
            sepc: entry,
        };
        cx.set_sp(sp);
        cx
        // 修改其中的 sepc 寄存器为应用程序入口点 entry
        // sp 寄存器 为我们设定的一个栈指针，
        // sstatus 寄存器的 SPP 字段设置为 User
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
    },
    stval
};

use crate::syscall::syscall;
use crate::batch::run_next_app;
use core::arch::global_asm;

global_asm!(include_str!("trap.S"));

// 在操作系统初始化的时候， 我们需要修改stvec 寄存器来指向正确的Trap处理入口点
pub fn init(){
    extern "C" {fn __alltraps();}
    // 引入外部符号 __alltraps
    unsafe{
        stvec::write(__alltraps as usize, TrapMode::Direct);
        // 将stvec设置为Direct模式指向它的地址
    }
}

#[no_mangle]
pub fn trap_handler(cx: &mut TrapContext) -> &mut TrapContext{
    let scause = scause::read();
    let stval = stval::read();
    match scause.cause(){
        // scause 寄存器所保存的 Trap 的原因进行分发处理。这里我们无需手动操作这些 CSR ，
        // 而是使用 Rust 的 riscv 库来更加方便的做这些事情
        // 要引入riscv库， 需要在os/Cargo.toml 配置文件中 添加
        // [dependencies]
	   // riscv = { git = "https://github.com/rcore-os/riscv", features = ["inline-asm"] }
        Trap::Exception(Exception::UserEnvCall) => {
            // 如果触发 Trap 的原因是来自 U态 的 系统调用
            cx.sepc += 4;
            // 首先修改保存在内核栈上的 Trap 上下文里面 sepc，让其增加 4
            // 我们知道这是一个由 ecall 指令触发的系统调用，在进入 Trap 的时候，
            // 硬件会将 sepc 设置为这条 ecall 指令所在的地址（因为它是进入 Trap 之前最后一条执行的指令）。
            // 而在 Trap 返回之后，我们希望应用程序控制流从 ecall 的下一条指令开始执行。
            // 因此我们只需修改 Trap 上下文里面的 sepc，让它增加 ecall 指令的码长，也即 4 字节。
            // 这样在 __restore 的时候 sepc 在恢复之后就会指向 ecall 的下一条指令，
            // 并在 sret 之后从那里开始执行
            cx.x[10] = syscall(cx.x[17], [cx.x[10], cx.x[11], cx.x[12]]) as usize;
            // x10=a0           x17=a7    x10=a0   x11=a1    x12=a2
            // 处理正常系统调用的控制逻辑
            // 用来保存系统调用返回值的 a0 寄存器也会同样发生变化。
            // 我们从 Trap 上下文取出作为 syscall ID 的 a7 和系统调用的三个参数 a0~a2 
            // 传给 syscall 函数并获取返回值
            // syscall 在 syscall 子模块中实现
        }
        Trap::Exception(Exception::StoreFault) |
        Trap::Exception(Exception::StorePageFault) => {
            // 处理应用程序出现访存错误
            println!("[kernel] PageFault in application, kernel killed it.");
            // 打印错误信息
            run_next_app();
            // 直接切换并运行下一个应用程序
        }
        Trap::Exception(Exception::IllegalInstruction) => {
            // 非法指令错误
            println!("[kernel] IllegalInstruction in application, kernel killed it.");
            run_next_app();
        }
        _ => {
            // 当遇到目前还不支持的Trap类型时，操作整个panic报错退出
            panic!("Unsupported trap {:?}, stval = {:#x}!", scause.cause(), stval);
        }
    }
    cx
    // 将传入的 cx 原样返回，因此在 __restore 的时候 a0 寄存器在调用 trap_handler 前后并没有发生变化，
    // 仍然指向分配 Trap 上下文之后的内核栈栈顶，和此时 sp 的值相同，这里的 sp <- a0 并不会有问题
}

pub use context::TrapContext;
```

#### os/src/trap/trap.S

```asm
.altmacro
# 加入.altmacro 才能正常使用 .rept 命令
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
    # .align 将 __alltraps 的地址 4 字节对齐，这是 RISC-V 特权级规范的要求
__alltraps:
	# 将Trap上下文保存在内核栈上
	# sp -> user stack, sscratch -> kernel stack
    csrrw sp, sscratch, sp
    # sp -> kernel stack, sscratch -> user stack
    # csrrw 原型是 csrrw rd, csr, rs 可以将 CSR 当前的值读到通用寄存器 rd 中，
    # 然后将通用寄存器 rs 的值写入该 CSR 。
    # 因此这里起到的是交换 sscratch 和 sp 的效果。在这一行之前 sp 指向用户栈， 
    # sscratch 指向内核栈（原因稍后说明），现在 sp 指向内核栈， sscratch 指向用户栈

    addi sp, sp, -34*8
    # 准备在内核栈上保存Trap上下文， 预先分配 34x8 字节的栈帧
    # 此时 sp -> kernel stack

    sd x1, 1*8(sp)
	# 我们在这里也不保存 sp(x2)，因为我们要基于它来找到每个寄存器应该被保存到的正确的位置
    sd x3, 3*8(sp)
	#  tp(x4) 之前说明了原因
    .set n, 5
    .rept 27
    # 类似循环
        SAVE_GP %n
        .set n, n+1
    .endr
    # 保存32 个通用寄存器的后 27个
    # 在栈分配之后，可用于保存Trap上下文的地址区间为 [sp, sp + 8x34]
    # 按照TrapContext的内存布局
    # pub struct TrapContext{
    # pub x: [usize; 32],
    # pub sstatus: Sstatus,
    # pub sepc: usize,
    # 基于内核栈的位置（sp所指地址）来从低地址到高地址分别按顺序放置 x0~x31这些通用寄存器，
    # 最后是 sstatus 和 sepc 
    # 因此通用寄存器 xn 应该被保存在地址区间 [sp + 8n, sp + 8(n+1)]
    
}

    csrr t0, sstatus
    csrr t1, sepc
    # 指令 csrr rd, rs 的功能就是将 CSR 的值读到寄存器 rd 中。
    # 这里我们不用担心 t0 和 t1 被覆盖，因为它们刚刚已经被保存了
    sd t0, 32*8(sp)
    sd t1, 33*8(sp)
    # 因为不能直接 操作 csr
    # 我们将 CSR sstatus 和 sepc 的值分别读到寄存器 t0 和 t1 中然后保存到内核栈对应的位置上

    csrr t2, sscratch
    sd t2, 2*8(sp)
    # 首先将 sscratch 的值读到寄存器 t2 并保存到内核栈上，
    # 注意： sscratch 的值是进入 Trap 之前的 sp 的值，指向用户栈。而现在的 sp 则指向内核栈

    mv a0, sp
    # sp -> a0
    # 让寄存器 a0 指向内核栈的栈指针也就是我们刚刚保存的 Trap 上下文的地址，
    # 这是由于我们接下来要调用 trap_handler 进行 Trap 处理，它的第一个参数 cx 由调用规范要从 a0 中获取。
    # 而 Trap 处理函数 trap_handler 需要 Trap 上下文的原因在于：它需要知道其中某些寄存器的值，
    # 比如在系统调用的时候应用程序传过来的 syscall ID 和对应参数。我们不能直接使用这些寄存器现在的值，
    # 因为它们可能已经被修改了，因此要去内核栈上找已经被保存下来的值
    call trap_handler
    # 跳转到用rust编写的 trap_handler 函数完成trap分发及处理
    
# RISC-V 中读写 CSR 的指令是一类能不会被打断地完成多个读写操作的指令。
# 这种不会被打断地完成多个操作的指令被称为 原子指令 (Atomic Instruction)
# RISC-V 架构中常规的数据处理和访存类指令只能操作通用寄存器而不能操作 CSR 。
# 因此，当想要对 CSR 进行操作时，需要先使用读取 CSR 的指令将 CSR 读到一个通用寄存器中，
# 而后操作该通用寄存器，最后再使用写入 CSR 的指令将该通用寄存器的值写入到 CSR 中

# trap_handler返回 会回到此处
__restore:
	# 从保存的内核栈上的trap上下文恢复寄存器
    mv sp, a0
    # sp -> kernel stack, sscratch -> user stack
    
    ld t0, 32*8(sp)
    ld t1, 33*8(sp)
    ld t2, 2*8(sp)
    csrw sstatus, t0
    csrw sepc, t1
    csrw sscratch, t2
    # CSR寄存器不能直接操作，要通过通用寄存器作为媒介
    # 先恢复CSR， 再恢复通用寄存器， 这样上面上个临时寄存器才能被正确恢复

    ld x1, 1*8(sp)
    ld x3, 3*8(sp)
    .set n, 5
    .rept 27
        LOAD_GP %n
        .set n, n+1
    .endr
	# sp -> kernel stack(这是保存内核上下文之后的栈， 即 sp + 各个寄存器的空间), 
	# sscratch -> user stack
    addi sp, sp, 34*8
    # 在内核栈上回收Trap上下文所占用的内存
    # sp -> kernel stack(trap之前的内核栈)

    csrrw sp, sscratch, sp
    # 再次交换sscratch 和 sp 
    # sp -> 用户栈栈顶， sscratch -> 保存进入Trap之前的状态并指向内核栈栈顶
    sret
    # 执行 sret 从 S 特权级切换到 U 特权级
    # 回到应用程序执行
    
# sscratch CSR 的用途

# 在特权级切换的时候，我们需要将 Trap 上下文保存在内核栈上，因此需要一个寄存器暂存内核栈地址，
# 并以它作为基地址指针来依次保存 Trap 上下文的内容。但是所有的通用寄存器都不能够用作基地址指针，
# 因为它们都需要被保存，如果覆盖掉它们，就会影响后续应用控制流的执行。

# 事实上我们缺少了一个重要的中转寄存器，而 sscratch CSR 正是为此而生。从上面的汇编代码中可以看出，
# 在保存 Trap 上下文的时候，它起到了两个作用：首先是保存了内核栈的地址，
# 其次它可作为一个中转站让 sp （目前指向的用户栈的地址）的值可以暂时保存在 sscratch 。
# 这样仅需一条 csrrw  sp, sscratch, sp 指令（交换对 sp 和 sscratch 两个寄存器内容）
# 就完成了从用户栈到内核栈的切换，这是一种极其精巧的实现。
```

#### os/src/syscall/mod.rs

```rust
const SYSCALL_WRITE: usize = 64;
const SYSCALL_EXIT: usize = 93;

mod fs;
mod process;

use fs::*;
use process::*;

// syscall 函数并不会实际处理系统调用，
// 而只是根据 syscall ID 分发到具体的处理函数
pub fn syscall(syscall_id: usize, args: [usize; 3]) -> isize{
    match syscall_id {
        SYSCALL_WRITE => sys_write(args[0], args[1] as *const u8, args[2]),
        // 这里 只是将传进来的参数 args 转化成能够被具体的系统调用处理函数接受的类型
        SYSCALL_EXIT => sys_exit(args[0] as i32),
        _ => panic!("Unsupported syscall_id: {}", syscall_id),
    }
}
```

#### os/src/syscall/fs.rs

```rust
const FD_STDOUT: usize = 1;

pub fn sys_write(fd: usize, buf: *const u8, len: usize) -> isize{
    match fd{
        FD_STDOUT => {
            let slice = unsafe{ core::slice::from_raw_parts(buf, len)};
            let str = core::str::from_utf8(slice).unwrap();
            print!("{}", str);
            len as isize
            // sys_write 我们将传入的位于应用程序内的缓冲区的开始地址和长度转化为一个字符串 &str ，
            // 然后使用批处理操作系统已经实现的 print! 宏打印出来。注意这里我们并没有检查传入参数的安全性，
            // 即使会在出错严重的时候 panic，还是会存在安全隐患。这里我们出于实现方便暂且不做修补
        },
        _ => {
            panic!("Unsupported fd in sys_write!");
        }
    }
}
```

#### os/src/syscall/process.rs

```rust
use crate::batch::run_next_app;

pub fn sys_exit(xstate: i32) -> !{
    println!("[kernel Application exited with code {}", xstate);
    run_next_app()
    // sys_exit 打印退出的应用程序的返回值并同样调用 run_next_app 切换到下一个应用程序
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
	@cargo build --release
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



### user 文件夹下

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

```Ld
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
# target/riscv64gc-unknown-none-elf
APPS := $(wildcard $(APP_DIR)/*.rs)
# src/bin/*.rs
ELFS := $(patsubst $(APP_DIR)/%.rs, $(TARGET_DIR)/%, $(APPS))
# src/bin/*.rs	target/riscv64gc-unknown-none-elf/*.rs
BINS := $(patsubst $(APP_DIR)/%.rs, $(TARGET_DIR)/%.bin, $(APPS))
# src/bin/*.rs	target/riscv64gc-unknown-none-elf/*.bin

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

1. 进入entr.asm，设置函数栈，跳转到 rust_main
2. 清零除.bss.stack以外的.bss段
3. 调用println! -> print -> Stdout.write_fmt
4. 调用 painc! -> println! -> shutdown -> sbi_call(SBI_SHUTDOWN, 0, 0, 0)



## 执行流程

1. os/src/main.rs

    - 导入 entry.asm
- 设置函数栈， 跳转到rust_main
    - 导入 link_app.S
        - 这里全是数据段， 可执行文件在部分数据段中
    - 调用 clear_bass 函数， 清除 除 .bss.stack 以外的 .bss 段
    - 调用 println! 打印 [kernel] Hello, world! , 此时处于内核态
    - 调用 trap::init
    
2. os/src/trap/mod.rs

    -  我们需要修改stvec 寄存器来指向正确的Trap处理入口点
         将stvec设置为Direct模式指向它的地址

        **注意：由于前面有导入 trap.S， 所有这里知道 Trap的处理入口**

        **即 __alltraps**

3. os/src/main.rs

    - 调用batch::init

4. os/src/batch.rs

    - lazy_static! 宏提供了全局变量的运行时初始化功能，

        生成一个 AppManager 的实例 APP_MANAGER

        找到 link_app.S 中提供的符号 _num_app
        并从这里开始解析出应用数量以及各个应用的起始地址

    - 调用 print_app_info 函数

    - prin_app_info 函数调用 APP_MANAGER 实例的 print_app_info 函数

        打印出 任务数 和 每个任务的地址区间

5. os/src/main.rs

    - 调用 batch::run_next_app

6. os/src/batch.rs

    - 对全局变量 APP_MANAGER 进行操作

    - 将全局变量的 APP_MANAGER 的所有权 借给一个 可变变量

    - 调用实例的 get_current_app 获取当前是 第几个 任务

    - 调用实例的 load_app加载当前任务

    - load_app 的逻辑

        - 如果 当前任务的编号 大于 总的任务数，直接 panic ， 而 panic 中会调用 shutdown， 所以会直接关机

        - 如果没超过任务数，打印当前任务编号

        - 调用汇编指令 fence.i 来清理 i-cache

        - 将加载应用程序的地址 [0x80400000 ~ 0x80400000 + 0x20000] 用0填充

        - 从 AppManger 的实例中获取当前任务的 地址区间

            此时的地址为 跟着内核加载到的 .data 段中的地址

        - 生成待放置应用的 地址区间 ，即 0x80400000 ~ 0x8040000 + len

        - 调用 copy_from_slice , 将应用复制过去

    - 调用 AppManager 实例的 move_to_next_app 函数

        - 就是将当前任务的编号 + 1

    - 调用drop函数， 即析构函数， 销毁这个可变变量

    - 获取 __restore 的地址， \_\_restore 在 Trap.S 里

    - 将 __restore 变成一个需要 一个参数 的函数

    - 在这个文件开头定义了两个静态变量， 他们是结构体类型，结构体都是 一个 u8 数组

        初始化的参数为 0 数组，结构体的数组大小为 4096 * 2

        用于存放 trap 的上下文

    - 调用 KERNEL_STACK 这个 KernelStack 结构体的实例 的函数 push_context

        - 将获取到的 trap 上下文作为参数 传入

            返回一个trap上下文

        - 调用 TrapContext 的 app_init_context

            传入应用程序的地址 APP_BASE_ADDRESS ，即 0x80400000

            传入 用户栈 USER_STACK 的 sp，即用户栈的栈顶，

            USER_STACK 是一个用于存放  trap上下文的 u8数组, 用于存放一个用户栈

7. os/src/trap/context.rs

    - 调用TrapContext 结构的 app_init_context 方法

    - app_init_context 逻辑

        - 传入 入口地址，即应用程序的入口地址， 0x80400000

            和用户栈 的 栈顶 sp

            **注意：这个用户栈是通过USER_STACK这个实例的 get_sp 方法得到的**

            **但这个方法中 的用户栈的起始位置是不确定的，我们只能知道栈的大小**

            **所以用户栈的地址是动态随机的**

        - 获取 sstatus, 将sstatus 的 SPP字段 设为 User

        - 创建一个TrapContext 的 结构 cx， 将32 个通用寄存器的值 置0

        - 将 刚刚获取并改变SPP字段的 sstatus 寄存器 的值写入结构 cx 的sstatus中

        - 将 sepc 字段 设为 应用程序的入口地址

        - 将sp 改为 传入的应用程序的 用户栈段

        - 返回这个结构 cx， 即一个trap上下文的结构

8. os/src/batch.rs

    - 获得这个trap上下文的结构后，将其传入到 push_context 中

    - 创建一个可变的 TrapContext 结构体的指针

    - 这个指针指向的是距离内核栈栈顶一个Trap上下文大小的 地址处

    - 在这个地址处开始存放  cx 的结构

        **注意：riscv中栈是向下增长的**

        **但向栈中存放数据时是从sp处开始向上存放，我们根据要存放的数据的大小将sp向下移的，数据不会越界**

    - 相当于将 这个Trap上下文 push 到 内核栈

    - 同时返回这个在指针的 解引用 即指针所指的地址的值，即trap上下文

        的地址，就是内核栈的离栈顶一个TrapContext 结构大小的地方

    - 将返回内核栈的离栈顶一个TrapContext 结构大小的地址作为参数，传递给 __restore 函数

        <font color="red">最终，用户栈的上下文存放在内核栈中</font>

9. os/src/trap.S

    - 传入的参数 a0, 即 内核栈的离栈顶一个TrapContext 结构大小的地址

    - 将这个地址传个 sp， sp -> 内核栈， 而这个内核栈中的内容是 用户栈的上下文

    - 首先恢复sstatus, sepc 和 sscratch 寄存器

        其实也不叫恢复，而是将用户栈的上下文 给 通用寄存器 和 CSR

    - 将sp+34*8，即pop内核栈，回收了内核上存放的用户栈的上下文

    - 将 sp -> user stack, 

    - sret, 跳转到用户栈，执行用户程序

10. user/lib.rs

    再转到用户程序的逻辑里

    rust 用cargo生成

    - --bin 项目只会 有一个可执行程序生成 就main.rs
    - --lib 项目，只要一个文件中有且仅有一个 main 函数，这个文件就可以生成对应的可执行程序

    在这个user项目下，每个应用程序都导入了外部链接，那外部的整个程序都会被编译在一起

    由于我们有linker.ld文件，而且将 _start 函数分配在 .text.entry 段中，所以每个可执行文件中还是会先运行 \_start 函数，在 \_start 函数中跳转到 main 函数中

    - 执行 \_start 函数

        - 调用 clear_bss 函数，注意，此处没有 .bss.stack 段

            所以直接将整个 .bss 段清零

    - 先执行 main 函数

        - 就一个简单的打印 hello world, 返回0

    - 将main函数的返回值作为参数 传入到 exit函数中

    - exit 调用 sys_exit

11. user/syscall.rs

     - sys_exit 调用 syscall
     - syscall 将 SYSCALL_EXIT 和 main函数返回值0 作为参数 进行 trap 陷入，即调用 ecall

12. os/src/trap/trap.S

     由于 trap::init 将 stvec 指向了 __alltraps, 那进入 _\_alltraps 中

     - 将当前这个 用户上下文 存入到 内核栈上

     - 将这个内核栈的 栈指针存入到  a0 寄存器中
     - 调用 trap_handler 函数

13. os/src/trap/mod.rs

     - trap_handler 函数的逻辑

         - 将 a0 寄存器的值 即trap上下文的地址 作为参数传入

         - 直接获取当前 scause 和 stval 两个 控制寄存器的 值

             由于刚才只是保存 trap上下文， 没有改变过 scause 和 stval , 所有可以直接使用

         - 通过 scause.cause 判断 trap的原因

         - 第一个原因是系统调用

             一个由 ecall 指令触发的系统调用，在进入 Trap 的时候，
             硬件会将 sepc 设置为这条 ecall 指令所在的地址

             - sepc + 4 ，让其指向 ecall 的下一条指令 （不过这里是不会执行到 ecall的下一条指令的)
             - 根据 传入的系统调用的类型和参数值(这里保存在通用寄存器中)进行内核的系统调用

14. os/src/syscall/mod.rs

     - 根据传入的系统调用的 id 和 参数值，转到对应的系统调用的执行函数中

15. os/src/syscall/process.rs

     - 打印退出码
     - 调用 run_next_app

16. 加载第二个应用程序

     出现非法指令， 跳到 __alltraps, 保存上下文， call trap_handler

     trap_handler 发现是 StoreFault

     打印 错误原因

     跳转到下一个 应用程序

17. 加载第三个 应用程序

     正常执行，和第一个一样正常退出

18.   加载第四个

     出现非法指令， 打印错误

     跳转到下一个应用程序

19. 加载第五个

     出现非法指令， 打印错误

     跳转到下一个应用程序

20. load_app 时发现 app_id 已经大于 num_app，

     直接调用 panic ，打印后 调用 shutdown 

     正常关机

     

     





