1.1进程内核态栈

```
ENTRY(entry_SYSCALL_64)					/*sfw** x86系统调用入口  */
	UNWIND_HINT_EMPTY
	/*
	 * Interrupts are off on entry.
	 * We do not frame this tiny irq-off block with TRACE_IRQS_OFF/ON,
	 * it is too small to ever cause noticeable irq latency.
	 */

	swapgs
	/*
	 * This path is only taken when PAGE_TABLE_ISOLATION is disabled so it
	 * is not required to switch CR3.
	 */
	movq	%rsp, PER_CPU_VAR(rsp_scratch)
	movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp
  
  END
```
