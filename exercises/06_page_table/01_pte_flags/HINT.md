# HINT

1. 先定义好位布局：低 8 位是 flags，PPN 位于更高位（按题目常量移位）。
2. `make_pte` = `(ppn << shift) | (flags & mask)`。
3. `extract_ppn/extract_flags` 分别做右移+掩码、低位掩码。
4. `is_leaf` 检查 R/W/X 任一位；`check_permission` 先验 `V` 再按请求权限判断。
5. 常见错误：位移位数写错、忘记先检查有效位。
