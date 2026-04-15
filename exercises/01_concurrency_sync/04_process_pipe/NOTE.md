# NOTE

## 核心知识
- pipe 是经典 IPC 机制，父子进程通过文件描述符读写字节流。
- 管道读写是阻塞语义，EOF 通常依赖写端关闭触发。
- `Command` + `Stdio::piped` 在 Rust 中对应 shell 管道操作。

## 拓展应用
- 构建 mini-shell、命令编排器、沙箱执行器。
- 结合 `stderr` 重定向可做子进程诊断和审计日志。
