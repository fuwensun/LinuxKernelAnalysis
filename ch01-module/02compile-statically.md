# 1.2模块静态编译

总的流程是：  
-    编译阶段，链接脚本定义 .initcall6.init 段，module_init 标记模块初始化函数 xxx_init 的属性为.initcall6.init，链接时放入 .initcall6.init 段。  
-    运行阶段，start_kernel函数会逐个调用 .initcall6.init 段里的 xxx_init。  


## 1.2.1 链接脚本定义 .initcall6.init 段

在链接文件里由 INIT_DATA_SECTION(16) 定义了各种类型的段，其中包含 .initcall6.init段，代码如下：
```
/* linux\arch\x86\kernel\vmlinux.lds.S */
	INIT_DATA_SECTION(16)
```

INIT_DATA_SECTION(16) 是个宏，调用链如下：
```
INIT_DATA_SECTION(16)
	INIT_CALLS
		INIT_CALLS_LEVEL(6)
			KEEP(*(.initcall6.init))
```

现在具体展开这个宏：
```
/* linux\include\linux\export.h */
#define __VMLINUX_SYMBOL(x) _##x
#define VMLINUX_SYMBOL(x) __VMLINUX_SYMBOL(x)				
````
所以 __VMLINUX_SYMBOL(x) 等于 x，进而 VMLINUX_SYMBOL(x) 等于 x。

```	
/* linux\include\asm-generic\vmlinux.lds.h */
#define INIT_CALLS_LEVEL(level)						\
		VMLINUX_SYMBOL(__initcall##level##_start) = .;		\
		KEEP(*(.initcall##level##.init))			\
		KEEP(*(.initcall##level##s.init))			\
```
所以INIT_CALLS_LEVEL(x)等于：
```
		__initcall_x_start = .;
		KEEP(*(.initcallx.init))	
		KEEP(*(.initcallxs.init))
```

```
/* linux\include\asm-generic\vmlinux.lds.h */
#define INIT_CALLS							\
		VMLINUX_SYMBOL(__initcall_start) = .;			\
		KEEP(*(.initcallearly.init))				\
		INIT_CALLS_LEVEL(6)					\
		VMLINUX_SYMBOL(__initcall_end) = .;
```
所以 INIT_CALLS 等于：
```					
		__initcall_6_start) = .;
		KEEP(*(.initcall6.init))	
		KEEP(*(.initcall6s.init))				
		__initcall_end = .;
```


```		
/* linux\include\asm-generic\vmlinux.lds.h */
#define INIT_DATA_SECTION(initsetup_align)				\		
	.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {		\
		INIT_CALLS						\			
	}

```
```
/* linux\arch\x86\kernel\vmlinux.lds.S */
	INIT_DATA_SECTION(16)
```
所以 INIT_DATA_SECTION(16) 等于：
```
	.init.data : {		
		__initcall_6_start) = .;
		KEEP(*(.initcall6.init))	
		KEEP(*(.initcall6s.init))				
		__initcall_end = .;
	}
```
这定义了.init.data段，链接时带 .initcall6.init 或 .initcall6s.init 属性的内容都放在这里。

## 1.2.2 module_init() 修饰模块初始化函数属性为 .initcall6.init 。

有下面代码可知__define_initcall(xxx, 6) 会被展开成：
static initcall_t __initcall_xxx6 __used __attribute__((__section__(.initcall6.init))) = xxx

```
/* linux\include\linux\module.h */
	#define __define_initcall(fn, id) \
		static initcall_t __initcall_##fn##id __used \
		__attribute__((__section__(".initcall" #id ".init"))) = fn	
```
由下面的代码可知module_init(xxx)最终调用了__define_initcall(xxx, 6)
因此，它也定义了static initcall_t __initcall_xxx6 __used __attribute__((__section__(.initcall6.init))) = xxx。
```
/* linux\include\linux\module.h */
#define module_init(x)  __initcall(x);
	#define __initcall(fn) device_initcall(fn)
		#define device_initcall(fn)     __define_initcall(fn, 6)
```


## 1.2.3 内核初始化时调用模块初始化函数

内核初始化时调用 initcall_t 类型的 __initcall_xxx6 函数，也就是模块初始化函数。调用链如下：
```
/* linux\init\main.c */
	start_kernel
		rest_init
			kernel_thread
				kernel_init
					kernel_init_freeable
						do_basic_setup
							do_initcalls
								do_initcall_level(level)
									do_one_initcall(initcall_t fn)
```