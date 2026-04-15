# NOTE

## 核心知识
- syscall ABI 的本质是“用户态与内核态的寄存器契约”。
- 不同架构的 syscall 号与参数寄存器顺序不同，不能混用。
- inline asm 是最小封装路径，但要严格声明输入输出和 clobber。

## 拓展应用
- 可继续封装 `open/mmap/futex` 等系统调用形成微型 libc。
- 在裸机或 unikernel 场景中，这类封装是运行时基础。
