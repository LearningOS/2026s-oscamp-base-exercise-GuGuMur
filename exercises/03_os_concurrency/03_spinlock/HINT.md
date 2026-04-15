# HINT

1. `lock` 用 `compare_exchange(false, true, Acquire, Relaxed)` 自旋直到成功。
2. 成功后通过 `UnsafeCell` 返回 `&mut T`，并确保同一时刻只有一个持有者。
3. `unlock` 用 `store(false, Release)` 释放锁。
4. `try_lock` 只尝试一次 CAS，失败返回 `None`。
5. 常见错误：`unlock` 用了 `Relaxed`，导致临界区写入可见性问题。
