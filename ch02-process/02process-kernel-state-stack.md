# 1.2进程内核态栈

## 1.2.1进程内核态栈的初始化
-//todo
```

```

## 1.2.2进程内核态栈的使用-系统调用


内核初始到系统调用初始化：
```
start_kernel()
	trap_init()
		cpu_init()
			syscall_init()
```


加载 entry_SYSCALL_64 到 MSR_LSTAR 中。之后用户空间发起系统调用，将触发 entry_SYSCALL_64 函数。
linux\arch\x86\kernel\cpu\common.c
```
void syscall_init(void)	
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
```

entry_SYSCALL_64 函数先保存当下的sp，也就用户空间的sp，到rsp_scratch，然后把 cpu_current_top_of_stack 加载到sp中。
linux\arch\x86\entry\entry_64.S
```
ENTRY(entry_SYSCALL_64)
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp		
END
``` 

发起系统调用的用户进程也是由它其他进程切换而来的，当时把 task_top_of_stack(next_p) 存入 cpu_current_top_of_stack。
linux\arch\x86\kernel\process_64.c
```
__switch_to(struct task_struct *prev_p, struct task_struct *next_p)
	this_cpu_write(cpu_current_top_of_stack, task_top_of_stack(next_p));
```

而 task_top_of_stack(next_p) 是进程的内核态指针。因此，前面加载到 sp 中的 cpu_current_top_of_stack 是进程内核栈指针
linux\arch\x86\include\asm\processor.h
```
#define task_top_of_stack(task) ((unsigned long)(task_pt_regs(task) + 1))

#define task_pt_regs(task) \
({									\
	unsigned long __ptr = (unsigned long)task_stack_page(task);	\
	__ptr += THREAD_SIZE - TOP_OF_KERNEL_STACK_PADDING;		\
	((struct pt_regs *)__ptr) - 1;					\
})
```