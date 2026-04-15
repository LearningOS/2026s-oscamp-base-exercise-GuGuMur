# HINT

1. `lock()` 获取锁后返回 `SpinGuard`，把“解锁责任”绑定到 guard 生命周期。
2. 给 `SpinGuard` 实现 `Deref/DerefMut`，让它像 `&T/&mut T` 一样使用。
3. 在 `Drop` 中执行解锁 `store(false, Release)`，保证 panic 时也释放锁。
4. 关键点是生命周期：`SpinGuard<'a, T>` 不能比 `SpinLock<T>` 活得更久。
5. 常见错误：`Drop` 忘记释放锁，或返回引用时违反别名规则。
