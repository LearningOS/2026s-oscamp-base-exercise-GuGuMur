你好！欢迎来到虚拟内存性能优化的核心——**TLB（Translation Lookaside Buffer）** 模拟练习。

作为一名 OS 开发者，你已经知道页表翻译（尤其是三级页表）是极其昂贵的，因为每次访存都会变成 3-4 次访存。TLB 就是为了解决这个问题而存在的硬件缓存。

### 1. 核心原理：TLB 与 ASID

* **缓存原理**：TLB 存储了虚拟页号（VPN）到物理页号（PPN）的直接映射。MMU 在翻译地址时会并行查找 TLB，如果命中（Hit），则瞬间完成翻译。
* **ASID（地址空间标识符）**：在早期的处理器中，每次切换进程（切换页表）都必须刷新整个 TLB。ASID 允许 TLB 条目带上“标签”，区分不同进程的映射。这样切换进程时，旧进程的缓存可以保留，极大提升了上下文切换的性能。
* **刷新（Flush）**：当 OS 修改了页表项时，必须手动通知硬件刷新 TLB（如 RISC-V 的 `sfence.vma`），否则 CPU 可能会继续使用旧的、错误的地址映射。



---

### 2. 习题实现

我们将实现 TLB 的查找、FIFO 淘汰策略以及 MMU 的二级翻译流程。

```rust
impl Tlb {
    /// 查找 TLB
    pub fn lookup(&mut self, vpn: u64, asid: u16) -> Option<u64> {
        for entry in &self.entries {
            // 必须有效，且 VPN 和 ASID 都要对上
            if entry.valid && entry.vpn == vpn && entry.asid == asid {
                self.stats.hits += 1;
                return Some(entry.ppn);
            }
        }
        self.stats.misses += 1;
        None
    }

    /// 插入 TLB (FIFO 策略)
    pub fn insert(&mut self, vpn: u64, ppn: u64, asid: u16, flags: u64) {
        // 1. 先查重：如果已经存在相同的有效条目，直接更新它（不移动指针）
        for entry in &mut self.entries {
            if entry.valid && entry.vpn == vpn && entry.asid == asid {
                entry.ppn = ppn;
                entry.flags = flags;
                return;
            }
        }

        // 2. 否则，在 FIFO 指针位置写入新条目
        self.entries[self.fifo_ptr] = TlbEntry {
            valid: true,
            asid,
            vpn,
            ppn,
            flags,
        };

        // 3. 推进 FIFO 指针
        self.fifo_ptr = (self.fifo_ptr + 1) % self.capacity;
    }

    pub fn flush_all(&mut self) {
        for entry in &mut self.entries {
            entry.valid = false;
        }
    }

    pub fn flush_by_vpn(&mut self, vpn: u64) {
        for entry in &mut self.entries {
            if entry.vpn == vpn {
                entry.valid = false;
            }
        }
    }

    pub fn flush_by_asid(&mut self, asid: u16) {
        for entry in &mut self.entries {
            if entry.asid == asid {
                entry.valid = false;
            }
        }
    }

    pub fn valid_count(&self) -> usize {
        self.entries.iter().filter(|e| e.valid).count()
    }
}

impl Mmu {
    /// 模拟 MMU 地址翻译流程
    pub fn translate(&mut self, vpn: u64) -> Option<u64> {
        // Step 1: 尝试从 TLB 获取
        if let Some(ppn) = self.tlb.lookup(vpn, self.current_asid) {
            return Some(ppn);
        }

        // Step 2: TLB Miss, 遍历页表 (这里简化为线性查找)
        let mapping = self.page_table.iter()
            .find(|(asid, map)| *asid == self.current_asid && map.vpn == vpn);

        if let Some((_, map)) = mapping {
            // Step 3: 页表命中，回填（Refill）到 TLB
            self.tlb.insert(map.vpn, map.ppn, self.current_asid, map.flags);
            Some(map.ppn)
        } else {
            // Step 4: 页表也未命中 -> Page Fault
            None
        }
    }
}
```

---

### 3. 顶级程序员的性能笔记

#### 1. 抖动（Thrashing）与容量
在测试用例 `test_mmu_thrashing` 中，你会看到如果 TLB 容量只有 2，但循环访问 3 个页面，命中率会跌至 0%。在真实的内核开发中，我们会根据工作集（Working Set）的大小来调整页表结构（例如使用大页来减少对 TLB 槽位的占用）。

#### 2. FIFO vs LRU
虽然我们这里用了简单的 **FIFO**（先进先出），但真实的硬件往往使用 **Pseudo-LRU**（伪最久未使用）。FIFO 有一个著名的缺陷叫 **Belady 异常**：增加缓存大小反而可能导致命中率下降。

#### 3. 软件回填 vs 硬件回填
* **x86/RISC-V**：硬件 MMU 会自动走页表（Page Table Walk）并回填 TLB。OS 程序员只需维护好内存里的页表。
* **MIPS/早期 LoongArch**：TLB Miss 会触发异常，由 OS 软件通过汇编手动查找页表并执行写入 TLB 的指令。这种方式灵活性极高，但性能压力很大。

### 总结
你现在实现的 `Mmu::translate` 逻辑完美刻画了计算机系统处理虚拟内存的每一微秒。**缓存的思想**在 OS 中无处不在：从 TLB 到页表缓存，再到文件系统的 Buffer Cache。

既然 TLB 极大地加速了翻译，你觉得如果两个进程共享同一块物理内存（共享内存），它们的 TLB 条目中的 `ppn` 是一样的吗？`asid` 呢？