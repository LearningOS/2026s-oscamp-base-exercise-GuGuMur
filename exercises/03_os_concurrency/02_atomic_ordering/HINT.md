# HINT

1. 生产者先写数据，再 `ready.store(true, Release)`；消费者用 `ready.load(Acquire)` 等待。
2. 形成 Release-Acquire 配对后，消费者看到 `ready=true` 时才能安全读取数据。
3. `OnceCell::init` 用 `compare_exchange(false, true, AcqRel, Acquire)` 保证只初始化一次。
4. `get` 先检查 initialized（Acquire），再读取值。
5. 常见错误：全部用 `Relaxed` 导致“看到 ready 但看不到数据”。
