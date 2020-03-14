# Lab7 实验报告

## 练习1

Q：请在实验报告中给出内核级信号量的设计描述，并说明其大致执行流程。

参考教科书*Operating Systems Internals and Design Principles*第五章《同步互斥》中对信号量实现的原理性描述：
```c++
struct semaphore {
    int count;
    queueType queue;
};
void semWait(semaphore s)
{
    s.count--;
    if (s.count < 0) {
        /* place this process in s.queue */;
        /* block this process */;
    }
}
void semSignal(semaphore s)
{
    s.count++;
    if (s.count<= 0) {
        /* remove a process P from s.queue */;
        /* place process P on ready list */;
    }
}
```

当多个（>1）进程可以进行互斥或同步合作时，一个进程会由于无法满足信号量设置的某条件而在某一位置停止，直到它接收到一个特定的信号（表明条件满足了）。为了发信号，需要使用一个称作信号量的特殊变量。为通过信号量`s`传送信号，信号量的 V 操作采用进程可执行原语`semSignal(s)`；为通过信号量`s`接收信号，信号量的 P 操作采用进程可执行原语`semWait(s)`；如果相应的信号仍然没有发送，则进程被阻塞或睡眠，直到发送完为止。

Q：请在实验报告中给出给用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。

用户态进程/线程可采用于内核态基本相同的设计方案，不同之处在于需要给信号量添加一个权限标识，并将P/V操作封装成系统调用。

## 练习2

Q: 请在实验报告中给出内核级条件变量的设计描述，并说明其大致执行流程。

关键数据结构设计如下：
```c++
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;

typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;

// Initialize variables in monitor.
void     monitor_init (monitor_t *cvp, size_t num_cv);
// Unlock one of threads waiting on the condition variable. 
void     cond_signal (condvar_t *cvp);
// Suspend calling thread on a condition variable waiting for condition atomically unlock mutex in monitor,
// and suspends calling thread on conditional variable after waking up locks mutex.
void     cond_wait (condvar_t *cvp);
```

参考*OS Concept*一书中的 6.7.3 小节“用信号量实现管程”，`cond_wait`和`cond_signal`的原理如下：
```c++
// cond_wait
++cv.count;
if (monitor.next_count > 0)
    sem_signal(monitor.next);
else
    sem_signal(monitor.mutex);
sem_wait(cv.sem);
--cv.count;

// cond_signal
if (cv.count > 0) {
    ++monitor.next_count;
    sem_signal(cv.sem);
    sem_wait(monitor.next);
    --monitor.next_count;
}
```

为了让整个管程正常运行，还需在管程中的每个函数的入口和出口增加相关操作，即：
```c++
func_t function_in_monitor (...) {
    sem.wait(monitor.mutex);
/*-----------------------------
    the real body of function;
  -----------------------------*/
    if(monitor.next_count > 0)
        sem_signal(monitor.next);
    else
        sem_signal(monitor.mutex);
}
```

这样带来的作用有两个：
- 只有一个进程在执行管程中的函数；
- 避免由于执行了`cond_signal`函数而睡眠的进程无法被唤醒。如果进程 A 由于执行了`cond_signal`函数而睡眠（这会让`monitor.next_count`大于 0，且执行`sem_wait(monitor.next)`），则其他进程在执行管程中的函数的出口，会判断`monitor.next_count`是否大于 0，如果大于 0，则执行`sem_signal(monitor.next)`，从而执行了`cond_signal`函数而睡眠的进程被唤醒。

上诉措施将使得管程正常执行。

Q：请在实验报告中给出给用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。

可采用与内核态基本相同的设计方案，不同之处在于：
- 需要为管程添加一个权限标识；
- 信号量的P/V操作需要设计成系统调用。

Q：能否不用基于信号量机制来完成条件变量？如果不能，请给出理由，如果能，请给出设计说明和具体实现。

能。

由于管程中一次最多只能有一个进程在运行，所以信号量中的`value`便是冗余的，这样我们其实可以直接基于`wait_queue`来实现条件变量。

在原有代码的基础上，做出如下修改：

1. 修改数据结构的设计。
```c++
typedef struct condvar{
    wait_queue_t sem;        // the wait queue is used to down the waiting proc, and the signaling proc should up the waiting proc. Here the identifier "sem" is reserved for the backward compatibility.
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;

typedef struct monitor{
    wait_queue_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    wait_queue_t next;       // the next wait queue is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```
2. 设计新的P/V操作。
```c++
void
up(wait_queue_t *wait_queue) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(wait_queue)) != NULL) {
            assert(wait->proc->wait_state == WT_KSEM);
            wakeup_wait(wait_queue, wait, WT_KSEM, 1);
        }
    }
    local_intr_restore(intr_flag);
}

void
down(wait_queue_t *wait_queue) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t __wait, *wait = &__wait;
        wait_current_set(wait_queue, wait, WT_KSEM);
    }
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(wait_queue, wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != WT_KSEM) {
        assert(wait->wakeup_flags == 0);
    }
}
```
3. 重新设计 monitor 的接口：此前的`cond_signal`和`cond_wait`无需改动，只需重写`monitor_init`即可。
```c++
// Initialize monitor.
void     
monitor_init (monitor_t * mtp, size_t num_cv) {
    int i;
    assert(num_cv>0);
    mtp->next_count = 0;
    mtp->cv = NULL;
    wait_queue_init(&(mtp->mutex)); //unlocked
    wait_queue_init(&(mtp->next));
    mtp->cv =(condvar_t *) kmalloc(sizeof(condvar_t)*num_cv);
    assert(mtp->cv!=NULL);
    for(i=0; i<num_cv; i++){
        mtp->cv[i].count=0;
        wait_queue_init(&(mtp->cv[i].sem));
        mtp->cv[i].owner=mtp;
    }
}
```

## 总结

### 本实验所涉及的知识点

- 同步与互斥的概念
- 实现同步互斥的机制：
    - 关闭中断使能
    - 信号量
    - 管程和条件变量
- 哲学家就餐问题

### 本实验应涉及但未涉及的知识点

- 读者-写者问题
- 死锁
- 进程通信