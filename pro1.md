
#  什么是系统移植？
系统:
* 软件系统
* 硬件系统。  

硬件系统:
```
		  核心板exynos4412(片上系统) ―― CPU(核是cortex-a9)，IRAM(片内内存256K),IROM，外设控制器
		/
FS4412
		\
		  外设板 ―― usb、led、蜂鸣器、按键、串口、片外内存1G...
```

软件系统:
```
linux操作系统――既包含应用程序，又包含内核源码
用户空间
		应用程序(第三方源码)
――――――――――――――――――――――――――――――――――――――――――――――――――――――――――
内核空间                        
		进程管理
		设备管理
		内存管理
		文件系统
		驱动
		网络协议栈
		中断机制
――――――――――――――――――――――――――――――――――――――――――――――――――――――――――
硬件
```

### 上层应用到底层基本流程 ###
```
用户空间
			应用程序
			  ||		||
			  ||		\/
			  ||	  库函数
――――――――――――――||――――――――||――――――――――――――――――――――――――――――――――――――
内核空间	  ||		\/
			  \/
			    系统调用
				   ||
				   \/
			   虚拟文件系统(描述文件的，linux系统中一切皆文件，一切设备在内核看来也都是文件)
				   ||
				   \/
				  驱动(逻辑，硬件信息)
―――――――――――――――――――||―――――――――――――――――――――――――――――――――――――――
				   \/
				  设备

```

` 移植:需要修改并且编译一些源码使其运行于另一种(CPU架构的不同)不同的平台上`

### 软硬件可裁剪 ###  
硬件裁剪:设计硬件板子时根据需求添加或者删除某些不需要的设备。    
软件裁剪:内核功能的添加和删除。make menuconfig(Kconfig Makefile 驱动)    

```
gcc
arm-none-linux-gnueabi-gcc 交叉编译器
编写主机     运行主机
  x86        x86       使用gcc
  x86        arm       使用交叉编译器

假设已有编译好的交叉编译器，需要使用到Ubuntu中。

```

# 系统移植到底在干吗？#
```C
对比windows:进入BIOS界面――>拷贝windows镜像到硬盘上――>引导系统镜像加载到内存中――>看到windows桌面后，可能装一些驱动以及一些应用软件
系统移植:安装linux系统到某个平台运行的过程。
	     引导程序(u-boot)――>拷贝linux内核到emmc中，并且加载到内存中运行――>挂载文件系统――>后续可能修改重新运行一些驱动或者应用软件
```

## 编译过程: ##
1. 预处理 gcc -E 1.c -o 1.i 或者 cpp 1.c -o 1.i  
2. 编译 gcc -S 1.i -o 1.s 或者 cc1 1.i -o 1.s  
3. 汇编 gcc -c 1.s -o 1.o 或者 as 1.s -o 1.o  
4. 链接 gcc 1.o  或者 ld 1.o  

```C
ld -m elf_i386 -dynamic-linker /lib/ld-linux.so.2 -o 1 1.o
   /usr/lib/i386-linux-gnu/crt1.o /usr/lib/i386-linux-gnu/crti.o    
   /usr/lib/gcc/i686-linux-gnu/4.6/crtbegin.o -lc  
   /usr/lib/gcc/i686-linux-gnu/4.6/crtend.o /usr/lib/i386-linux-gnu/crtn.o  

```
链接器在链接时必须要使用符号来生成可执行文件。  
符号主要代表的是函数名称或者静态变量名称  

nm 二进制文件  
虚拟地址  符号类型  符号名称

符号名称给链接器使用的。链接器只认识符号名字  
符号类型:T正文段 D数据段 R只读数据段 U未定义 W弱标号

符号名称的作用是给链接器使用，通过符号名称索引到虚拟地址进而索引到二进制信息，最终存放到可执行文件中。  

strip 可执行文件    删除符号表，用来进行代码瘦身，因为嵌入式平台资源有限，索引要尽最大可能节省资源。  
注意:strip 不能对中间文件进行操作，例如.o文件。  


