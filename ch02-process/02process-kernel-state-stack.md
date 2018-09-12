# 1.2进程内核态栈

## 1.2.1进程内核态栈的来源--内核初始化栈
-   内核代码静态定义内核初始化栈 init_stack 和 内核线程 init_task，并将 init_stack 指定为 init_task 的栈。
-   内核初始化，将 init_task 转化为 idle 内核线程，然后第一此动态创建内核线程 kernel_init，即1号进程，第一次动态创建进程内核态栈 kernel_init->stack。之后同样的方式创建了 kthreadd，即2号进程，和它的内核态栈 kthreadd->stack。

--------------------------------------
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
由上面的代码可知 init_stack 指定为 init_task 的栈

--------------------------------------
现在来看内核初始化，相关的流程如下：
-  startup 函数初始化 sp 为 init_stack（不够准确，具体值看后面），通常叫它0号进程或者 init_task。
-  接着 start_kernel 调用 sched_init，sched_init 完成调度器的初始化，并把 init_task 加入调度队列，此后 init_task 转化成 idle 进程。
-  最后 start_kernel 调用 rest_init，reset_init 创建 kernel_init 和 kthreadd，然后启动调度器，进程正式运行起来。


```c
startup_64
	start_kernel
		sched_init
			init_idle
		rest_init
			kernel_thread(kernel_init...)
			kernel_thread(kthreadd...)

```
```c
/* linux\arch\x86\kernel\head_64.S */
startup_64:
	leaq	(__end_init_task - SIZEOF_PTREGS)(%rip), %rsp		
	
```
这里为什么载入 sp 的是 __end_init_task - SIZEOF_PTREGS ?
```c
/* linux\arch\x86\entry\calling.h */
#define R15		0*8
#define R14		1*8
#define R13		2*8
		.
		.
		.
#define RSP		19*8
#define SS		20*8

#define SIZEOF_PTREGS	21*8				// <---重点
```
由上面的代码，可知这是对线程切换时硬件上下文入栈的模拟。为什只是模拟而不是真实的？这个函数不返回了，如果要返回这里也要 push 硬件上下文。另外如果这里不模拟进程切入，那么切出时也要特殊的操作了，这样这个进程变为 idle 进程后对它的调度时的切入切出操作也和其它进程不一样。

总结：
到这里我知道了内核创建了3个进程：0号idle、1号kernel_init、2号kthreadd。而且 idle 的内核态栈为 init_task，那么kernel_init 和 kthreadd 的内核态栈是什么？这里直接给出答案，是 init_task 的拷贝，具体后面分析。


## 1.2.2进程内核态栈的使用-系统调用
-    系统调用机制的初始化
-    发起系统调用流程

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


## 1.2.3进程内核态栈的使用-内核线程
//todo

内核线程：所有的内核线程都共享同一套代码段--内核代码，共享同一个虚拟地址空间--内核空间，但是各自有独立的栈。

-    内核线程的创建
-    内核线程栈的创建
-    调度器调度内核线程时为它指定的栈

内核线程的创建者 kthreadd_task 内核线程,即2号进程

```c
startup_64
	start_kernel
		rest_init
			kernel_thread(kthreadd...)

```

内核定义了内核线程的生产者 kthreadd_task 内核线程（消息的消费者）和消息中间件 kthread_create_list 链表：

```c
static LIST_HEAD(kthread_create_list);
struct task_struct *kthreadd_task;
```
```c
int kthreadd(void *unused)
{

	for (;;) {
		while (!list_empty(&kthread_create_list)) {
			struct kthread_create_info *create;

			create_kthread(create);
		}
	}

}
```
现在来看下消息的生产者内核例程 __kthread_create_on_node，它生成 create 消息，放入 kthread_create_list 消息中间件，唤醒消息消费者 kthreadd_task 。

```c
__kthread_create_on_node

	struct kthread_create_info *create = kmalloc(sizeof(*create),GFP_KERNEL);

	list_add_tail(&create->list, &kthread_create_list);

	wake_up_process(kthreadd_task);

```
更常用的是对 __kthread_create_on_node 的封装（附录A4）


下面展示了内核线程的栈的创建流程：
```c
kthreadd //kthreadd_task
	create_kthread
		kernel_thread
			_do_fork
				copy_process
					dup_task_struct(current, node);
						stack = alloc_thread_stack_node(tsk, node);
						tsk->stack = stack;
```