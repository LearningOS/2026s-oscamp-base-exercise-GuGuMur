# HINT

1. `increment/decrement/get` 可直接用 `fetch_add/fetch_sub/load`。
2. `compare_and_swap` 使用 `compare_exchange`，把 `Ok/Err` 转成题目期望的返回格式。
3. `fetch_multiply` 用 CAS 循环：读旧值 -> 计算新值 -> 尝试交换 -> 失败重试。
4. 若题目未要求全序，通常 `Relaxed` 足够；但要和测试预期一致。
5. 常见错误：把返回值理解成“新值”而不是“旧值”。
