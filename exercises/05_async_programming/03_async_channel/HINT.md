# HINT

1. 用 `tokio::sync::mpsc::channel(cap)`，生产者 `send().await`，消费者 `recv().await`。
2. fan-in 场景下每个生产者持有一个 sender clone。
3. 让消费者退出的关键是关闭通道：所有 sender 都 drop 后 `recv` 才会返回 `None`。
4. 如果需要汇总结果，消费者循环读到 `None` 后再返回集合。
5. 常见错误：主 sender 没 drop，消费者循环无法结束。