objdump -D 二进制文件 > 反汇编文件   将二进制文件转化为汇编文件  
```
反汇编.bin文件: arm-none-linux-gnueabi-objdump -D -b binary     -m arm           u-boot.bin > u-boot.dis
生成.bin文件  指定使用arm平台	 

```
.bin文件用于开发板上运行的一种文件，不同厂家使用的后缀也可能不同，三星规定了使用.bin为后缀的文件  
.bin文件里面存放的是纯数据。  

objcopy 用来格式转化的命令  
`arm-none-linux-gnueabi-objcopy -O binary u-boot boot.bin`  从可执行文件u-boot中提取出数据存放到u-boot.bin文件中。  

### 3种文件 ###
* uImage内核镜像
* exynos4412-fs4412.dtb设备树的二进制文件
* ramdisk.img文件系统镜像

## tftp简单文件传输协议 ##
这种协议不能操作目录。  
1. sudo apt-get install tftpd-hpa 下载安装tftp服务器
2. sudo vi /etc/default/tftpd-hpa
3. TFTP_USERNAME="tftp"
4. TFTP_DIRECTORY="/tftpboot" 这个目录是根目录下的tftpboot文件夹，这个文件夹名字和路径其实都可以自己指定
5. TFTP_ADDRESS="0.0.0.0:69"
6. TFTP_OPTIONS="-l -c -s"  既可以上传又可以下载
3. sudo /etc/init.d/tftpd-hpa restart 或者 sudo service tftpd-hpa restart

## nfs服务 ##
1. sudo apt-get install nfs-kernel...
2. sudo vi /etc/exports
```C
	最后一行添加:/rootfs *(rw,sync,no_subtree_check,no_root_squash)
	/rootfs  自己在根目录下创建rootfs文件夹
	squash 排挤
	root_squash 排挤root用户
	no_root_squash 不排挤root用户
```  
3. sudo /etc/init.d/nfs开头的一个服务名称 restart
   sudo service nfs开头的服务名称 restart

`rootfs文件夹下存放的是文件系统内容`

## u-boot 引导程序 ##
u-boot源码:汇编(初始化硬件)、c语言、脚本、Makefile  
u-boot有两种模式:交互模式和自启动模式  


## 交互模式: FS4412 # ##

u-boot命令:  
printenv 打印u-boot交互模式下的环境信息。 print pri  

ipaddr  代表了开发板ip  
serverip 代表了Ubuntu的ip  

删除环境变量:setenv 环境变量名  
保存环境变量:saveenv  

设置环境变量内容:setenv ipaddr 192.168.1.123    

两个ip地址设置好后，要通过ping来验证一下。在开发板上使用ping + Ubuntu的ip 来验证。


## tftp命令: ##
```c
tftp 41000000 uImage 通过tftp服务从Ubuntu的/tftpboot目录下下载uImage文件到开发板的片外内存41000000地址处
三星的片外内存地址范围：[40000000,A0000000] 华清的板子片外内存范围[40000000,80000000]

tftp 42000000 exnos4412-fs4412.dtb

bootm 内核镜像地址 文件系统镜像地址 设备树二进制文件地址
	  注意:如果某个地址没有用到用-来替代，这三个地址顺序不能改变。
bootm的作用是:1、跳转到指定地址处运行  2、给内核传参(主要是设备树)

go 41000000  单纯跳转到41000000地址去运行

bootcmd tftp 41000000 uImage\;tftp 42000000 exynos4412-fs4412.dtb\;bootm 41000000 - 42000000
bootcmd告诉u-boot如果进入到自启动模式需要做哪些工作。

setenv bootargs root=/dev/nfs(使用nfs服务) nfsroot=ubuntu的ip地址:/rootfs rw console=ttySAC2,115200 init=/linuxrc  ip=开发板ip
当前的bootargs最主要的目的是告诉开发板需要使用Ubuntu的nfs服务

sudo mount -t nfs ubuntu的ip地址:/rootfs /mnt

```
