# HINT

1. 并发执行用 `tokio::spawn` 收集 `JoinHandle`，不要顺序 `await` 每个任务。
2. 全部任务创建后，再统一 `await` 回收结果。
3. 如果测试关心结果顺序，记得按任务索引重排。
4. 处理 `JoinHandle` 的 `Result`，区分任务 panic 与业务错误。
5. 常见错误：在循环里 `spawn` 后马上 `await`，等价于串行。
