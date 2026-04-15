# NOTE

## 核心知识
- `Future` 是惰性状态机，只有被 poll 才推进。
- `Pending` 不代表失败，只表示“现在还没好”。
- `Waker` 是执行器回调接口，用于通知“可以再 poll 了”。

## 拓展应用
- 手写 Future 有助于理解 async fn 编译后的状态机结构。
- 可尝试实现自定义定时器 future 或多阶段协议 future。
