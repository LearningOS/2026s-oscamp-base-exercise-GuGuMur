# HINT

1. 先填写三种架构 ABI：syscall 号寄存器、参数寄存器、返回寄存器。
2. `syscall3` 用 `core::arch::asm!`，严格按目标架构寄存器约定绑定输入输出。
3. 再基于 `syscall3` 封装 `sys_write/sys_read/sys_close/sys_exit`，对应 Linux syscall 号。
4. `sys_exit` 返回类型是 `!`，调用后应不可返回，必要时加 `unreachable!()`。
5. 常见错误：寄存器填错（尤其 x86_64 的 `r10` 规则）、clobber 不完整、号表混架构。
