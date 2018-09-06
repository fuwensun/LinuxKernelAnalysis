# 1.1模块
## 1.1.1 HELLO WORLD

HELLO WORLD！！！  
伟大的HELLO WORLD！！！  
每次打印HELLO WORLD都说明我又在学新的编程语言了！！！  
helloworld.c
```
#incldue <linux/kernel.h>
#incldue <linux/module.h>

static int hw_init(void){
	printk("hello world !");
	return 0;
}

static void hw_exit(void){
	printk("goodbye!");
}
module_init(hw_init);
module_exit(hw_exit);
```

内核模块有两种用法，通过MODULE宏来控制：  
1、静态编译：不定义MODULE宏，helloworld.c 和内核一起编译。  
2、静态编译：定义MODULE宏，helloworld.c 独立编译。通过 insmod 指令插入内核。  



## 1.1.1模块静态编译

总的流程是：
	编译阶段，链接文件定义 __initcall 区，module_init 将模块的 xxx_module_init，放入 __initcall 区。
	运行阶段，start_kernel函数会逐个调用 __initcall 区里的 xxx_module_init。


```

```

```
#define module_init(x)  __initcall(x);
	#define __initcall(fn) device_initcall(fn)
		#define device_initcall(fn)     __define_initcall(fn, 6)
			#define __define_initcall(fn, id) \
                static initcall_t __initcall_##fn##id __used \
                __attribute__((__section__(".initcall" #id ".init"))) = fn	<--module静态编译时，xxx_mod_init函数被设置进initCall区！！！！
*/
```

```
//sfw**module**内核初始化调用链
start_kernel
	rest_init
    	kernel_thread
        	kernel_init
            	kernel_init_freeable
                	do_basic_setup				<------------
                    	do_initcalls
                        	do_initcall_level(level)
                            	do_one_initcall(initcall_t fn)	<--module静态编译时，xxx_mod_init函数被调用！！！！
```



## 1.1.2模块动态编译

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

