# 1.1进程结构体
```c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
	/* 线程的状态、标志等需要在汇编代码快速访问的 */
	struct thread_info		thread_info;
#endif
	/* -1 unrunnable, 0 runnable, >0 stopped: */
	volatile long			state;
  	void				*stack;
  
  	int				prio;
	int				static_prio;
	int				normal_prio;
	unsigned int			rt_priority;
  
  	struct list_head		tasks;
  
  	struct mm_struct		*mm;
	struct mm_struct		*active_mm;
  
  	pid_t				pid;
	pid_t				tgid;
  
  	/* Real parent process: */
	struct task_struct __rcu	*real_parent;

	/* Recipient of SIGCHLD, wait4() reports: */
	struct task_struct __rcu	*parent;

	/*
	 * Children/sibling form the list of natural children:
	 */
	struct list_head		children;
	struct list_head		sibling;
	struct task_struct		*group_leader;
  
  	/* PID/PID hash table linkage. */
	struct pid_link			pids[PIDTYPE_MAX];
	struct list_head		thread_group;
	struct list_head		thread_node;
  
  	/* CPU-specific state of this task: */
	/* 和CPU相关的线程状态，这个字段和 thread_info 作用相近。
	*在有些ARCH中为空的struct，内容都迁入 thread_info。
	*另外一些ARCH中 thread_info 大小有限制，就依赖 thread （如x86）。
	*/
	struct thread_struct		thread;	
  }
  ```
  
  
  linux\arch\x86\entry\entry_64.S

  #include <asm/thread_info.h>

首先来看下内核定义的三个静态数据 init_thread_union、init_stack 和 init_task：

声明：init_thread_union、init_stack
```c
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;					//<-----
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;					//<-----
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)]; 
};

extern union thread_union init_thread_union;				//<=====

extern unsigned long init_stack[THREAD_SIZE / sizeof(unsigned long)];	//<=====
```

--------------------------------------
当定义CONFIG_ARCH_TASK_STRUCT_ON_STACK，也就是 thread_info 在 init_stack 里：
```c
union thread_union {
	struct thread_info thread_info;
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
union thread_union init_thread_union;
unsigned long init_stack[THREAD_SIZE / sizeof(unsigned long)];

struct task_struct init_task __init_task_data = {
	.stack		= init_stack,
};
```

--------------------------------------
当定义CONFIG_THREAD_INFO_IN_TASK，也就是 thread_info 在 init_task 里：
```c
union thread_union {
	struct task_struct task;
	unsigned long stack[THREAD_SIZE/sizeof(long)]; 
};

extern union thread_union init_thread_union;
extern unsigned long init_stack[THREAD_SIZE / sizeof(unsigned long)];

struct task_struct init_task {
	.thread_info	= INIT_THREAD_INFO(init_task),
	.stack_refcount	= ATOMIC_INIT(1),
	.stack		= init_stack,
};
```

```
			                                +----------------+
			                                |		 |
			                                |		 |
			                                |		 |
			                                |		 |
			                                |		 |
			                                |		 |
       ---		+---------------+               |		 |
	|		|		|               |		 |
	|		|		|               |		 |
      stack 		|		|               |		 |
       	|      ---	|		|               |		 |
	|	|	|		|               |		 |
	|  thread_info	|		|               |		 |
	|	|	|		|               |		 |
       ---     ---	+---------------+               +----------------+
			thread_union			 task_struct
```
  ```
