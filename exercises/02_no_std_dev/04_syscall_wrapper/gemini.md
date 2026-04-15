在深入操作系统内核开发时，**系统调用（System Call）** 是连接用户态与内核态的唯一桥梁。由于 Rust 编译器本身并不直接提供“系统调用”这一语法结构，我们必须直接与 CPU 寄存器打交道，这通常涉及内联汇编（Inline Assembly）。

### 1. 跨架构 ABI 的核心差异

不同架构在设计系统调用时，主要的考量是**性能**与**寄存器压力**：

* **x86_64**：拥有较老的 `int 0x80` 历史，但在 64 位下使用专门的 `syscall` 指令。它有一个陷阱：`syscall` 会破坏 `rcx` 和 `r11` 寄存器（用于存储返回地址和标志位），这在编写汇编时必须告知编译器。
* **AArch64 & RISC-V 64**：设计更现代，系统调用号（Syscall ID）与参数传递寄存器区分得很开，且通常不破坏额外的寄存器。它们共享了 Linux 的 `asm-generic` 调用号表。

---

### 2. 代码实现

以下是完整的 ABI 描述及内联汇编封装：

```rust
#![cfg_attr(not(test), no_std)]

use core::arch::asm;

// --- ABI 描述部分 ---

pub fn x86_64_abi() -> SyscallABI {
    SyscallABI {
        arch: "x86_64",
        instruction: "syscall",
        id_reg: "rax",
        ret_reg: "rax",
        arg_regs: &["rdi", "rsi", "rdx", "r10", "r8", "r9"],
        clobbered: &["rcx", "r11"],
        sys_write: 1,
        sys_read: 0,
        sys_close: 3,
        sys_exit: 60,
    }
}

pub fn aarch64_abi() -> SyscallABI {
    SyscallABI {
        arch: "aarch64",
        instruction: "svc #0",
        id_reg: "x8",
        ret_reg: "x0",
        arg_regs: &["x0", "x1", "x2", "x3", "x4", "x5"],
        clobbered: &[],
        sys_write: 64,
        sys_read: 63,
        sys_close: 57,
        sys_exit: 93,
    }
}

pub fn riscv64_abi() -> SyscallABI {
    SyscallABI {
        arch: "riscv64",
        instruction: "ecall",
        id_reg: "a7",
        ret_reg: "a0",
        arg_regs: &["a0", "a1", "a2", "a3", "a4", "a5"],
        clobbered: &[],
        sys_write: 64,
        sys_read: 63,
        sys_close: 57,
        sys_exit: 93,
    }
}

// --- 实际系统调用实现 ---

#[cfg(all(target_arch = "x86_64", target_os = "linux"))]
pub unsafe fn syscall3(id: usize, arg0: usize, arg1: usize, arg2: usize) -> isize {
    let ret: isize;
    asm!(
        "syscall",
        inlateout("rax") id => ret,
        in("rdi") arg0,
        in("rsi") arg1,
        in("rdx") arg2,
        out("rcx") _, // 显式标记被破坏的寄存器
        out("r11") _,
        options(nostack, preserves_flags)
    );
    ret
}

#[cfg(all(target_arch = "aarch64", target_os = "linux"))]
pub unsafe fn syscall3(id: usize, arg0: usize, arg1: usize, arg2: usize) -> isize {
    let ret: isize;
    asm!(
        "svc #0",
        in("x8") id,
        inlateout("x0") arg0 => ret,
        in("x1") arg1,
        in("x2") arg2,
        options(nostack, preserves_flags)
    );
    ret
}

// --- 高层 Wrapper ---

pub fn sys_write(fd: usize, buf: &[u8]) -> isize {
    unsafe { syscall3(NATIVE_SYS_WRITE, fd, buf.as_ptr() as usize, buf.len()) }
}

pub fn sys_read(fd: usize, buf: &mut [u8]) -> isize {
    unsafe { syscall3(NATIVE_SYS_READ, fd, buf.as_ptr() as usize, buf.len()) }
}

pub fn sys_close(fd: usize) -> isize {
    unsafe { syscall3(NATIVE_SYS_CLOSE, fd, 0, 0) }
}

pub fn sys_exit(code: i32) -> ! {
    unsafe {
        syscall3(NATIVE_SYS_EXIT, code as usize, 0, 0);
    }
    loop {} // 确保符合 ! 返回类型
}
```

---

### 3. 重点解析与调试指南

#### 1. 内联汇编的关键参数
* `inlateout("rax") id => ret`: 这是一个高性能技巧。它告诉编译器：首先将 `id` 放入 `rax`，当汇编指令执行完后，`rax` 里的新值就是 `ret`。这节省了一个寄存器分配。
* `out("rcx") _`: 这种写法叫做 **Clobber 列表**。即使我们不使用 `rcx`，也要告诉编译器它的值在 `syscall` 后已经不可信了，防止编译器在跨指令优化时出错。

#### 2. 系统调用号的魔力数字
细心的你会发现，x86_64 的 `sys_write` 是 **1**，而 ARM/RISC-V 是 **64**。
这是因为 ARM 和 RISC-V 采用了 Linux 的“新一代”通用系统调用接口（`asm-generic`），去掉了许多陈旧冗余的系统调用，对寄存器的利用也更加统一。

#### 3. 错误处理
Linux 系统调用的返回值遵循一个约定：
* **0 到 正数**：成功（例如 `write` 返回写入的字节数）。
* **-1 到 -4095**：失败，其绝对值为 `errno`（例如 `-9` 代表 `EBADF`，即错误的文描述符）。



### 思考建议
在内核开发中，如果你需要调用超过 6 个参数的系统调用（比如 `mmap`），某些架构会要求将参数打包成一个结构体并将指针传递给寄存器。这种处理方式在 OS 开发中被称为 **Argument Marshaling**。

你觉得在 Rust 中，我们该如何安全地封装这些 `unsafe` 的系统调用，使得应用层开发者不需要关心寄存器？