## 内核的编译过程:
make uImage

宏观来看，uImage内核镜像的生成过程:  
```c
1、在顶层目录下首先生成了vmlinux
2、arch/arm/boot/Makefile有一个目标文件Image，以vmlinux为源文件并且调用gzip压缩，最终转化为了piggy.gzip.o
3、调用链接器，链接各种.o文件在arch/arm/boot/compressed下生成vmlinux
4、回到arch/arm/boot目录下，以compressed/vmlinux为源文件，调用objcopy命令生成zImage
5、以zImage为源文件调用uimage相关命令生成uImage镜像。
```
uImage和zImage，差了64字节的头信息。这个头信息是给u-boot使用的。  

## 内核的启动过程:
`arch/arm/kernel/vmlinux.lds` 这个链接脚本中有个ENTRY(stext)，由此知道内核的启动入口在stext全局标号处。  

## 内核的启动文件
```C
从arch/arm/kernel/head.S开始:
1、设置特权模式并且屏蔽中断
2、判断了引导程序给内核传参的方式(包括了设备树传参)
3、创建页表
4、初始化MMU
5、开启MMU
	3-5步都是为了后续的地址映射来做铺垫的。

	32位的操作系统中主要使用的是三级映射机制:页目录、页表、页
	32位的虚拟地址:10位 + 10位 + 12位

	无论页目录还是页表还是页他们的本质都是数组。并且上面10 + 10 + 12都可以用来描述数组的大小。

	页目录数组下标是0-1023
	页表下标0-1023
	页数组下标0-1023

	页目录中的一个数组元素存放的是页表的下标值，页表的数组元素存放了页的下标值，页的数组元素接收了MMU传递过来的物理地址。

6、b start_kernel
		||
		\/
	509     setup_arch(&command_line);  
				||
				\/
			 883     mdesc = setup_machine_fdt(__atags_pointer); __atags_pointer这是一个u-boot给内核传递的参数地址
	581     console_init(); 初始化控制台，在控制台初始化之前使用printk函数是不能打印出数据到终端的。

	652     rest_init();
				||
				\/
			382     kernel_thread(kernel_init, NULL, CLONE_FS | CLONE_SIGHAND); 操作系统启动后创建init进程
									 ||
									 \/
								  840     kernel_init_freeable();
												||
												\/
											913     if (sys_open((const char __user *) "/dev/console", O_RDWR, 0) < 0)  应用层的open调用了sys_open
											928         prepare_namespace();
															||
															\/
														589     mount_root();
																	||
																	\/
																510         if (mount_nfs_root())
																					||
																					\/
																				459         err = do_mount_root(root_dev, "nfs",root_moflags, root_data); 内核可能执行这里的函数来使用nfs服务挂载文件系统。
```

## 设备树：
描述硬件信息的一种文件。  
```
最初的驱动逻辑代码和硬件信息写一起的。移植性非常差。
		||
		\/
	驱动和设备的分离(driver专门描述驱动逻辑，device专门描述硬件信息)
													  ||
													  \/
											使用时被放在了内核的某些指定目录下存储并且编译。
													  ||
													  \/
											linus说:arm架构下的东西全是垃圾
													  ||
													  \/
											 参照powerpc中fdt机制，从而有了arm下的设备树

```

设备树:节点和属性

```
节点从根节点

/{
	model = "描述性的语句";
	compatible = "厂家，具体芯片系列或者外设名称","";

	#address-cells = <100>;
	#size-cells = <50>;

	reg = <地址1 偏移量1 地址2 偏移量2 ... >;
	子节点1{
	};

	子节点2@0x50000{
	};

xxx:子节点3{
		interrupt-parent = <&gpx0>;
		interrupts = <中断类型 中断号 中断触发方式>;
	};
};

```

printk使用方法和printf使用方法完全一样。  
printk用于在内核中打印信息的，printf是在应用层打印信息。  

printk在内核中有缓存区、printk健壮性比较高，但是效率低下(这个函数在内核中通常都是用于调试使用的，调试完成后要注释掉)  


使用printk时能否在终端打印信息有两个主要的要求:  
1. 在console_init()之后打印。
2. 消息级别大于控制台级别。

```
消息级别:   0 1 2 3 4 5 6 7 八种级别
控制台级别:   1 2 3 4 5 6 7 8 八种级别
数字越大级别越小
假设消息级别是6，控制台级别为7、8时才会打印信息到终端。

在Ubuntu中:/proc/sys/kernel/printk   4 4 1 7
内核源码中:kernel/printk/printk.c    7 4 1 7

		  4         	4 			   1                    7
		  7         	4 			   1                    7
	  指定控制台级别  默认消息级别   控制台最高级别    默认控制台级别

如果printk没有在终端上打印信息，信息被存放到一个配置文件中――/var/log/syslog

printk("%d\n",a);
```

## 文件系统和文件系统镜像的制作:
文件系统分类:磁盘文件系统、网络文件系统、虚拟文件系统  
文件系统和文件系统镜像的关系:内容上是一样的。  
文件系统镜像是一个二进制文件――通过一些特殊的工具或者命令以文件系统作为源生成的。  

/rootfs目录下有bin sbin etc dev lib ...按理说我们只需要使用mkdir来创建。比如bin目录存放的是命令，如果使用mkdir来创建bin，  
那么bin下面的可执行文件需要我们自己写源码然后编译。是否需要做这些工作――不需要，shell命令别人写好了，我们需要做的是将别人写好的命令移植过来。  

常用的工具:busybox，就是帮助我们产生可执行文件并且存放到文件系统的指定目录下。  
根文件系统的制作工作主要体现在对etc文件夹的操作。etc下有最基本的4个文件――inittab、fstab、profile、init/rcS,这四个文件是使根文件系统运行起来的基础。  

inittab文件特性:  
每一行都会被分为4个域:  
域1:域2:域3:域4  
在嵌入式根文件系统中重来不使用前两个域。  

域3:根文件系统从此开始需要执行哪些动作  
```
	sysinit 通知文件系统后续需要执行哪个脚本
	askfirst 挂载根文件系统成功后会提示――请输入任意键进入根文件系统
	respawn  直接进入根文件系统。
	restart  调用init命令
	ctrlaltdel 执行重启操作系统
```

域4:使用相关命令来完成域3的动作  
```
	/etc/init.d/rcS
	-/bin/sh   -代表了交互的意思。
	/sbin/init
	/sbin/reboot
```

脚本rcS中
```
/bin/mount -a 启动文件系统后可能挂载所有的设备或者文件系统
/sbin/mdev 动态创建设备文件的作用
```

需要在开机后自动挂载设备或者文件系统,这时不能使用mount命令，而需要使用fstab文件――开机挂载、永久挂载  
fstab文件每行有6部分:设备名称 挂载点 挂载设备使用的文件系统 挂载时使用的选项  文件系统备份的时间间隔  fsck检索时间间隔  

## 特殊文件系统:
proc专门描述进程特性，一定挂载到/proc  
sysfs 和动态创建设备节点相关的一种文件系统，一定会被挂载到/sys  
tmpfs 临时的文件系统  通常被挂载到/dev目录下、也可能被挂载到/tmp目录下  
```
dd  if=/dev/zero  of=ramdisk  bs=1k  count=8192

```
if指定了输入文件  
of指定了输出文件  
bs 块的大小  
count 代表了块的个数  

/dev/zero 无限产生二进制0，俗称无底洞  
