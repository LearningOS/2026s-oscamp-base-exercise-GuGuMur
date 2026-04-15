# HINT

1. `tokio::select!` 用来“谁先完成选谁”，可直接建模 race/timeout。
2. 超时模式通常是 `select! { v = fut => ..., _ = sleep(dur) => ... }`。
3. 被选中的分支执行后，其他分支 future 会被取消，注意清理语义。
4. 若函数返回 `Option/Result`，提前统一好 timeout 分支的返回值。
5. 常见错误：误把 `select!` 当成“都要执行完”。
