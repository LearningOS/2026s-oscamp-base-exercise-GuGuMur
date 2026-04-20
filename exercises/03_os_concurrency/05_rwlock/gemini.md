你好！作为精通 OS 架构的程序员，我们知道在高性能并发系统中，**读写锁（RwLock）** 是对自旋锁的重大优化。它利用了“读取操作不改变数据”的特性，允许成百上千个线程同时进入临界区，从而极大提升了系统的吞吐量。

### 1. 深度解析：写者优先（Writer-Priority）策略

在并发编程中，最怕的就是**饥饿（Starvation）**。
* **读者优先**：如果读者源源不断地进来，写者会因为始终无法获得排他权而永远等待。
* **写者优先**：一旦有写者在排队（`WRITER_WAITING`），**后续所有新到的读者都必须在门口等着**，直到写者完成任务。这保证了数据的更新能够及时反映到系统中。



---

### 2. 代码实现

我们将使用一个 `AtomicU32` 来管理复杂的状态：
* **Low 30 bits**: 读者的数量。
* **Bit 30**: `WRITER_HOLDING`（写者正在操作）。
* **Bit 31**: `WRITER_WAITING`（写者正在排队，拦截新读者）。

```rust
impl<T> RwLock<T> {
    /// 获取读锁
    pub fn read(&self) -> RwLockReadGuard<'_, T> {
        loop {
            let s = self.state.load(Ordering::Acquire);
            
            // 写者优先逻辑：如果有写者持有锁，或者有写者正在等待，读者就不能进
            if (s & (WRITER_HOLDING | WRITER_WAITING)) != 0 {
                std::hint::spin_loop();
                continue;
            }

            // 检查读者计数是否溢出
            let reader_count = s & READER_MASK;
            if reader_count == READER_MASK {
                std::hint::spin_loop();
                continue;
            }

            // 尝试增加读者计数
            if self.state.compare_exchange_weak(
                s, s + 1, 
                Ordering::AcqRel, 
                Ordering::Acquire
            ).is_ok() {
                return RwLockReadGuard { lock: self };
            }
        }
    }

    /// 获取写锁（写者优先）
    pub fn write(&self) -> RwLockWriteGuard<'_, T> {
        // 1. 先插旗：告诉所有人“有个写者在等了”，拦截后续读者
        self.state.fetch_or(WRITER_WAITING, Ordering::Release);

        loop {
            let s = self.state.load(Ordering::Acquire);
            
            // 2. 检查是否还有读者在里面，或者其他写者持锁
            if (s & READER_MASK) != 0 || (s & WRITER_HOLDING) != 0 {
                std::hint::spin_loop();
                continue;
            }

            // 3. 尝试抢锁：
            // 我们需要把 WRITER_HOLDING 置位，并清除 WRITER_WAITING（因为我们已经进去了）
            // 注意：此时可能有多个写者在竞争这一个位
            let old_waiting_state = s | WRITER_WAITING;
            let new_holding_state = WRITER_HOLDING; // 清除所有读者位和等待位，只留持有位

            if self.state.compare_exchange_weak(
                s, new_holding_state,
                Ordering::AcqRel,
                Ordering::Acquire
            ).is_ok() {
                return RwLockWriteGuard { lock: self };
            }
        }
    }
}

// --- Guard 实现 ---

impl<T> Deref for RwLockReadGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

impl<T> Drop for RwLockReadGuard<'_, T> {
    fn drop(&mut self) {
        // 读者离开，计数减一
        self.lock.state.fetch_sub(1, Ordering::Release);
    }
}

impl<T> Deref for RwLockWriteGuard<'_, T> {
    type Target = T;
    fn deref(&self) -> &T {
        unsafe { &*self.lock.data.get() }
    }
}

impl<T> DerefMut for RwLockWriteGuard<'_, T> {
    fn deref_mut(&mut self) -> &mut T {
        unsafe { &mut *self.lock.data.get() }
    }
}

impl<T> Drop for RwLockWriteGuard<'_, T> {
    fn drop(&mut self) {
        // 写者离开，清除 HOLDING 和 WAITING 位
        self.lock.state.fetch_and(!(WRITER_HOLDING | WRITER_WAITING), Ordering::Release);
    }
}
```

---

### 3. 顶级程序员的避坑指南

#### 1. 内存排序的艺术
在写锁的 `fetch_or(WRITER_WAITING, Ordering::Release)` 中，我们使用了 `Release`。这确保了在写者宣布“我要排队”之前，所有的内存准备工作都已经完成。而读者在 `load(Ordering::Acquire)` 看到这个标志后，会产生一个同步屏障，保证了不会看到过时的数据。

#### 2. CAS 循环中的状态竞争
在 `write` 方法的 `compare_exchange` 中，我们要非常小心。如果多个写者同时在等待，它们都会试图将状态改为 `WRITER_HOLDING`。成功的那个写者会清除 `WRITER_WAITING` 标志（如果它是最后一个写者的话）。但在本练习的简化模型中，我们直接设为 `WRITER_HOLDING` 是因为只要有一个写者拿到了锁，其他的写者依然会在循环中看到 `WRITER_HOLDING` 从而继续自旋。

#### 3. 活锁风险
在高冲突环境下，如果写者极其频繁，读者可能会饿死。虽然这是“写者优先”的本意，但在高性能生产环境下，我们通常会引入**时间戳**或**队列**来实现完全的“公平读写锁”。

### 专家思考建议
在目前的 `write` 实现中，如果两个写者同时调用 `write`，它们都会设置 `WRITER_WAITING` 位。当第一个写者释放锁时，它是通过 `fetch_and(!WRITER_WAITING)` 一次性清除了所有等待标志。

这会导致一个微妙的问题：**第二个写者虽然还在循环等待，但它的“优先拦截权”消失了。** 此时新来的读者可能会趁虚而入。你认为该如何修改 `WRITER_WAITING` 的存储方式（例如改用计数器），来解决这个多写者并存时的拦截失效问题？