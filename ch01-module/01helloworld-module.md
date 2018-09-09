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
-    1、静态编译：helloworld.c 和内核一起编译。  
-    2、动态编译：helloworld.c 独立编译。通过 insmod 指令插入内核。  

常用的命令：
lsmod						//list module
insmod helloworld.ko		//insert module
rmmod helloworld.ko			//remove module
modinfo hellowrld			//module probe
dmesg 						//read all messages
demsg -c					//read and clear all messages

insmod helloworld.ko | dmesg -c
rmmod helloworld.ko | dmesg -c

## 1.1.2 ubuntu 下内核开发环境的建立

sudo apt-get update
sudo apt-get install linux-source --fix-missing				/* /usr/src */
cd /usr/src
sudo make oldconfig
suod make -j2




