# HINT

1. 先明确状态位：读者计数 + writer 持有位 + writer 等待位。
2. `read()` 在“无 writer 持有、无 writer 等待(若要求 writer 优先)”时递增读计数。
3. `write()` 先声明 writer 等待，再等待读者清零并获取写锁。
4. 读/写 guard 的 `Drop` 负责回收状态位，保持状态机闭环。
5. 常见错误：状态位更新非原子、writer 优先策略没生效导致饥饿。
