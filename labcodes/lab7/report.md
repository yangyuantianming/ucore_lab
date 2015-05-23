###Lab7 report

计24  杨元  2012011331

###[练习0]填写已有实验
```
使用meld移植了已有实验，并进行了适当修改。
```

###[练习1]理解内核级信号量和基于内核级信号量的哲学家就餐问题

####给出内核级信号量的设计描述，说出其大致流程。

信号量的实现包括一个结构体和四个方法。
```C
typedef struct {
    int value;
    wait_queue_t wait_queue;
} semaphore_t;
```

```C
void sem_init(semaphore_t *sem, int value) {
    sem->value = value;
    wait_queue_init(&(sem->wait_queue));
}
```
>sem_init() 初始化了value值和等待队列。

```C
static __noinline void __up(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    {
        wait_t *wait;
        if ((wait = wait_queue_first(&(sem->wait_queue))) == NULL) {
            sem->value ++;
        }
        else {
            assert(wait->proc->wait_state == wait_state);
            wakeup_wait(&(sem->wait_queue), wait, wait_state, 1);
        }
    }
    local_intr_restore(intr_flag);
}
```
>up()方法调用了__up(),首先关闭中断，若队列为空则value+1，否则等待队列队首进程。最后恢复中断。

```C
static __noinline uint32_t __down(semaphore_t *sem, uint32_t wait_state) {
    bool intr_flag;
    local_intr_save(intr_flag);
    if (sem->value > 0) {
        sem->value --;
        local_intr_restore(intr_flag);
        return 0;
    }
    wait_t __wait, *wait = &__wait;
    wait_current_set(&(sem->wait_queue), wait, wait_state);
    local_intr_restore(intr_flag);

    schedule();

    local_intr_save(intr_flag);
    wait_current_del(&(sem->wait_queue), wait);
    local_intr_restore(intr_flag);

    if (wait->wakeup_flags != wait_state) {
        return wait->wakeup_flags;
    }
    return 0;
}
```
>down()方法调用了__down()，首先关闭中断。若value>0，则value-1，并直接返回恢复中断；否则将当前进程加入等待队列并恢复中断。再调用schedule()交出CPU使用权，再关闭中断，从等待队列中删除当前进程，最后恢复中断。

####给出用户态进程/线程提供信号量机制的设计方案，并比较说明给内核级提供信号量机制的异同。
>在实现的原理上没有区别，只要将相关方法设置成系统调用的方式，以在用户态直接调用。

###[练习2]完成内核条件变量和基于内核级条件变量的哲学家就餐问题

####请给出内核级条件变量的设计描述，并说出其大致执行流程。
```C
typedef struct condvar{
    semaphore_t sem;        // the sem semaphore  is used to down the waiting proc, and the signaling proc should up the waiting proc
    int count;              // the number of waiters on condvar
    monitor_t * owner;      // the owner(monitor) of this condvar
} condvar_t;
```

>其中用到了monitor结构体

```C
typedef struct monitor{
    semaphore_t mutex;      // the mutex lock for going into the routines in monitor, should be initialized to 1
    semaphore_t next;       // the next semaphore is used to down the signaling proc itself, and the other OR wakeuped waiting proc should wake up the sleeped signaling proc.
    int next_count;         // the number of of sleeped signaling proc
    condvar_t *cv;          // the condvars in monitor
} monitor_t;
```

>在monitor.c中实现了如下方法:

```C
// Initialize variables in monitor.
void     monitor_init (monitor_t *cvp, size_t num_cv);
// Unlock one of threads waiting on the condition variable.
void     cond_signal (condvar_t *cvp);
// Suspend calling thread on a condition variable waiting for condition atomically unlock mutex in monitor,
// and suspends calling thread on conditional variable after waking up locks mutex.
void     cond_wait (condvar_t *cvp);
```
>init_monitor() 用于初始化monitor结构体，cond_signal() 用于唤醒一个在条件变量上等待的进程， cond_wait() 表示该进程等待某个条件需要睡眠。

####给出用户态进程/线程提供条件变量机制的设计方案，并比较说明给内核级提供条件变量机制的异同。
>在用户态的信号量已经实现的前提下，只需提供用户态条件变量接口即可。

