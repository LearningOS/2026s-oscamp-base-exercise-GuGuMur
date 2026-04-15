# HINT

1. bump 分配器核心是一个“当前指针”，每次分配按 `layout.align()` 对齐后向前推进。
2. 并发下用 `AtomicUsize` + CAS 循环更新 `next`，失败就重试。
3. 先算 `aligned = align_up(cur, align)`，再算 `end = aligned + size`，并检查溢出/越界。
4. `dealloc` 在 bump 策略里通常是 no-op，整体回收靠 `reset`。
5. 常见错误：对齐公式写错、未检查 `end > heap_end`、CAS 成功条件写反。
