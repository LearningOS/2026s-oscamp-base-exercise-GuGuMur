# NOTE

## 核心知识
- RAII 把“资源获取与释放”绑定到对象生命周期，避免手工清理遗漏。
- guard + `Drop` 能保证异常路径（panic）也执行解锁。
- `Deref/DerefMut` 让 guard 的使用体验接近普通引用。

## 拓展应用
- Rust 标准库的 `MutexGuard/RwLockReadGuard` 都是同一设计思想。
- 可扩展为可重入锁或带调试信息的锁守卫。
