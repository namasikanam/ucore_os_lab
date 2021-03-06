# Lab3 实验报告

## 练习1

依据帮助注释，我首先使用`get_pte`函数得到一个`pte`，如果对应物理页不存在，则通过`pgdir_alloc_page`分配一个新的物理页、在逻辑地址和物理地址之间建立映射并刷新TLB。

我最初的实现没有做足够的鲁棒性处理，对于`get_pte`和`pgdir_alloc_page`并没有做其返回指针是否为空的检查，在学习了参考代码之后，我加入了对于空指针的检查。


Q1: 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中组成部分对 ucore 实现页替换算法的潜在用处。

PDE 和 PTE 的标志位可为页替换算法提供所需的信息。比如依据 `PTE_D` 判断换出页面是否需要写回；clock 算法访问并修改`PTE_A`。

Q2: 如果 ucore 的缺页服务例程在执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

根据[Stack Overflow的这篇问答](https://stackoverflow.com/questions/55963187/can-a-page-fault-handler-generate-more-page-faults)和[ucore关于页面置换机制的文档](https://learningos.github.io/ucore_os_webdocs/lab3/lab3_5_2_page_swapping_principles.html)，操作系统不会将内核内存换出，缺页服务例程、页表和硬盘读写例程都会保持在内存中，在缺页服务例程执行过程中不会出现页访问异常。

## 练习2

1. 按帮助注释实现缺页例程中将页换入的部分。
2. 按帮助注释理解了PRA之后，完成了`_fifo_map_swappable`和`_fifo_swap_out_victim`的核心代码。

在与参考实现比较后，我认为参考实现有以下几点不足：
1. 缺页例程的参考实现并没有做足够的鲁棒性考量，未对`page_insert`和`swap_map_swappable`的返回值不为0的情况作处理。
2. 在将页面换出后，`pra_vaddr`已无意义，应当将其清零。
3. `_fifo_map_swappable`的最后一个参数`swap_in`是冗余设计，其实并未被使用到。

Q: 如果要在 ucore 上实现"extended clock 页替换算法"请给你的设计方案，现有的 swap_manager 框架是否足以支持在 ucore 中实现此算法？如果是，请给你的设计方案。如果不是，请给出你的新的扩展和基此扩展的设计方案。并需要回答如下问题

可以支持，设计方案如下：

以一个环形结构维护所有可交换内存页，环中有一个特殊的节点`head`。类似`struct Page`，设计一个新的链表节点结构，其中有两个标志位A（访问位）和D（修改位）。
- 插入新的可交换内存页：
    - 在`head`处插入新页即可。
    - 根据VMA的状态（读/写/可执行）为新页设置标志位。
- 寻找下一个被交换的内存页：维护一个环形结构中的指针，依据指针所指的虚拟页所对应的的物理页的标志位A和D来做出下一步决策。
    - A = 0, D = 0: 选择此页交换。将此页从环中删去，将`head`移至此页原先所在的位置。
    - A = 0, D = 1: 将D清零，利用`swapfs_write`将此页写回交换分区，指针指向环中下一页。
    - A = 1, D = 0: 将A清零，指针指向环中下一页。
    - A = 1, D = 1: 将A清零，指针指向环中下一页。

Q: 需要被换出的页的特征是什么？

A = D = 0

Q: 在 ucore 中如何判断具有这样特征的页？

通过设计一个新的链表结构，在新结构中记录标志位。

Q: 何时进行换入和换出操作？

- 换入：
    - 触发缺页异常，内存中尚有闲置页面。
    - 触发缺页异常，但内存中已无闲置页面，便需将内存中某页换出之后，再将访问的物理页换入，
- 换出：触发缺页异常，但内存中已无闲置页面，便需将内存中某页换出。

## 总结

列出你认为本实验中重要的知识点，以及与对应的 OS 原理中的知识点，并简要说明你对二者的含义，关系，差异等方面的理解（也可能出现实验中的知识点没有对应的原理知识点）

### 本实验涉及的知识点

- 虚拟内存管理
- 缺页异常
- 缺页服务例程
- 页面置换算法
- 交换区
- Belady现象