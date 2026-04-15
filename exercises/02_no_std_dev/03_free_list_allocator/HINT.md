# HINT

1. 先遍历 free list 做 first-fit，找到 `size` 足够且对齐满足的块。
2. 如果 free list 没命中，再从 bump 区域切新块，保持“先复用再扩张”。
3. `dealloc` 时把释放块包装成 `FreeBlock` 挂到链表头（LIFO 即可通过基础测试）。
4. 注意最小块大小至少容纳 `FreeBlock` 元数据，否则无法再次挂链。
5. 常见错误：链表 next 指针丢失、块大小记录错误、释放未对齐地址。
