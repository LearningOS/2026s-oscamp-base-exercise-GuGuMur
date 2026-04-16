你好！在理解了底层 `Future` 的运作机制后，现在我们来看看在实际工程中是如何使用这些工具的。

在 Rust 的异步世界里，**`tokio::spawn`** 相当于传统线程中的 `thread::spawn`。它会将一个 `Future` 提交给 Tokio 的执行器（Executor），由执行器在后台的线程池中调度运行。

### 1. 核心概念：任务（Task）与并发

* **Task**：Tokio 管理的最小调度单位。它是非阻塞的，当遇到 `await` 时，它会主动让出 CPU 核心给其他 Task 使用。
* **JoinHandle**：当你 `spawn` 一个任务时，你会得到一个“句柄”。通过 `await` 这个句柄，你可以获取任务的返回值。
* **并发 (Concurrency) vs 串行 (Sequential)**：
    * **串行**：`await` 一个任务后再开始下一个，总耗时是所有任务之和。
    * **并发**：先启动（spawn）所有任务，最后统一 `await` 结果，总耗时接近于最慢的那个任务。

---

### 2. 代码实现

我们将展示如何正确地批量启动任务并收集结果。

```rust
use tokio::task::JoinHandle;
use tokio::time::{sleep, Duration};

/// 并发计算平方值
pub async fn concurrent_squares(n: usize) -> Vec<usize> {
    let mut handles = Vec::with_capacity(n);

    // 1. 批量启动任务
    for i in 0..n {
        // spawn 会立即将任务提交到后台运行
        let handle: JoinHandle<usize> = tokio::spawn(async move {
            i * i
        });
        handles.push(handle);
    }

    // 2. 按顺序等待所有结果
    let mut results = Vec::with_capacity(n);
    for handle in handles {
        // 等待任务完成，unwrap() 是为了处理任务执行过程中可能发生的 panic
        results.push(handle.await.unwrap());
    }

    results
}

/// 模拟并行的耗时任务
pub async fn parallel_sleep_tasks(n: usize, duration_ms: u64) -> Vec<usize> {
    let mut handles = Vec::with_capacity(n);

    for i in 0..n {
        let handle = tokio::spawn(async move {
            sleep(Duration::from_millis(duration_ms)).await;
            i
        });
        handles.push(handle);
    }

    let mut results = Vec::with_capacity(n);
    for handle in handles {
        results.push(handle.await.unwrap());
    }

    // 题目要求排序（尽管按顺序 await JoinHandle 通常已经是有序的）
    results.sort();
    results
}
```

---

### 3. 深度解析：调度器的魔法

当你调用 `tokio::spawn` 时，Tokio 会使用 **工作窃取（Work-Stealing）** 算法。



* **多线程协作**：Tokio 默认会在你的每个 CPU 核心上启动一个工作线程。
* **负载均衡**：如果一个线程完成了它队列里的所有任务（比如处理完了它负责的 `sleep`），它会去其他繁忙线程的队列里“偷”一些任务过来执行，从而确保所有核心都不闲置。

### 4. 常见陷阱：忘记 `spawn`

初学者经常犯的一个错误是直接写 `for` 循环然后在循环体里 `await`：
```rust
// 错误示范：这是串行的！
for i in 0..n {
    let res = some_async_work(i).await; // 必须等这个 i 完事才能跑下一个 i
}
```
这种写法不会产生并发效果。必须先通过 `spawn`（或者使用 `futures::future::join_all`）将 Future 显式交给执行器。

### 专家建议
在处理大量任务（例如上万个）时，无限制地使用 `tokio::spawn` 可能会耗尽系统资源。在真实的 OS 或后端系统中，我们通常会使用 `Semaphore`（信号量）或任务池来限制**最大并发数**。

你认为在编写一个高性能的网络爬虫时，为什么要限制并发任务的数量，而不是直接 `spawn` 所有的请求？