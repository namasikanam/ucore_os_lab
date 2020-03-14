# Lab6 实验报告

## 练习1

Q: 请理解并分析`sched_class`中各个函数指针的用法，并结合 Round Robin 调度算法描述 ucore 的调度执行过程。

- `void (*init)(struct run_queue *rq)`: 初始化运行队列。
- `void (*enqueue)(struct run_queue *rq, struct proc_struct *proc)`: 把进程`proc`添加到运行队列`rq`中，调用这个函数时必须要加锁`rq_lock`。
- `void (*dequeue)(struct run_queue *rq, struct proc_struct *proc)`: 把进程`proc`从运行队列`rq`中删除，调用这个函数时必须要加锁`rq_lock`。
- `struct proc_struct *(*pick_next)(struct run_queue *rq)`: 选出下一个可以运行的进程。
- `void (*proc_tick)(struct run_queue *rq, struct proc_struct *proc)`: 响应时间中断。

---

维护当前 runnable 进程的有序运行队列。当前进程将分配给自己的时间片用完之后，调度器将当前进程放置到运行队列的尾部，再从其头部取出进程进行调度。

Q: 请在实验报告中简要说明如何设计实现“多级反馈队列调度算法”。

多级反馈队列调度算法使用多个进程队列，每个队列有一个与其它队列均不同的级别。每个队列有一个该队列约定的时间片长度，从该队列中取出的进程被分配的时间片长度即为该队列的时间片长度，队列的级别越高，其时间片长度越短。

算法的基本操作如下：

- 插入：新进程被插入至最高级队列的尾部。
- 调度：将最高级的非空队列的首进程分配给CPU。
- 删除：当进程在时间片内结束时，将其从系统中删除。
- 当进程自愿放弃CPU的控制权时（比如因为等待I/O），将其从系统中暂时删除，当进程再次RUNNABLE时，将其插入之前其存在的队列的尾部。
- 当进程用完了分给它的时间片时，便将其抢占（pre-empt）。
    - 若该进程不在最低级队列中，则将其移入下一级队列的队尾。
    - 若该进程在最低级队列中，则将其移入最低级队列的队尾。

以上译自[Wikipedia](https://en.wikipedia.org/wiki/Multilevel_feedback_queue)。

## 练习2

首先阅读了论文 Stride Scheduling: Deterministic Proportional-Share Resource Management 和文档，实现了一个基于链表的 Stride Scheduling 算法。

文档中的一些小坑：
- 文档和 ucore 的框架代码搞反了原始论文中 stride 和 pass 的含义。
- 文档和注释中遗漏了一些细节：
    - `priority`可能为0；
    - 入队时进程的`time_slice`如何处理。

代码中的一些小坑：
- 参考代码的`make grade`并不能跑到满分，而是需要删去两行`assert`。这点在piazza的[这个帖子](https://piazza.com/class/i5j09fnsl7k5x0?cid=821)和[这个帖子](https://piazza.com/class/i5j09fnsl7k5x0?cid=801)中有提及，在`lab6`中这两行`assert`已被删去，但是在`lab6_result`中并未将其删去。
- `make grade`的结果不稳定，尤其是最后`matrix`和`priority`两个点，有时会报错。

## 总结

### 本实验涉及的知识点

- 进程调度的流程
- 进程调度算法
    - Round Robin
    - Multilevel Feedback Queue
    - Stride Scheduling

### 本实验应涉及但未涉及的知识点

- 调度准则