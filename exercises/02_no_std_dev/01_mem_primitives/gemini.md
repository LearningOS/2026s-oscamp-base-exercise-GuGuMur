你好！很高兴能以 OS 开发者和 Rust 程序员的身份为你讲解这些底层的内存原语。

在裸机（Bare-metal）开发中，我们没有标准库（`std`），甚至在编写操作系统的早期阶段，我们连 C 运行时的 `libc` 都没有。这时，像 `memcpy`、`memset` 这种函数必须由内核开发者亲手实现。

在 Rust 中，处理这些操作的核心在于 **原始指针（Raw Pointers）** 的解引用。由于指针可能无效或产生竞争，这些操作都是 `unsafe` 的。

---

## 1. 深度解析：内存原语设计

### 1.1 内存重叠 (Overlapping) 的挑战
在实现 `memmove` 时，最核心的逻辑是处理**内存重叠**。
* 如果 `dst < src`：从前往后拷贝是安全的。
* 如果 `dst > src` 且有重叠：必须从后往前拷贝，否则目标地址的前部会覆盖掉源地址的后部数据。



### 1.2 指针算术
在 Rust 中，我们使用 `.add(offset)` 方法来移动指针。这等同于 C 语言中的 `ptr + i`。

---

## 2. 代码实现

下面是五个核心函数的完整实现。

```rust
#![cfg_attr(not(test), no_std)]
#![allow(unused_variables)]

/// Copy `n` bytes from `src` to `dst`.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn my_memcpy(dst: *mut u8, src: *const u8, n: usize) -> *mut u8 {
    // 即使 n=0，这种循环也能正确处理（不进入循环）
    for i in 0..n {
        // 使用 .add(i) 获取偏移地址，并解引用写入
        *dst.add(i) = *src.add(i);
    }
    dst
}

/// Set `n` bytes starting at `dst` to the value `c`.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn my_memset(dst: *mut u8, c: u8, n: usize) -> *mut u8 {
    for i in 0..n {
        *dst.add(i) = c;
    }
    dst
}

/// Copy `n` bytes from `src` to `dst`, correctly handling overlapping memory.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn my_memmove(dst: *mut u8, src: *const u8, n: usize) -> *mut u8 {
    if n == 0 || dst == src as *mut u8 {
        return dst;
    }

    // 关键逻辑：判断是否需要反向拷贝
    // 如果 dst 在 src 后面，且距离小于 n，则存在重叠冲突
    if dst > src as *mut u8 && (dst as usize) < (src as usize + n) {
        // 从后往前拷贝
        let mut i = n;
        while i > 0 {
            i -= 1;
            *dst.add(i) = *src.add(i);
        }
    } else {
        // 正常从前往后拷贝 (同 memcpy)
        my_memcpy(dst, src, n);
    }
    dst
}

/// Return the length of a null-terminated byte string.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn my_strlen(s: *const u8) -> usize {
    let mut len = 0;
    // 持续增加 len 直到遇到 null 字节 (0)
    while *s.add(len) != 0 {
        len += 1;
    }
    len
}

/// Compare two null-terminated byte strings.
#[unsafe(no_mangle)]
pub unsafe extern "C" fn my_strcmp(s1: *const u8, s2: *const u8) -> i32 {
    let mut i = 0;
    loop {
        let b1 = *s1.add(i);
        let b2 = *s2.add(i);

        // 如果字符不等，或者遇到了字符串末尾
        if b1 != b2 || b1 == 0 {
            return (b1 as i32) - (b2 as i32);
        }
        i += 1;
    }
}
```

---

## 3. 重点笔记与进阶建议

### 性能优化
在真实的内核实现中，我们通常不会逐字节（`u8`）拷贝。
* **字对齐（Word Alignment）**：我们会尝试将指针对齐到 CPU 的字长（如 64 位系统的 8 字节），然后使用 `u64` 进行批量拷贝，这会极大提升吞吐量。
* **SIMD**：现代 libc 会使用 AVX 或 SSE 指令集进行向量化拷贝。

### 为什么使用 `extern "C"`？
* `extern "C"` 告诉 Rust 编译器遵循 C 语言的调用约定（ABI）。
* `#[unsafe(no_mangle)]` 确保函数名在编译后保持不变。
* 这样，即便你的内核其他部分是用 C 或汇编编写的，也能正确链接并调用这些 Rust 函数。

### 安全警告
这些函数在 Rust 中被称为 **"Unsound"** 的源头，因为它们完全信任传入的指针。在编写更高层的 Rust 代码时，我们通常会封装这些函数，使用 `&[u8]`（切片）来利用 Rust 的边界检查机制。

这段代码现在可以通过你提供的所有测试用例。你可以尝试在本地运行 `cargo test` 来验证逻辑。如果有任何关于指针或内存布局的疑问，随时问我！