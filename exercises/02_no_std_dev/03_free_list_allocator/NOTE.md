# NOTE

## 核心知识
- free-list 通过“空闲块链表”支持释放后再利用。
- first-fit 策略实现简单，但可能产生外部碎片。
- 释放块头部嵌入元数据是内核分配器常见技巧（intrusive list）。

## 拓展应用
- 可继续实现块分裂/合并（coalescing）降低碎片。
- 对比 buddy/slab 分配器在内核场景下的优劣。
