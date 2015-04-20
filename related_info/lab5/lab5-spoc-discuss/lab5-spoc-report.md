# Lab5 Spoc Report
# 裴中煜 2012010685

#### spoc练习报告

内核创建用户进程的方法就是调用user_main函数来创建用户进程，user_main函数调用kernel_execve函数使用int指令发出一个软中断，接着就进行系统调用do_execve处理，在这个函数中会设置好进程的cr3寄存器与内存空间，然后调用load_icode函数载入ELF格式的二进制用户程序文件，而在load_icode中初始化trapframe的代码为

```
tf->tf_cs = USER_CS;
tf->tf_ds = tf->tf_es = tf->tf_ss = USER_DS;
tf->tf_esp = USTACKTOP;
tf->tf_eip = elf->e_entry;
tf->tf_eflags = FL_IF;
```

将相应的段寄存器都设为了USER_CS或USER_DS，esp设为了USTACKTOP，这样表明这个进程运行在用户态。
切换到用户态的过程是在trap中实现的，在do_execve结束后此进程处于RUNNABLE状态等待调度，在下一次系统因为时钟中断调用schedule函数时，proc_run函数将调用switch_to将相应的段寄存器设置好。然后在trapentry.S的尾部有一段代码

```
# call trap(tf), where tf=%esp
    call trap

    # pop the pushed stack pointer
    popl %esp

    # return falls through to trapret...
.globl __trapret
__trapret:
    # restore registers from stack
    popal

    # restore %ds, %es, %fs and %gs
    popl %gs
    popl %fs
    popl %es
    popl %ds

    # get rid of the trap number and error code
    addl $0x8, %esp
    iret

.globl forkrets
forkrets:
    # set stack to this new process's trapframe
    movl 4(%esp), %esp
    jmp __trapret
```

调用完trap后，连续使用6个popl语句恢复了当前程序执行的现场，最后用一条iret指令切换到用户态。

而在memlayout.h中有宏定义
```
#define USERTOP             0xB0000000
#define USTACKTOP           USERTOP
#define USTACKPAGE          256                         // # of pages in user stack
#define USTACKSIZE          (USTACKPAGE * PGSIZE)       // sizeof user stack
```
规定了用户态的堆栈顶在0xB000000处。

用户进程相关函数
1. 启动、运行：do_execve, proc_run; 等待: do_wait; 退出: do_exit
2. 管理与调度: schedule
3. 上下文切换: switch_to
4. 特权级切换: trapentry.S
5. 创建过程并完成资源占用: alloc_proc -> do_fork -> do_execve -> load_icode
6. 退出并完成资源回收: do_exit -> init_main （由initproc内核线程回收）