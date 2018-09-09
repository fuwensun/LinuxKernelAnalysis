# 1.3模块动态编译

总的流程是：  
-    编译阶段，编译生成.ko文件。  
-    运行阶段，通过 insmod 系统调用，在内核空间调用 xxx_init()函数。  

## 1.3.1
由下面的代码可知，动态编译的模块的初始化函数都被重命名为init_module。
```
//sfw**module**module_init定义(模块动态编译)
#define module_init(initfn)					\
	static inline initcall_t __maybe_unused __inittest(void)\	//sfw**__inittest函数测试initfn的类型为initcall_t
	{ return initfn; }					\
	int init_module(void) __attribute__((alias(#initfn)));		//sfw**initfn函数被重命名为init_module
```

#1.3.2
动态编译的模块通过 init_module 系统调用，被载入内核空间，然后调用其初始化函数完成模块的初始化。
```
SYSCALL_DEFINE3(init_module, ...)
	load_module
		do_init_module(mod)
			do_one_initcall(mod->init)
```

这里的模块分两种：
-    在内核代码里，配置成以模块的方式编译，通过开机脚本自动加载（通过系统调用init_module）。
-    用户编写的，通过脚本或指令加载（通过系统调用init_module）。