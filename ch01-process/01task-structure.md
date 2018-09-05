1.1.1进程描述符
```c
struct task_struct {
#ifdef CONFIG_THREAD_INFO_IN_TASK
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
	struct thread_struct		thread;	
  }
  ```