你好！现在我们触及了操作系统内核最硬核的部分：**上下文切换（Context Switch）**。这是实现多线程、协程以及任务调度的灵魂。

在 RISC-V 架构中，上下文切换的本质就是**“换一套寄存器”**。由于我们要手动操作 `sp`（栈指针）和 `ra`（返回地址），传统的函数调用约定（Prologue/Epilogue）会干扰我们的逻辑，因此我们必须使用 Rust 的 `#[naked]` 函数。

### 1. 核心概念：被调用者保存寄存器 (Callee-saved)

根据 RISC-V 调用约定，有些寄存器是由被调用者（Callee）负责保护的。这意味着当我们从函数 A 切换到函数 B 时，必须手动保存 `s0-s11`。而 `ra` 记录了我们要跳回哪里，`sp` 记录了我们要切换到哪个栈。



---

### 2. 代码实现

我们将使用 `core::arch::naked_asm!` 来编写这段关键的汇编代码。

```rust
use core::arch::naked_asm;

impl TaskContext {
    pub fn init(&mut self, stack_top: usize, entry: usize) {
        // RISC-V 栈必须 16 字节对齐
        self.sp = (stack_top & !0xf) as u64;
        // 当 switch_context 执行 ret 时，它会跳转到 ra 指向的地址
        self.ra = entry as u64;
    }
}

/// 切换上下文：将当前寄存器存入 old，从 new 加载寄存器
/// a0 = &mut old, a1 = &new
#[unsafe(no_mangle)]
#[unsafe(naked)]
pub unsafe extern "C" fn switch_context(old: &mut TaskContext, new: &TaskContext) {
    naked_asm!(
        // 1. 保存当前上下文到 old (a0)
        "sd sp, 0(a0)",
        "sd ra, 8(a0)",
        "sd s0, 16(a0)",
        "sd s1, 24(a0)",
        "sd s2, 32(a0)",
        "sd s3, 40(a0)",
        "sd s4, 48(a0)",
        "sd s5, 56(a0)",
        "sd s6, 64(a0)",
        "sd s7, 72(a0)",
        "sd s8, 80(a0)",
        "sd s9, 88(a0)",
        "sd s10, 96(a0)",
        "sd s11, 104(a0)",

        // 2. 从 new (a1) 加载上下文
        "ld sp, 0(a1)",
        "ld ra, 8(a1)",
        "ld s0, 16(a1)",
        "ld s1, 24(a1)",
        "ld s2, 32(a1)",
        "ld s3, 40(a1)",
        "ld s4, 48(a1)",
        "ld s5, 56(a1)",
        "ld s6, 64(a1)",
        "ld s7, 72(a1)",
        "ld s8, 80(a1)",
        "ld s9, 88(a1)",
        "ld s10, 96(a1)",
        "ld s11, 104(a1)",

        // 3. 擦除 a0, a1 寄存器，防止在新的上下文中泄露旧栈指针
        "mv a0, zero",
        "mv a1, zero",

        // 4. 返回。此时 ra 已经是新任务的 entry，sp 已经是新任务的栈。
        "ret"
    );
}

pub fn alloc_stack() -> (Vec<u8>, usize) {
    let mut buf = vec![0u8; STACK_SIZE];
    let top = buf.as_ptr() as usize + STACK_SIZE;
    // 强制 16 字节对齐
    let aligned_top = top & !0xf;
    (buf, aligned_top)
}
```

---

### 3. 深度解析：为什么这么写能行？

#### `ret` 到底干了什么？
在 RISC-V 中，`ret` 伪指令实际上是 `jalr x0, 0(ra)`。
* 当我们第一次执行 `switch_context` 切换到新任务时，`ld ra, 8(a1)` 把我们通过 `init` 设置的 `entry` 地址加载到了 `ra` 寄存器。
* 执行 `ret` 时，CPU 就会跳转到这个 `entry` 地址开始执行。

#### `#[unsafe(naked)]` 的必要性
如果这是一个普通函数，编译器会在开头加上 `addi sp, sp, -64` 之类的指令来保存调用者寄存器。这会破坏我们手动控制 `sp` 的逻辑。`naked` 告诉编译器：“别动这个函数，汇编里写了什么就是什么”。

#### 栈增长方向
记住，在 RISC-V 和大多数现代架构中，**栈是向低地址增长的**。所以 `alloc_stack` 返回的是 `buf` 的末尾地址。

### 顶级程序员的思考
目前的 `switch_context` 只处理了通用寄存器。在实际的内核中（比如带有浮点运算的 OS），你还需要在上下文切换时保存和恢复 **浮点寄存器 (f0-f31)** 以及相关的状态寄存器（如 `sstatus`）。

既然你已经实现了任务切换，你觉得在发生**硬件中断**（如时钟中断）时进行的上下文保存，与这种**主动调用**（Cooperative yield）的上下文保存有什么本质区别？（提示：想想 `caller-saved` 寄存器）