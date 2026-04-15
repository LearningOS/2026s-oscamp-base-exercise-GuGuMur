这一节我们进入了 Rust 系统编程的核心领域：**进程管理与管道通信**。如果说之前的线程是在同一个房间（内存空间）里协作，那么进程就是让 Rust 启动并指挥另一个独立的房间。

---.

### 1. 核心知识点：从 Rust 到内核

在 Rust 中，`std::process` 是对底层操作系统系统调用（Syscalls）的高级抽象。

| Rust 代码 | 对应底层系统调用 (Linux/Unix) | 作用 |
| :--- | :--- | :--- |
| `Command::new()` | `fork()` | 创建一个子进程（当前进程的副本） |
| `Command::args()` | `execve()` | 用新程序替换子进程的内容 |
| `Stdio::piped()` | `pipe()` | 在内核中创建一个缓冲区（管道） |
| `child.wait()` | `waitpid()` | 挂起父进程，直到子进程结束并回收资源 |



---

### 2. 练习题解思路与资源管理

#### 管道通信的“生死劫”：`drop` 的重要性
在 `pipe_through_cat` 和 `pipe_through_grep` 中，最关键的操作是 **`drop(stdin)`**。

* **原理**：管道是一个有方向的缓冲区。子进程（如 `cat` 或 `grep`）会一直读取 `stdin`，直到它接收到 **EOF (End Of File)**。
* **Rust 的魔法**：在 Rust 中，关闭管道写入端的唯一方法是销毁 `ChildStdin` 对象。如果你不 `drop` 它，子进程会以为父进程“还在组织语言”，从而永远阻塞在读取操作上，导致死锁。



#### 基础执行：`run_command`
```rust
pub fn run_command(program: &str, args: &[&str]) -> String {
    let output = Command::new(program)
        .args(args)
        .stdout(Stdio::piped()) // 必须明确要求捕获输出，否则会打印到控制台
        .output()               // 内部调用了 wait()，会阻塞直到结束
        .expect("Failed to execute command");

    String::from_utf8_lossy(&output.stdout).to_string()
}
```

#### 双向管道：`pipe_through_cat`
```rust
pub fn pipe_through_cat(input: &str) -> String {
    let mut child = Command::new("cat")
        .stdin(Stdio::piped())
        .stdout(Stdio::piped())
        .spawn() // spawn 不会等待进程结束
        .unwrap();

    // 1. 写入数据
    let mut stdin = child.stdin.take().unwrap();
    stdin.write_all(input.as_bytes()).unwrap();
    
    // 2. 必须关闭 stdin，否则 cat 不会停止读取，也就不会结束
    drop(stdin);

    // 3. 读取结果
    let mut output = String::new();
    child.stdout.take().unwrap().read_to_string(&mut output).unwrap();
    
    child.wait().unwrap(); // 良好的习惯：显式回收子进程
    output
}
```

---

### 3. 进程退出状态 (Exit Status)

每个进程结束时都会返回一个整数（0-255）。
* **`0`**：代表成功。
* **非 `0`**：代表各种各样的错误（例如 `1` 通常是通用错误，`127` 是命令未找到）。
* **注意**：在 Rust 中，`status.code()` 返回的是 `Option<i32>`，因为如果进程是被信号（Signal，如 `kill -9`）强杀的，它可能没有退出码。

---

### 4. 为什么使用 `run_command_with_result`？

在真实的生产环境下，直接 `unwrap()` 进程操作是非常危险的。
* **命令不存在**：用户没装 `grep` 怎么办？
* **权限不足**：尝试执行 `/root/secret` 怎么办？
* **无效编码**：子进程输出了非 UTF-8 的二进制乱码怎么办？

通过返回 `io::Result<String>`，你可以让调用者决定是重试、报错还是寻找替代方案。

**思考题**：如果你启动了一个计算任务进程，由于父进程逻辑错误导致没有调用 `wait()` 也没有 `drop` 子进程句柄，这个子进程结束后会变成什么状态？（提示：这种进程被称为“僵尸”）。