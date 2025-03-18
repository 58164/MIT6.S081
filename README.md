[README-ORIGINAL](./README)

关键词：Rv39、39位地址、56位物理地址、44位物理地址、VPN、PPN、PTE

## 主要文件和数据结构

1. [kernel/memlayout.h](kernel/memlayout.h) - 定义内存布局
2. [kernel/vm.c](kernel/vm.c) - 包含页表管理的主要代码（绝大多数和页表管理相关的代码都在这儿，建议每个函数都过一遍，做到心中有数）
3. [kernel/kalloc.c](kernel/kalloc.c) - 物理内存分配器
4. [kernel/riscv.h](kernel/riscv.h) - 定义 RISC-V 相关的硬件结构

## 页表初始化和管理流程

前提：在xv6中，使用的是 RISC-V 的 Sv39 分页方案，即虚拟地址是 64 位的，但实际上只使用了其中的 39 位（进程寻址范围是 512GB）。

所谓三级页表，其实就是三个数组。在这三个数组中，只有第一个数组的内存地址是已知的，其他的两个数组需要根据上一个数组（页表）计算得出，启用分页后：

### 1. 系统启动时的页表初始化

系统启动时， `kernel/start.c` 中的代码设置一个简单的页表，然后跳转到 main() 函数（这部分内容可以忽略，跟主线任务关系不大）。

在 `kernel/main.c` 中，系统调用 `kvminit()` 初始化内核页表。

```c
// start() jumps here in supervisor mode on all CPUs.
void main() {
    kinit();         // physical page allocator
    kvminit();       // create kernel page table
}
```

### 2. 内核页表初始化

1. 调用 kalloc() 函数申请一页物理内存（默认4kb），保存到全局变量 kernel_pagetable，它作为内核的根页表。
2. kernel_pagetable 指向的 4kb 空间，里面保存的是 512 个 PTE（page table entry 页表项），每个 PTE 的大小为 4kb * 1024 / 512 = 8 byte。
3. 用户进程的根页表，是创建进程时，通过 `uvmcreate()` 函数创建的，它保存了进程的根页表。

`kvminit()` 函数创建完内核页表以后，映射以下内容：

- 内核代码和数据
- 物理内存
- 设备寄存器

```c
void
kvminit(void)
{
  kernel_pagetable = (pagetable_t) kalloc(); // 申请一页物理内存（4096字节）并保存到全局变量 kernel_pagetable，注意，此时未开启分页，这里返回的是物理地址
  memset(kernel_pagetable, 0, PGSIZE);       // 清空内核页表

  // 把 UART 和虚拟 IO 设备，直接映射（va和pa系统）到主存，这样读写主存就等于读写设备，用于控制设备。
  kvmmap(UART0, UART0, PGSIZE, PTE_R | PTE_W);
  kvmmap(VIRTIO0, VIRTIO0, PGSIZE, PTE_R | PTE_W);
  
  // 同上，直接映射 CLINT 和 PLIC，区别是映射大小不同，CLINT 是 64KB，PLIC 是 4MB
  kvmmap(CLINT, CLINT, 0x10000, PTE_R | PTE_W);
  kvmmap(PLIC, PLIC, 0x400000, PTE_R | PTE_W);
  
  // 把内核代码设置为 只读可执行
  kvmmap(KERNBASE, KERNBASE, (uint64)etext-KERNBASE, PTE_R | PTE_X);
  
  // 内核代码和数据
  kvmmap((uint64)etext, (uint64)etext, PHYSTOP-(uint64)etext, PTE_R | PTE_W);
  
  // TRAMPOLINE
  kvmmap(TRAMPOLINE, (uint64)trampoline, PGSIZE, PTE_R | PTE_X);
}
 ```

### 3. 用户进程页表管理

每个进程有自己的页表，在 `kernel/proc.c` 中的 `allocproc` 函数分配进程时，会调用 `proc_pagetable` 创建进程的页表。

## 关键函数分析

### 页表创建和映射

1. `walk` - 在页表中查找或创建 PTE
2. `mappages` - 在页表中添加映射
3. `kvmmap` - 在内核页表中添加映射

### 内存分配和释放

1. `kalloc` - 分配一页物理内存
2. `kfree` - 释放一页物理内存

### 页表操作

1. `uvmalloc` - 为用户进程分配内存
2. `uvmdealloc` - 释放用户进程内存
3. `uvmcopy` - 复制用户内存（fork时使用）
4. `freewalk` - 递归释放页表

## 内存管理调用链

### 系统启动时的页表初始化

```c
_entry (kernel/entry.S) 
  -> start (kernel/start.c)
    -> main (kernel/main.c)
      -> kvminit (kernel/vm.c)
        -> kalloc (kernel/kalloc.c)
        -> kvmmap (kernel/vm.c)
          -> mappages (kernel/vm.c)
            -> walk (kernel/vm.c)
 ```

### 进程创建时的页表初始化

```c
fork (kernel/proc.c)
  -> allocproc (kernel/proc.c)
    -> proc_pagetable (kernel/proc.c)
      -> uvmcreate (kernel/vm.c)
        -> kalloc (kernel/kalloc.c)
      -> mappages (kernel/vm.c)
    -> uvmalloc (kernel/vm.c)
      -> kalloc (kernel/kalloc.c)
      -> mappages (kernel/vm.c)
 ```

### 进程执行时的页表切换

```c
scheduler (kernel/proc.c)
  -> swtch (kernel/swtch.S)
  -> kvminithart (kernel/vm.c) // 切换到进程页表
 ```

## 页表结构

xv6 使用 RISC-V 的 Sv39 分页方案，虚拟地址为 39 位，物理地址为 56 位。页表结构如下：

- 虚拟地址分为三部分：9位页目录索引 + 9位页表索引 + 9位页内偏移
- 每个 PTE 包含一个 44 位的物理页号（PPN）和一些标志位（权限等）

## 总结

xv6 的页表管理主要集中在 `kernel/vm.c` 文件中，通过一系列函数实现了页表的创建、映射、复制和释放等操作。系统维护了内核页表和每个进程的用户页表，通过页表切换实现了进程隔离和内存保护。