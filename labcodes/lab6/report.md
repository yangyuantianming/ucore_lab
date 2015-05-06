#Lab6 report
####计24  杨元  2012011331

###[练习0]移植已有实验
```
使用meld对实验1,2,3,4,5进行了移植
```

###[练习1]使用Round Robin 调度算法
####请理解并分析sched_class 中各个函数指针的用法，结合Round Robin 调度算法描述ucore 的调度执行过程。
```
RR_init() 用于初始化run_queue，RR_enqueue() 用于向run_queue 中添加进程， RR_dequeue() 用于从run_queue 中移除一个进程， RR_pick_next() 用于从run_queue 中选取一个进程运行，proc_tick() 用于在每次时钟中断时将当前进程的time_slice 减一。

ucore 调度过程：
时钟中断时调用sche_class_proc_tick() 改变当前进程的time_slice。在do_exit(), do_wait(), init_main(), cpu_idle() 中调用schedule() 进行进程调度，schedule() 将当前进程放入run_queue，并从中选出下一进程，调用proc_run() 执行当前进程。
```

###[练习2]使用Stride Scheduling调度算法
```
用default_sched_strde_c覆盖default_sched.c 
stride_init(): 初始化
stride_enqueue():初始化刚进入run_queue 的进程proc 的stride， 将proc 插入runqueue。
stride_dequeue():从run_queue 中删除相应proc。
stride_pick_next():找到run_queue中stride 最小的进程，并更新其stride。
stride_proc_tick():减小time_slice, 检测当前进程是否用完了分配的时间片，若已用完，则设置切换标记。
```

