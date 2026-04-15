# NOTE

## 核心知识
- TLB 是页表项缓存，核心指标是 hit/miss 与替换策略。
- ASID 用于区分地址空间，减少上下文切换时全量 flush。
- FIFO 易实现但不一定最优，真实系统常见更复杂策略。

## 拓展应用
- 可实现 LRU/Random 替换并对比命中率。
- 可加入多核场景下的 TLB shootdown 模拟。
