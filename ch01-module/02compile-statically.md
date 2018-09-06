# 1.2模块静态编译

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