# 1.3模块动态编译

总的流程是：
	编译阶段，编译生成.ko文件。
	运行阶段，通过 insmod 系统调用，在内核空间调用xxx_mod_init()函数。

```
//sfw**module**module_init定义(模块动态编译)
#define module_init(initfn)					\
//sfw**__inittest函数测试initfn的类型为initcall_t
	static inline initcall_t __maybe_unused __inittest(void)		\
	{ return initfn; }					\
//sfw**initfn函数被重命名为init_module
	int init_module(void) __attribute__((alias(#initfn)));	
```

