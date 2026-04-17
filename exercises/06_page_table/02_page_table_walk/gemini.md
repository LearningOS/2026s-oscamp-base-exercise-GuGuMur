你好！完成了上一关 PTE 标志位的位运算，现在我们进入 **地址翻译（Address Translation）** 的实战阶段。

单级页表是理解分页机制的最直观模型。你可以把它看作一个巨大的**数组**，数组的下标是**虚拟页号 (VPN)**，而数组的内容则是对应的**物理页号 (PPN)** 和权限信息。

### 1. 核心流程：虚拟到物理的跃迁

地址翻译的过程可以总结为：**“拆分 -> 查表 -> 权限检查 -> 重组”**。

1.  **拆分**：将 32 位虚拟地址切开。低 12 位是页内偏移（Offset），因为页大小是 $4KB$ ($2^{12}$)；高 20 位是虚拟页号（VPN）。
2.  **查表**：用 VPN 作为索引去页表数组里找对应的页表项（PTE）。
3.  **重组**：物理地址 = PPN 对应的起始地址 + 偏移量。



---

### 2. 习题实现

我们将实现 `SingleLevelPageTable` 的核心逻辑。

```rust
impl SingleLevelPageTable {
    /// 建立 vpn -> ppn 的映射
    pub fn map(&mut self, vpn: usize, ppn: u32, flags: u8) {
        // 在对应索引处存入 Option::Some
        self.entries[vpn] = Some(PageTableEntry { ppn, flags });
    }

    /// 取消映射
    pub fn unmap(&mut self, vpn: usize) {
        self.entries[vpn] = None;
    }

    /// 查询映射
    pub fn lookup(&self, vpn: usize) -> Option<&PageTableEntry> {
        // 使用 .as_ref() 将 Option<T> 转为 Option<&T>
        self.entries.get(vpn).and_then(|opt| opt.as_ref())
    }

    /// 核心逻辑：地址翻译
    pub fn translate(&self, va: u32, is_write: bool) -> TranslateResult {
        let vpn = va_to_vpn(va);
        let offset = va_to_offset(va);

        // 1. 查找页表项
        let pte = match self.lookup(vpn) {
            Some(entry) => entry,
            None => return TranslateResult::PageFault,
        };

        // 2. 检查有效位 (V)
        if (pte.flags & PTE_VALID) == 0 {
            return TranslateResult::PageFault;
        }

        // 3. 检查写入权限
        if is_write && (pte.flags & PTE_WRITE) == 0 {
            return TranslateResult::PermissionDenied;
        }

        // 4. 组装物理地址
        TranslateResult::Ok(make_pa(pte.ppn, offset))
    }
}

/// 提取虚拟页号 (VA >> 12)
pub fn va_to_vpn(va: u32) -> usize {
    (va >> PAGE_OFFSET_BITS) as usize
}

/// 提取页内偏移 (VA & 0xFFF)
pub fn va_to_offset(va: u32) -> u32 {
    va & ((1 << PAGE_OFFSET_BITS) - 1)
}

/// 拼装物理地址 (PPN << 12 | Offset)
pub fn make_pa(ppn: u32, offset: u32) -> u32 {
    (ppn << PAGE_OFFSET_BITS) | offset
}
```

---

### 3. 顶级程序员的深度讲解

#### 1. 偏移量（Offset）为什么不参与翻译？
这是一个关键点：**虚拟地址的低 12 位和物理地址的低 12 位是完全一样的**。因为页的大小是 $4KB$，所以无论你在虚拟页的哪个位置，映射到物理页后，你相对于页起始位置的偏移量不会改变。这也是为什么我们只需要翻译 PPN 的原因。

#### 2. 内存浪费的挑战
单级页表的缺点显而易见：**太占内存**。
* 在 32 位系统下，如果有 $2^{20}$ 个页，每个 PTE 占 4 字节，那么页表本身就要占 $4MB$。
* 更糟糕的是，即使你的程序只用了 $10KB$ 内存，为了覆盖整个虚拟空间，这 $4MB$ 的页表也必须存在。
* **这就是为什么我们在现代 OS（如 RISC-V SV39）中需要多级页表的原因**。多级页表允许我们只为“已经使用了的”虚拟地址区域分配页表空间。

#### 3. 权限检查的顺序
注意在 `translate` 中，我们要先检查 `PTE_VALID`。
在硬件实现中，权限检查是并行的，但逻辑上，如果 `Valid` 位为 0，其他的标志位都是不可信的。

### 总结
你已经成功模拟了 MMU（内存管理单元）最核心的工作！

当你运行 `ls` 或 `cd` 时，CPU 的硬件逻辑就在疯狂执行你刚刚写的这些位移和查表操作。在真实的内核中，你会把这个 `SingleLevelPageTable` 存储在物理内存里，并把它的起始物理地址告诉 CPU 的一个特殊寄存器（在 RISC-V 中是 `satp`）。

既然单级页表这么浪费内存，你觉得如果换成**多级页表**，我们在查找 VPN 时，查找步骤会发生什么变化？我们需要把 VPN 拆成几段？