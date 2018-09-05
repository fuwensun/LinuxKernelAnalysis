1.1进程内核态栈
```

```

```
ENTRY(entry_SYSCALL_64)	
		.
		.
		.
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp
		.
		.
		.
  
  END
```


```
ENTRY(entry_SYSCALL_64)	
		.
		.
		.
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp
		.
		.
		.
  
  END
```
