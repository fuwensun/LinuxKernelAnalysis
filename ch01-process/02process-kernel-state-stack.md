1.1进程内核态栈
```

```

```
void syscall_init(void)	
{	
	.
	.
	.
	if (static_cpu_has(X86_FEATURE_PTI))
		wrmsrl(MSR_LSTAR, SYSCALL64_entry_trampoline);
	else
		wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
	.
	.
	.
}
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
