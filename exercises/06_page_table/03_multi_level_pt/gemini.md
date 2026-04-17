你好！欢迎挑战 RISC-V 体系结构中最迷人也最复杂的机制之一：**SV39 三级页表**。

在 64 位系统中，我们无法像 32 位系统那样使用单级页表（否则页表本身将占据数 GB 的空间）。SV39 采用了一种**多级树状结构**。你可以把它想象成一个字典：VPN[2] 是目录，VPN[1] 是章节，VPN[0] 是具体的页。这种按需分配（On-demand）的特性极大节省了内存。



---

### 1. 核心逻辑实现

我们需要实现页表遍历（Page Table Walk）的三个核心操作：拆分虚拟地址、建立映射、地址翻译。

```rust
impl Sv39PageTable {
    /// 从 39 位虚拟地址中提取指定级别的 VPN 索引 (9 bits)
    pub fn extract_vpn(va: u64, level: usize) -> usize {
        // 每个 VPN 占据 9 位，offset 占据 12 位
        // level 0 从第 12 位开始，level 1 从 21 位开始，level 2 从 30 位开始
        ((va >> (12 + level * 9)) & 0x1FF) as usize
    }

    /// 建立 4KB 页映射
    pub fn map_page(&mut self, va: u64, pa: u64, flags: u64) {
        let mut curr_ppn = self.root_ppn;
        
        // 遍历前两级页表 (Level 2 和 Level 1)
        for level in (1..=2).rev() {
            let vpn = Self::extract_vpn(va, level);
            let node = self.nodes.get_mut(&curr_ppn).unwrap();
            let pte = node.entries[vpn];
            
            if (pte & PTE_V) == 0 {
                // 如果当前 PTE 无效，创建一个新的中间页表节点
                let new_ppn = self.alloc_node();
                // 写入新的 PPN 且只设 V 标志（中间节点不设 R/W/X）
                node.entries[vpn] = (new_ppn << PPN_SHIFT) | PTE_V;
                curr_ppn = new_ppn;
            } else {
                // 如果已存在，获取该 PTE 指向的下一个节点的 PPN
                curr_ppn = pte >> PPN_SHIFT;
            }
        }

        // 最后在 Level 0 写入物理页号和用户权限标志
        let vpn0 = Self::extract_vpn(va, 0);
        let node0 = self.nodes.get_mut(&curr_ppn).unwrap();
        node0.entries[vpn0] = ((pa >> 12) << PPN_SHIFT) | flags | PTE_V;
    }

    /// 地址翻译（页表走读）
    pub fn translate(&self, va: u64) -> TranslateResult {
        let mut curr_ppn = self.root_ppn;
        let offset = va & 0xFFF;

        for level in (0..=2).rev() {
            let vpn = Self::extract_vpn(va, level);
            let node = match self.nodes.get(&curr_ppn) {
                Some(n) => n,
                None => return TranslateResult::PageFault,
            };
            
            let pte = node.entries[vpn];
            if (pte & PTE_V) == 0 {
                return TranslateResult::PageFault;
            }

            // 判断是否为叶子节点：R/W/X 中任意一个为 1
            if (pte & (PTE_R | PTE_W | PTE_X)) != 0 {
                // 找到了映射！计算物理地址
                let base_ppn = pte >> PPN_SHIFT;
                
                if level > 0 {
                    // 处理大页情况：如果是在 level 1 或 2 发现的叶子
                    // 此时物理地址需要包含低级别的偏移
                    let mask = (1 << (12 + level * 9)) - 1;
                    return TranslateResult::Ok((base_ppn << 12) | (va & mask));
                } else {
                    // 普通 4KB 页
                    return TranslateResult::Ok((base_ppn << 12) | offset);
                }
            }

            // 还没到叶子，进入下一级
            curr_ppn = pte >> PPN_SHIFT;
        }

        TranslateResult::PageFault
    }

    /// 建立 2MB 大页映射
    pub fn map_superpage(&mut self, va: u64, pa: u64, flags: u64) {
        let mut curr_ppn = self.root_ppn;
        
        // 1. 只处理 Level 2 (找到 Level 1 的起始位置)
        let vpn2 = Self::extract_vpn(va, 2);
        let node2 = self.nodes.get_mut(&curr_ppn).unwrap();
        
        if (node2.entries[vpn2] & PTE_V) == 0 {
            curr_ppn = self.alloc_node();
            node2.entries[vpn2] = (curr_ppn << PPN_SHIFT) | PTE_V;
        } else {
            curr_ppn = node2.entries[vpn2] >> PPN_SHIFT;
        }

        // 2. 在 Level 1 写入叶子 PTE (R/W/X 至少一个为 1)
        let vpn1 = Self::extract_vpn(va, 1);
        let node1 = self.nodes.get_mut(&curr_ppn).unwrap();
        node1.entries[vpn1] = ((pa >> 12) << PPN_SHIFT) | flags | PTE_V;
    }
}
```

---

### 2. 顶级程序员的深度笔记

#### 关于“大页（Superpage）”映射
在 `map_superpage` 中，我们在 Level 1 写入了带有 `R/W/X` 标志的 PTE。当 CPU 硬件或我们的 `translate` 函数看到这一级就有权限标志时，它会**停止向下查找**。
* **优势**：减少了一次访存，且在大块连续内存（如内核代码、显存）中使用大页可以极大地提高 **TLB（地址翻译缓存）** 的命中率。

#### 为什么是 9 位？
每个页表节点的大小是 4KB，而每个页表项（PTE）是 8 字节（64 位）。
$$4096 / 8 = 512 = 2^9$$
这就是为什么每一级 VPN 刚好拆分成 9 位的原因——它完美契合了一个物理页能容纳的条目数。

#### 影子页表与 HashMap 模拟
在本练习中，我们用 `HashMap` 模拟了物理内存。但在真实的内核开发中，你会直接操作物理内存地址：
1.  分配一个 4KB 的物理页帧。
2.  将该页帧清零。
3.  将其物理页号（PPN）写入上级页表的 PTE 中。

### 总结
你现在实现的 `translate` 函数，在逻辑上就是 CPU 硬件中 **MMU（内存管理单元）** 内部逻辑的软件复现。如果你能理解三级页表的这种递归/树状搜索，你就已经掌握了现代操作系统内存管理的精髓。

既然我们支持了 4KB 页和 2MB 大页，你觉得在 SV39 架构中，最大的映射单元可以有多大？（提示：在 Level 2 就设为叶子节点会发生什么？）