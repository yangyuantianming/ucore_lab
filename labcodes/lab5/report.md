#Lab5 report
####yangyuan 2012011331

###[练习0]填写已有实验
```
使用meld移植了以前实验的代码。
```

###[练习1]加载应用程序并执行
```
填写load_icode() 函数，设置各寄存器的值。
按注释提示设置各值即可。
```

####[思考题]描述当创建一个用户态进程并加载应用程序后，CPU是如何让这个应用程序最终在用户态执行的。
```
由load_icode()函数实现细节可知，其加载了用户程序并设置了 trapframe 的值，当进行 sys_exex 系统调用后，使用 iret 返回继续执行 trapframe 中 tf_eip 指向的地址，即用户程序的入口地址，如此用户程序得以执行。
```

###[练习2]父进程复制自己的内存空间给子进程。
```
首先获取源页，目标页的内核虚地址，再使用 memcpy 进行复制，再设置与物理地址的对应关系。
```

####[思考题]如何实现COW机制？
```
可以再增加一个标志位， copy_range 时不复制页，只将这个标志位置1，当进程要写入此页时，若标志位为1，则通过 trap 进入处理程序， 复制内存页。
```

###[练习3]理解进程执行 fork/exec/wait/exit 的实现，以及系统调用的实现。
####分析 fork/exec/wait/exit 在实现中是如何形象进程执行状态的？
```
fork() 新建的进程初始状态为UNINIT，通过 wakeup_proc() 变为 RUNNABLE。
exec 时，进程状态为 RUNNABLE。
wait 时，若存在子进程，则状态变为 SLEEPING。
exit 时，状态变为 ZOMBIE。
```

####请给出 ucore 中一个用户态进程的执行状态生命周期图
```
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
```