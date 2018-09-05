1.1进程内核态栈
```

```

linux\arch\x86\kernel\cpu\common.c
```
void syscall_init(void)	
	wrmsrl(MSR_LSTAR, (unsigned long)entry_SYSCALL_64);
```

linux\arch\arm64\kernel\entry.S
```
ENTRY(entry_SYSCALL_64)
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp		
END
```
