# HINT

1. `lookup` 线性扫描有效项，匹配 `(vpn, asid)` 命中即返回 `ppn` 并更新命中统计。
2. `insert` 优先复用无效槽，否则按 FIFO 指针淘汰并前移指针。
3. `flush_all/flush_by_vpn/flush_by_asid` 统一做 valid 失效处理。
4. `Mmu::translate` 先查 TLB，未命中再查页表并回填 TLB，同时更新 miss 统计。
5. 常见错误：命中后没更新统计、FIFO 指针越界处理遗漏。
