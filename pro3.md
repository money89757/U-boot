# u-boot移植：配置 编译 启动 #
移植过程中需要掌握上面三个过程的机制原理。  
移植u-boot之前要自己分析需求。  

配置：顶层目录下执行make 板子名称_config  
为什么需要配置?选择合适的CPU以及标准板  
完成3个功能:生成了3个软链接、生成了`include/config.mk、include/config.h`

```C
启动过程:
1、 39 _start: b   reset 单纯的跳转
2、127     bl  save_boot_params 保存启动参数
3、设置特权模式，便于使用协处理器指令并且操作硬件
	131     mrs r0, cpsr
	132     bic r0, r0, #0x1f
	133     orr r0, r0, #0xd3
	134     msr cpsr,r0

4、设置异常向量表
	141 #if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))
	142     /* Set V=0 in CP15 SCTRL register - for VBAR to point to vector */
	143     mrc p15, 0, r0, c1, c0, 0   @ Read CP15 SCTRL Register
	144     bic r0, #CR_V       @ V = 0
	145     mcr p15, 0, r0, c1, c0, 0   @ Write CP15 SCTRL Register
	146
	147     /* Set vector address in CP15 VBAR register */
	148     ldr r0, =_start
	149     mcr p15, 0, r0, c12, c0, 0  @Set VBAR
	150 #endif                                  

5、154     bl  cpu_init_cp15
					||
					\/
	287 ENTRY(cpu_init_cp15)   关闭缓存、MMU
	288     /*
	289      * Invalidate L1 I/D
	290      */
	291     mov r0, #0          @ set up for MCR
	292     mcr p15, 0, r0, c8, c7, 0   @ invalidate TLBs
	293     mcr p15, 0, r0, c7, c5, 0   @ invalidate icache
	294     mcr p15, 0, r0, c7, c5, 6   @ invalidate BP array
	295     mcr     p15, 0, r0, c7, c10, 4  @ DSB
	296     mcr     p15, 0, r0, c7, c5, 4   @ ISB
	297
	298     /*
	299      * disable MMU stuff and caches
	300      */

6、155     bl  cpu_init_crit
			==》331     b   lowlevel_init
				 bl system_clock_init
				 85     bl mem_ctrl_asm_init
				 89     bl uart_asm_init
					    串口初始化，不同的板子串口不一样，所以这个位置可能需要我们添加或者修改一部分汇编代码时其支持我们自己的串口

7、158     bl  _main
		108     ldr sp, =(CONFIG_SYS_INIT_SP_ADDR) 设置堆栈指针
		115     bl  board_init_f   
						||
						\/
				for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr)
										||
										\/
									这是一个函数指针数组，数组内部存放了一堆的初始化相关接口(在移植时可能需要我们自己修改的代码)
		        462     gd->relocaddr = addr;   计算出内存的高位地址，为了后续对u-boot.bin搬移
				463     gd->start_addr_sp = addr_sp;
				464     gd->reloc_off = addr - _TEXT_BASE;


8、 130     adr lr, here
			......
	136     b   relocate_code  回到了arch/arm/cpu/armv7/start.S  relocate_code标号处,这部分代码就是执行u-boot.bin搬移操作的。
```

### 自搬移: ###
当前三星平台使用的u-boot-fs4412.bin文件特性:包含了`bl1.bin bl2.bin u-boot.bin padding.bin trustzone.bin`  
以上的.bin文件和用户相关的只有两个bl2.bin u-boot.bin  其中bl2.bin是u-boot.bin的前14K。  

## 掌握一个概念:###
bl0固化到IROM中的一段代码，可以将bl1.bin拷贝到IRAM中运行，bl1.bin会初始化一些片内的设备同时会将bl2拷贝到IRAM中，  
bl2也会进行一些初始化(有些和片外外设相关),计算出搬移的地址然后执行搬移工作  

```C
40000000
41000000
uImage
42000000
exynos4412-fs4412.dtb
43000000
文件系统镜像
43e00000  
u-boot.bin    可能会出现文件系统镜像将u-boot.bin覆盖，所以要进程自搬移

```
```

9、对bss段清零
10、167     ldr pc, =board_init_r
			==》包含了一堆的初始化函数(这些东西是移植时可能需要操作的代码)
			    main_loop中获取bootcmd命令进而执行bootm，最后跳转到41000000地址处运行内核镜像
				||
				\/
			内核的工作了

```
## u-boot编译: ##
编译方法:make all  
all是编译的入口，所以进入Makefile文件要去寻找all目标或者all伪目标  
通过查看Makefile可以知道all是伪目标，并且是第一个伪目标，所以编译时可以将all省略  

