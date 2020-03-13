# Lab5 实验报告

## 练习1

首先迁移了前4个lab的代码，并根据帮助注释作出了相应改动，再根据帮助注释完成了这个练习的代码。

Q: 当创建一个用户态进程并加载了应用程序后，CPU 是如何让这个应用程序最终在用户态执行起来的。即这个用户态进程被 ucore 选择占用 CPU 执行（RUNNING 态）到具体执行应用程序第一条指令的整个经过。

按加载应用程序的函数调用路径原路返回，执行中断返回指令`iret`（位于`trapentry.S`的最后一句）后，将切换到用户进程的第一条语句位置`_start`处开始执行。

## 练习2

Q: 如何设计实现“Copy on Write 机制”？

不再直接使用`memcpy`拷贝页，而是直接将页表指向原先的页，同时维护物理页的引用计数。对于读操作，可以直接读；对于写操作，如果物理页的引用计数不为1，则先复制出一个新的物理页来写，如果物理页的引用计数为1，则可以直接写。

## 练习3

Q: fork/exec/wait/exit 在实现中是如何影响进程的执行状态的？

- fork：创建当前进程的一个副本子进程，它们的执行上下文、代码、数据都一样。
- exec：将CPU切换到一个进程来运行。
- wait：完成进程的最后回收工作。
- exit：退出进程。

Q: 请给出 ucore 中一个用户态进程的执行状态生命周期图（包括执行状态，执行状态之间的变换关系，以及产生变换的事件或函数调用）。（字符方式画即可）

此题在 ucore 源码中已有很好的图示，我就不献丑了。
```c++
// kern/process/proc.c
/*
process state       :     meaning               -- reason
    PROC_UNINIT     :   uninitialized           -- alloc_proc
    PROC_SLEEPING   :   sleeping                -- try_free_pages, do_wait, do_sleep
    PROC_RUNNABLE   :   runnable(maybe running) -- proc_init, wakeup_proc, 
    PROC_ZOMBIE     :   almost dead             -- do_exit

-----------------------------
process state changing:
                                            
  alloc_proc                                 RUNNING
      +                                   +--<----<--+
      +                                   + proc_run +
      V                                   +-->---->--+ 
PROC_UNINIT -- proc_init/wakeup_proc --> PROC_RUNNABLE -- try_free_pages/do_wait/do_sleep --> PROC_SLEEPING --
                                           A      +                                                           +
                                           |      +--- do_exit --> PROC_ZOMBIE                                +
                                           +                                                                  + 
                                           -----------------------wakeup_proc----------------------------------
*/
```

## 总结

### 本实验涉及的知识点

应用进程的执行流程
- 创建
- 执行
- 退出
- 回收

### 本实验未涉及但很重要的知识点

系统调用的实现