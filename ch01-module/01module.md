# 1.1 HelloWorld模块
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
    1.静态编译：不定义MODULE宏，helloworld.c 和内核一起编译。  
    2.静态编译：定义MODULE宏，helloworld.c 独立编译。通过 insmod 指令插入内核。  