```c
1、			
439 $(obj)u-boot.bin:   $(obj)u-boot                                                                                          
440         $(OBJCOPY) ${OBJCFLAGS} -O binary $< $@
441         $(BOARD_SIZE_CHECK)			

2、寻找u-boot目标文件
	566 $(obj)u-boot:   depend \                                                                                                  
	567         $(SUBDIR_TOOLS) $(OBJS) $(LIBBOARD) $(LIBS) $(LDSCRIPT) $(obj)u-boot.lds
	568         $(GEN_UBOOT)			
SUBDIR_TOOLS = tools
tools目录下的mkimage.c编译后生成mkimage文件，这个文件最终会交给内核源码，用来生成uImage文件			

244 OBJS := $(addprefix $(obj),$(OBJS) $(RESET_OBJS-))

变量的引用 $(变量名)
函数的引用 $(函数名 参数1,参数2,参数3,....)

232 OBJS  = $(CPUDIR)/start.o ==> arch/arm/cpu/armv7/start.o 		
360 LIBBOARD = board/$(BOARDDIR)/lib$(BOARD).o ==> board/samsung/origen/liborigen.o  
LIBS存放了一堆编译时用到的.o文件

ifdef 判断变量是否有内容
ifndef 判断变量是否没有内容			

220         LDSCRIPT := $(TOPDIR)/arch/$(ARCH)/cpu/u-boot.lds ==> arch/arm/cpu/u-boot.lds

557 GEN_UBOOT = \
558         UNDEF_LST=`$(OBJDUMP) -x $(LIBBOARD) $(LIBS) | \
559         sed  -n -e 's/.*\($(SYM_PREFIX)_u_boot_list_.*\)/-u\1/p'|sort|uniq`;\
560         cd $(LNDIR) && $(LD) $(LDFLAGS) $(LDFLAGS_$(@F)) \
561             $$UNDEF_LST $(__OBJS) \
562             --start-group $(__LIBS) --end-group $(PLATFORM_LIBS) \
563             -Map u-boot.map -o u-boot
564 endif
调用了ld链接器，以前面查询到的一堆.o文件为源文件进行链接，链接前链接器需要知道链接哪些符号或者说哪些分段信息。
到底使用哪些分段信息?u-boot.lds指定了一些分段信息，指定的信息才会被链接器链接，没有指定的忽略。

```

### 什么是链接？###
多个.o文件相同的分段存放到可执行文件中。  

### linux内核移植: ###
内核版本:主版本号.次版本号.修订版本号  
通常次版本号为偶数是稳定版  

2.4  2.6  3.0  3.14  3.18  4.8  

对于3.1x后才使用设备树  

### 内核目录结构: ###

```C
arch 存放和CPU结构相关的代码
Documatition 存放的是说明文档(涉及到设备树的东西都在这里找)
drivers 存放的内核驱动
fs 文件系统
include 存放头文件
init/main.c 移植和驱动中这个文件我们会经常使用

```

### 移植内核前提:需要进程内核配置(使内核选出一部分支持当前平台的功能) ###
#### 如何配置？ ####
顶层目录下执行方法:  
1. cp arch/arm/configs/exynos_defconfig .config
2. make exynos_defconfig
上面的执行结果是可以修改内核选项  

### 什么是内核选项? 每个内核选项都会对应内核源码中的某个源文件 ###
软件裁剪――对内核选项的添加或者删除  

如何操作内核选项?  
事先修改Makefie：  
````C

ARCH ?= arm
CROSS_COMPILE ?= arm-none-linux-gnueabi-

```

顶层目录下执行make menuconfig  
1. 终端要足够大
2. 第一次使用menuconfig时可能会缺少一个软件包 libncurses5-dev

### 每个选项是否被选中，或者说每个选项的值为多少，这时我们需要掌握[] <> () ###
```

[] 有两种情况，*代表选中，空代表未选中
所谓的选中是说明编译内核时，选项对应的源码会参与编译
<> 有三种情况，*代表选中，空代表未选中，M代表模块
() 长的一样但是不一定内容一样。小括号可能存放的是十进制整数，也可能是十六进制整数，也可能是字符串

```
选项对应了哪个源码?这些选项是怎么来的?  
学习Kconfig文件语法――我们menuconfig中的选项就是由Kconfig决定的。  
很重要的特点:Kconfig文件、Makefile、源码 三者之间的关系。  

### Kconfig中的关键语法: ###
source "路径/Kconfig"  引用某个路径下的Kconfig文件  

顶层目录下的Kconfig只是控制使用哪个平台的Kconfig文件  
某个架构下的Kconfig文件决定了menuconfig的选项内容  
```C
menu "abc"  abc是一个主选项。就是说内部有一些子选项
endmenu

string "abc"  abc是一个子选项，用()表示，()中只能存字符串
int "abc"
hex "abc"

bool "abc"  abc是一个子选项，用[]表示
tristate "abc" abc是一个子选项，用<>表示

config 17021_OS
	bool "17021 系统移植课"
会在menuconfig中有一个选项叫做"17021系统移植课"，这个选项对应了一个变量，这个变量叫做CONFIG_17021_OS，
同时这个变量会出现在顶层目录的.config中以及当前目录下的Makefile中，进入Makefile寻找一句话 obj-$(CONFIG_17021_OS) += xxx.o
由此可以知道"17021 系统移植课"这个选项对应的源码是xxx.c 	

```

### 如何在内核中添加一个选项? ###
假设需要在device drivers ――> generic ... options ->添加一个叫做17021 first kconfig的选项。  
1. 切换到drivers/base目录下
2. 进入Makefile文件，在最后一行添加 obj-$(CONIFG_MYDEV) += xxx.o
3. 进入到Makefile的同级目录下的Kconfig文件中，在menu endmenu之间添加:
```C
  config MYDEV
		bool "17021 first kconfig"
		depends on ABC  当前选项是否出现依赖ABC对应的选项
```		
重新打开了menuconfig，发现可以找到一个新的选项叫做17021 first kconfig  

编译内核:make uImage 生成的uImage文件默认会存放在arch/arm/boot  
编译设备树:make dtbs 设备树二进制文件存放在arch/arm/boot/dts  
