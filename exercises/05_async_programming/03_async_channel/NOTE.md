# NOTE

## 核心知识
- async channel 在无数据时挂起任务而非阻塞线程。
- 有界通道提供背压，防止生产者无限制占用内存。
- sender 全部释放后，receiver 得到 `None` 作为结束信号。

## 拓展应用
- 管道化数据处理、异步日志汇聚、任务扇入扇出。
- 可对比 mpsc 和 broadcast/watch 在语义上的差异。
