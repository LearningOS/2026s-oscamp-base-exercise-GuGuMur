# NOTE

## 核心知识
- fd table 是进程资源命名空间：把整数 fd 映射到文件对象。
- `Arc<dyn File>` 允许统一抽象不同“文件类对象”（文件/管道/socket）。
- 复用最小可用 fd 可保持行为接近 Unix 语义。

## 拓展应用
- 可增加 `dup/dup2`、`close_on_exec`、权限位等行为。
- 在教学内核中这是连接 VFS 与进程管理的关键桥梁。
