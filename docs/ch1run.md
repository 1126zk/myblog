# 运行指令

### 生成内核可执行文件

cargo build --release

file target/riscv64gc-unknown-none-elf/release/os

### 手动加载内核可执行文件

rust-objcopy --strip-all target/riscv64gc-unknown-none-elf/release/os -O binary target/riscv64gc-unknown-none-elf/release/os.bin

我们可以使用 `stat` 工具来比较内核可执行文件和内核镜像的大小：

- stat target/riscv64gc-unknown-none-elf/release/os
- stat target/riscv64gc-unknown-none-elf/release/os.bin

### 基于GDB验证启动流程

qemu-system-riscv64 \
-machine virt \
-nographic \
-bios ../bootloader/rustsbi-qemu.bin \
-device loader,file=target/riscv64gc-unknown-none-elf/release/os.bin,addr=0x80200000 \
-s -S

-s 可以使 Qemu 监听本地 TCP 端口 1234 等待 GDB 客户端连接，而 -S 可以使 Qemu 在收到 GDB 的请求后再开始运行。因此，Qemu 暂时没有任何输出。

打开另一个终端，启动一个 GDB 客户端连接到 Qemu ：

riscv64-unknown-elf-gdb \
-ex 'file target/riscv64gc-unknown-none-elf/release/os' \
-ex 'set arch riscv:rv64' \
-ex 'target remote localhost:1234'

b *0x80200000

c

x/10i $pc

### GBD指令

- 这里 `x/10i $pc` 的含义是从当前 PC 值的位置开始，在内存中反汇编 10 条指令
- `si` 可以让 Qemu 每次向下执行一条指令，之后屏幕会打印出待执行的下一条指令的地址
- `p/x $t0` 以 16 进制打印寄存器 `t0` 的值，注意当我们要打印寄存器的时候需要在寄存器的名字前面加上 `$`
- 我们在内核的入口点，也即地址 `0x80200000` 处打一个断点。需要注意，当需要在一个特定的地址打断点时，需要在地址前面加上 `` 。接下来通过 `c` 命令（Continue 的缩写）让 Qemu 向下运行直到遇到一个断点
- `p/d $x1` 可以以十进制打印寄存器 `x1` 的值，它的结果正确