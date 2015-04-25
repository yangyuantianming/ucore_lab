#Lab4 Report
```
杨元  2012011331
```

###[练习0]填写已有实验
```
合并了实验1、2、3的代码
```

###[练习1]分配并初始化一个控制块
```
将PCB内各数据初始化即可
```
####[思考题]说明struct context 和 struct trapframe 成员变量的含义与作用。
```
struct中存储进程的上下文：
struct context {
        uint32_t eip;
        uint32_t esp;
        uint32_t ebx;
        uint32_t ecx;
        uint32_t edx;
        uint32_t esi;
        uint32_t edi;
        uint32_t ebp;

};

trapframe 存储中断发生前栈帧信息：

struct trapframe {
        struct pushregs tf_regs;
        uint16_t tf_gs;
        uint16_t tf_padding0;
        uint16_t tf_fs;
        uint16_t tf_padding1;
        uint16_t tf_es;
        uint16_t tf_padding2;
        uint16_t tf_ds;
        uint16_t tf_padding3;
        uint32_t tf_trapno;
        /* below here defined by x86 hardware */
        uint32_t tf_err;
        uintptr_t tf_eip;
        uint16_t tf_cs;
        uint16_t tf_padding4;
        uint32_t tf_eflags;
        /* below here only when crossing rings, such as from user to kernel */
        uintptr_t tf_esp;
        uint16_t tf_ss;
        uint16_t tf_padding5;

} __attribute__((packed));
```

###[练习2]为新创建的内核线程分配资源
```
先调用 alloc_proc()分配并初始化一个新进程，将其parent设置为当前进程。再为其分配栈，复制原有进程执行状态。然后将PCB加入哈希表及 proc_list。
```
####[思考题]请说明ucore是否做到给每个新fork的线程一个唯一的Id？说明理由。
```
做到了。从 get_pid() 函数可见，其将pid每次加1直到找到一个现在可用的的唯一的id。若进程数已达最大值，则会不断循环直至有进程结束。
```

###[练习3]理解 proc_run 函数和它调用的函数如何完成进程切换的。
>本实验执行过程中，创建且运行了几个进程？

```
两个，idle 和 init。
```

>语句local_intr_save(intr_flag);....local_intr_restore(intr_flag);在这里有何作用?

```
local_intr_save 用于关闭中断请求，local_intr_restore 用于恢复中断请求。
```


