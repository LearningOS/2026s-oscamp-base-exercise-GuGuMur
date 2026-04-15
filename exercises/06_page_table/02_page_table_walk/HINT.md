# HINT

1. 先实现地址辅助函数：`va -> vpn`、`va -> offset`、`ppn+offset -> pa`。
2. `map/unmap/lookup` 先保证边界检查，再对 `entries[vpn]` 读写。
3. `translate` 流程：取 vpn -> 查表 -> 检查 valid/权限 -> 拼接物理地址。
4. 失败路径区分清楚：无映射（page fault）与权限不足（permission fault）。
5. 常见错误：offset mask 写错、写访问未检查 W 位。
