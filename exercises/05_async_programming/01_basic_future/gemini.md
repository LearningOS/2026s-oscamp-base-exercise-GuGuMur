你好！很高兴带你进入 Rust 异步编程的最底层。

通常我们使用 `async/await` 编写异步代码，编译器会为我们自动生成一个复杂的**状态机**。但要真正理解异步运行时（Runtime）是如何调度任务的，必须亲手实现 `Future` 接口。

### 1. 核心原理：Poll 与 Waker

在 Rust 中，`Future` 不是主动运行的，而是被动被“轮询”的。
* **`Poll::Pending`**：告诉运行时：“我现在还没好，等下再来问我。”
* **`Poll::Ready(val)`**：告诉运行时：“我搞定了，这是结果。”
* **`Waker`**：这是关键。如果你返回了 `Pending`，你必须负责在未来的某个时刻调用 `cx.waker().wake()`，否则运行时会认为你永远没准备好，从而再也不去轮询你，导致任务“饿死”。



---

### 2. 代码实现

我们将通过手动操作 `Pin` 指针和状态字段来实现这两个 Future。

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

impl Future for CountDown {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.count == 0 {
            // 如果计数归零，直接返回 Ready
            Poll::Ready("liftoff!")
        } else {
            // 否则计数减 1
            self.count -= 1;
            
            // 重要：通知执行器（Executor）立即重新调度本任务进行下一次 poll
            // 在真实场景中，这里通常由硬件中断或 IO 事件触发
            cx.waker().wake_by_ref();
            
            Poll::Pending
        }
    }
}

impl Future for YieldOnce {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.yielded {
            // 第二次 poll：发现已经 yield 过，返回 Ready
            Poll::Ready(())
        } else {
            // 第一次 poll：设置标志位为 true
            self.yielded = true;
            
            // 触发唤醒，确保执行器会进行第二次轮询
            cx.waker().wake_by_ref();
            
            Poll::Pending
        }
    }
}
```

---

### 3. 深入解析：为什么需要 `Pin`？

你可能注意到 `poll` 方法的第一个参数是 `Pin<&mut Self>` 而不是简单的 `&mut Self`。
这是因为异步状态机经常包含**自引用指针**（即结构体内部的一个字段指向另一个字段）。
* 如果 Future 在内存中被移动（Move），这些指针就会指向无效地址。
* `Pin` 保证了 Future 一旦开始被 poll，它在内存中的位置就**固定**了，从而确保了自引用指针的安全。



---

### 4. OS 程序员视角：Waker 的本质

在 Linux 内核或裸机开发中，`Waker` 的后端通常是：
1.  **中断处理程序**：网卡收到数据包后，触发中断，中断服务例程调用 `wake()`。
2.  **定时器**：硬件时钟到达指定时间。
3.  **事件循环**：如 `epoll` 返回了可读事件。

你实现的 `CountDown` 通过 `wake_by_ref()` 实现了“自我唤醒”，这在测试中很有用，但在实际高性能异步编程中，我们应该避免不必要的 `Pending`，以减少上下文切换的开销。

你觉得在实现一个异步串口驱动（Serial Driver）时，`Waker` 应该存储在什么地方，才能保证在字符接收中断发生时能找到并唤醒对应的 Future？