# HINT

1. 用 `std::sync::mpsc::channel()` 建立 `(tx, rx)`，生产者线程持有 `tx`，消费者用 `rx`。
2. 多生产者场景使用 `tx.clone()`，每个线程各持一个 sender。
3. 如果消费者要靠“通道关闭”退出循环，确保主线程把原始 `tx` 也 drop 掉。
4. 接收端可以用 `recv()` 阻塞收，或者 `for msg in rx` 迭代直到关闭。
5. 常见错误：还保留着一个 sender 导致接收端永不结束。
