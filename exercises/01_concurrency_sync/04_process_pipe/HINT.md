# HINT

1. 用 `Command::new(...).stdin(Stdio::piped()).stdout(Stdio::piped())` 创建子进程管道。
2. 向子进程写数据后要及时关闭写端（drop stdin），否则子进程可能一直等 EOF。
3. 读取输出通常用 `read_to_string` 或 `wait_with_output`，并检查退出码。
4. 注意父子进程两端的阻塞关系，避免“父等子输出、子等父输入”的死锁。
5. 常见错误：忘记 `wait`，导致僵尸进程；忘记关闭写端，导致卡死。
