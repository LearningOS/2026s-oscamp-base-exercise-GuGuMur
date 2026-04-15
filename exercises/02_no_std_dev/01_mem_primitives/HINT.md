# HINT

1. `memcpy`/`memset`/`memmove` 都是裸指针循环，先处理 `n == 0` 的快速返回。
2. `memmove` 要区分重叠方向：`dst < src` 可前向拷贝，`dst > src` 需反向拷贝。
3. `strlen` 从起始地址向后扫描，直到 `\0`，返回不含结尾零的长度。
4. `strcmp` 逐字节比较，遇到第一个不同字节返回符号差值。
5. 常见错误：把 `memmove` 写成 `memcpy`、越界读取、空指针语义没处理好。
