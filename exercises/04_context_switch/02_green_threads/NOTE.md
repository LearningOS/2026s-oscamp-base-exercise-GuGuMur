# NOTE

## 核心知识
- green thread 在用户态调度，切换成本低于内核线程。
- 协作式调度依赖显式 `yield`，不会被内核抢占。
- 线程状态流转（Ready/Running/Finished）决定调度正确性。

## 拓展应用
- 可扩展为 M:N 调度器，把大量协程映射到少量内核线程。
- 与 async/await 对比可帮助理解两种并发模型差异。
