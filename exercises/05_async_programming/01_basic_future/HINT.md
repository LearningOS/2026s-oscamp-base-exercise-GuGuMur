# HINT

1. 每个 Future 都是状态机：根据当前状态返回 `Poll::Pending` 或 `Poll::Ready`。
2. 返回 `Pending` 前要安排下一次唤醒（`cx.waker().wake_by_ref()`）。
3. 对 `Pin<&mut Self>` 只做不移动字段的修改，必要时用投影。
4. `YieldOnce` 一般第一次 Pending、第二次 Ready；`CountDown` 每 poll 递减。
5. 常见错误：Pending 不唤醒导致任务“永远挂起”。
