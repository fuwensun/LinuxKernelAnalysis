1.1进程内核态栈
```

```
内核初始到系统调用初始化：
	start_kernel()
		trap_init()
			cpu_init()
				syscall_init()


加载 entry_SYSCALL_64 到 MSR_LSTAR 中。之后用户空间发起系统调用，将触发 entry_SYSCALL_64 函数。
linux\arch\x86\kernel\cpu\common.c
```
void syscall_init(void)	
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
```

entry_SYSCALL_64 函数先保存当下的sp，也就用户空间的sp，到rsp_scratch，然后 cpu_current_top_of_stack 加载到sp中。
linux\arch\x86\entry\entry_64.S
```
ENTRY(entry_SYSCALL_64)
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp		
END
```
