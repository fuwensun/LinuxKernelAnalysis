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

	struct thread_struct		thread;	
  }
  ```
   
struct task_struct 描述了一个进程的状态和拥有的资源，是一个进程实例的抽象。
-    thread_info：进程信息标志(thread information flags)标记进程的各种状态：有无信号挂起、系统调用跟踪、兼容32位等（见附录A2），entry_64.S 中要快速访问。
-    thread：特定体系结构的线程状态，如寄存器，特有的数据等。这个字段和 thread_info 作用相近，在有些ARCH中为空的struct，内容都迁入 thread_info，另外一些ARCH中 thread_info 大小有限制，就依赖 thread （如x86）。
-    state：进程的状态：运行，不可运行，停止。
-    stack：进程内核态栈。
-    
-    
-    
-    
-    
## 

首先来看下内核定义的三个数据结构：
由于所有的内核态栈都是从父进程复制来的（后面章节会讲解），而 init_task 是所有进程的祖先进程，它的栈 init_stack 是所有进程内态核栈的父本，这里用它来指代进程的 stack 字段。
声明：thread_union、stack
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
extern unsigned long init_stack[THREAD_SIZE / sizeof(unsigned long)];	//<=====
```

--------------------------------------
当定义CONFIG_ARCH_TASK_STRUCT_ON_STACK，也就是 thread_info 在 stack 里：
```c
union thread_union {
	struct thread_info thread_info;
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};

unsigned long init_stack[THREAD_SIZE / sizeof(unsigned long)];

struct task_struct = {
        volatile long			state;
};
```

--------------------------------------
当定义CONFIG_THREAD_INFO_IN_TASK，也就是 thread_info 在 task 里：
```c
union thread_union {
	struct task_struct task;
	unsigned long stack[THREAD_SIZE/sizeof(long)]; 
};

extern unsigned long init_stack[THREAD_SIZE / sizeof(unsigned long)];

struct task_struct {
	struct thread_info		thread_info;
	volatile long			state;
  	void				*stack;
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
