# HINT

1. 内部结构推荐 `Vec<Option<Arc<dyn File>>>`，天然支持“空洞复用”。
2. `alloc` 先找最小空槽位，没有再 `push` 到尾部。
3. `get` 返回 `Option<Arc<dyn File>>` 时要 `clone` Arc，避免移动走表项。
4. `close` 把对应槽位设为 `None`，并返回是否成功关闭。
5. `count` 统计 `Some` 数量，不是 `Vec` 长度。
