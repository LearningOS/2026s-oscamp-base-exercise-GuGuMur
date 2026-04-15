你好！在 OS 开发中，**Bump Allocator**（也叫堆栈分配器或线性分配器）是最简单且极其高效的分配方案。它在内核初始化阶段非常有用，尤其是在复杂的多级分配器还没准备好之前。

### 1. 核心原理：内存对齐与 CAS 循环

在实现 `GlobalAlloc` 时，有两个关键挑战：

1.  **内存对齐 (Memory Alignment)**：
    CPU 通常要求某种类型的数据必须存储在特定倍数的地址上（例如 `u64` 在 64 位机器上通常需要 8 字节对齐）。
    公式：`(addr + align - 1) & !(align - 1)`
    这个技巧利用了对齐值（2 的幂）的二进制特性。

2.  **并发安全 (Concurrency)**：
    `GlobalAlloc` 的方法接收的是 `&self` 而非 `&mut self`，这意味着分配器必须是线程安全的。在不使用 `Mutex` 的情况下，我们使用 **CAS (Compare-And-Swap)** 循环来确保即使多个线程同时请求内存，指针也能被原子地、正确地移动。



---

### 2. 代码实现

这是 `BumpAllocator` 的完整 `alloc` 实现：

```rust
unsafe impl GlobalAlloc for BumpAllocator {
    unsafe fn alloc(&self, layout: Layout) -> *mut u8 {
        let mut prev_next = self.next.load(Ordering::SeqCst);

        loop {
            // 1. 获取当前指针并计算对齐后的起始地址
            let align = layout.align();
            let size = layout.size();
            
            // 使用位运算快速对齐 (要求 align 必须是 2 的幂)
            let alloc_start = (prev_next + align - 1) & !(align - 1);
            
            // 2. 检查是否溢出以及是否超出堆边界
            let alloc_end = match alloc_start.checked_add(size) {
                Some(end) if end <= self.heap_end => end,
                _ => return null_mut(), // 内存不足 (OOM) 或地址溢出
            };

            // 3. 尝试原子地更新 next 指针
            // 如果在这期间 self.next 没有被其他线程修改（依然等于 prev_next），
            // 则将其设置为 alloc_end 并返回成功。
            match self.next.compare_exchange_weak(
                prev_next,
                alloc_end,
                Ordering::SeqCst,
                Ordering::SeqCst,
            ) {
                Ok(_) => return alloc_start as *mut u8,
                Err(actual) => {
                    // 如果 CAS 失败，说明其他线程抢先分配了。
                    // 更新 prev_next 为当前最新的值，继续循环重试。
                    prev_next = actual;
                }
            }
        }
    }

    unsafe fn dealloc(&self, _ptr: *mut u8, _layout: Layout) {
        // Bump 分配器不处理单个对象的释放。
        // 只有调用 reset() 时才会回收全部内存。
    }
}
```

---

### 3. 深度解析

#### 为什么使用 `compare_exchange_weak`？
在 `loop` 中，`compare_exchange_weak` 通常比 `compare_exchange` 性能更好。在某些架构（如 ARM）上，即使值没变，CAS 也可能由于伪失效而返回错误。在循环中使用 `weak` 版本可以生成更优的汇编代码。

#### 为什么 `dealloc` 是空的？
这就是 Bump Allocator 的特性：**只增不减**。
* **优点**：速度极快，只需几次加法和一次原子操作；没有内存碎片。
* **缺点**：无法回收已释放的内存。它适用于生命周期随整个系统或某个任务结束而结束的场景。

#### 内存对齐的视觉化
假设 `next` 当前在 `10`，请求对齐为 `8`：
1. `addr + align - 1` = `10 + 8 - 1 = 17` (二进制 `10001`)
2. `!(align - 1)` = `!(7)` = `...11111000`
3. 位与操作结果为 `16`。
指针从 `10` 跳到了 `16`，中间的 6 字节成为了对齐填充（Padding）。

你觉得在什么样的情况下，这种“不回收内存”的分配器反而比复杂的链表分配器更有优势？