# 1.2进程内核态栈

## 1.2.1进程内核态栈的初始化

首先来看下内核定义的 init_stack ：

------------------------------------------------------------------------------------
定义 init_stack ：
```c
/* linux\include\asm-generic\vmlinux.lds.h */
#define INIT_TASK_DATA(align)						\
	. = ALIGN(align);						\
	VMLINUX_SYMBOL(__start_init_task) = .;				\
	VMLINUX_SYMBOL(init_thread_union) = .;				\
	VMLINUX_SYMBOL(init_stack) = .;					\
	*(.data..init_task)						\
	*(.data..init_thread_info)					\
	. = VMLINUX_SYMBOL(__start_init_task) + THREAD_SIZE;		\
	VMLINUX_SYMBOL(__end_init_task) = .;
```
```c
/* linux\arch\x86\kernel\vmlinux.lds.S */
	INIT_TASK_DATA(THREAD_SIZE)
```

声明 init_stack ：
```c
/* linux\include\linux\sched.h */
extern unsigned long init_stack[THREAD_SIZE / sizeof(unsigned long)];
```
定义 init_task ：
```c
/* linux\init\init_task.c */
struct task_struct init_task = {
	.stack		= init_stack,
};
```

--------------------------------------

现在内核初始化相关的流程如下 
-  startup 函数初始化 sp 为 init_stack（具体值后面会说），通常叫它0号进程或者 init_task。
-  接着 start_kernel 调用 sched_init，sched_init 完成调度器的初始化，并把init_task加入调度队列，此后init_task又叫idle_task
-  最后 start_kernel 调用 rest_init，reset_init 创建kernel_init即1号进程，然后启动调度器，进程正式运行起来。


```c
startup_64
	start_kernel
		sched_init();
			init_idle(current, smp_processor_id());
```
```c
/* linux\arch\x86\kernel\head_64.S */
startup_64:
	leaq	(__end_init_task - SIZEOF_PTREGS)(%rip), %rsp		
	
```
这里为什么载入 sp 的是 __end_init_task - SIZEOF_PTREGS ?
```c
/* linux\arch\x86\entry\calling.h */
/* The layout forms the "struct pt_regs" on the stack: */
#define R15		0*8
#define R14		1*8
#define R13		2*8
#define R12		3*8
#define RBP		4*8
#define RBX		5*8
/* These regs are callee-clobbered. Always saved on kernel entry. */
#define R11		6*8
#define R10		7*8
#define R9		8*8
#define R8		9*8
#define RAX		10*8
#define RCX		11*8
#define RDX		12*8
#define RSI		13*8
#define RDI		14*8
/*
 * On syscall entry, this is syscall#. On CPU exception, this is error code.
 * On hw interrupt, it's IRQ number:
 */
#define ORIG_RAX	15*8
/* Return frame for iretq */
#define RIP		16*8
#define CS		17*8
#define EFLAGS		18*8
#define RSP		19*8
#define SS		20*8

#define SIZEOF_PTREGS	21*8				// <---重点
```
看了上面的代码，可知这是对线程切换时硬件上下文入栈的模拟。为什只是模拟而不是真实的？那是应为这是0号进程，是静态创建的不是切换而来的。假设有个-1号进程，0号进程是从-1号进程切换来的，那么这里就要真实的 push 硬件上下文了。

kernel_init 创建时从 init_task 的内核态栈 init_stack 复制了一份，并把它的地址存在struct task 的 stack 字段 。之后所有的进程都是 kernel_init 的子孙进程，它们的栈都是从init_stack复制子而来。


## 1.2.2进程内核态栈的使用-系统调用


内核初始到系统调用初始化：
```c
start_kernel()
	trap_init()
		cpu_init()
			syscall_init()
```


加载 entry_SYSCALL_64 到 MSR_LSTAR 中。之后用户空间发起系统调用，将触发 entry_SYSCALL_64 函数。
```c
/* linux\arch\x86\kernel\cpu\common.c */
void syscall_init(void)	
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
```

entry_SYSCALL_64 函数先保存当下的 sp，也就用户空间的 sp，到 rsp_scratch ，然后把 cpu_current_top_of_stack 加载到 sp 中。
```c
/* linux\arch\x86\entry\entry_64.S */
ENTRY(entry_SYSCALL_64)
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp		
END
``` 

发起系统调用的用户进程也是由它其他进程切换而来的，当时把 task_top_of_stack(next_p) 存入 cpu_current_top_of_stack。
```c
/* linux\arch\x86\kernel\process_64.c */
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
	this_cpu_write(cpu_current_top_of_stack, task_top_of_stack(next_p));
```

而 task_top_of_stack(next_p) 是进程的内核态指针。因此，前面加载到 sp 的 cpu_current_top_of_stack 是进程内核栈指针
```c
/* linux\arch\x86\include\asm\processor.h */
#define task_top_of_stack(task) ((unsigned long)(task_pt_regs(task) + 1))

#define task_pt_regs(task) \
({									\
	unsigned long __ptr = (unsigned long)task_stack_page(task);	\
	__ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;		\
	((struct pt_regs *)__ptr) - 1;					\
}。
```

//todo
-   由于内核态栈创建时栈指针是指向栈底的，当发生系统调用时，进程切换到内核态时内核态栈指针是指向栈底的，当系统调用结束时，内核态栈指针又指向栈底。
因此，系统调用前后内核态栈都是空的。
-   进程要么处于用户态，要么处于内核态。因为系统调用发生前，pc 和 sp 指向用户空间。发生系统调用时，用户空间的 pc 和 sp 被保存起来，内核空间对应的值被载入 pc 和 sp，此时用户空间上下文切换成内核空间上下文，系统调用结束时则相反。


## 1.2.3进程内核态栈的使用-内核进程
//todo

