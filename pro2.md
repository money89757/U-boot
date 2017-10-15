
# 系统移植 #

开发板上的SD卡或者EMMC中存放了.bin文件，.bin文件通过objcopy格式转化可执行文件来的，所以我们需要知道如何生成可执行文件。  

`u-boot-fs4412.bin:bl1.bin bl2.bin padding.bin u-boot.bin tzsw.bin`  
以上包含的.bin文件中不同厂家操作方法不同，但是u-boot.bin这一步都是一样的。  
u-boot.bin包含bl2.bin  

### u-boot源码目录: ###
版本的选择:根据你们自己的板子的硬件资源以及后续需要使用的内核版本来选择u-boot  
在内核的3.1x版本后开始使用了设备树，所以u-boot必须要具备操作设备树的功能。  
了解重要文件夹:平台相关、平台无关  
arch存放了和cpu体系架构相关文件夹  
board存放的是和厂家相关的文件夹  

如果板子以及芯片系列确定了，可以知道是哪个公司的核心，所以上网去查资料查看当前这个厂家的指定芯片模板是谁?  
三星的exynos4412的模板是origen  


### u-boot配置 ###
配置的目的为了后续编译选择出适合的厂家以及cpu架构  
配置方法:  
1. 上网搜索
2. 查看README文件
			顶层目录下执行make 板子名称_config
			板子名称将某个标准版备份后的名字，例如cp origen fs4412 -a
			所以make fs4412_config

配置之前最好执行一下:`make clean`或者`make distclean`  

配置后具体做了什么工作

```C
775 %_config::  unconfig         注意:这里是配置的入口                                                                                               
776     @$(MKCONFIG) -A $(@:_config=)  ==> mkconfig -A origen

```

寻找Makefile变量内容的方法:  
1. 在命令模式下输入#
2. 在u-boot顶层目录下执行ctags -R
   使用ctrl + ] 跟内容
3. make -p Makefile > farsight
	进入farsight文件寻找需要的变量

通过查找发现CURDIR是顶层目录所以 MKCONFIG := mkconfig  
`$(@:_config=)`由变量$@和规则 _config= 构成的一个表达式     
作用：去掉了origen_config _config字符串剩下了origen  

$@ <==> $(@)后面的才是Makefile的变量的真正用法  
$@代表所有的目标文件  

进入脚本mkconfig  
```c
 22 if [ \( $# -eq 2 \) -a \( "$1" = "-A" \) ] ; then
 23     # Automatic mode
 24     line=`egrep -i "^[[:space:]]*${2}[[:space:]]" boards.cfg` || {
 25         echo "make: *** No rule to make target \`$2_config'.  Stop." >&2
 26         exit 1
 27     }
\( 转义的作用是单纯的使用(这个字符而已，如果不转义(是一个特殊符号
egrep 作用类似于grep 用来搜索某个文件中的字符串
-i用来忽略大小写字母
^以某个字符串开头
[[:space:]] 代表空格或者TAB
$(2) = origen
`egrep -i "^[[:space:]]*${2}[[:space:]]" boards.cfg` 从boards.cfg文件中搜索origen

284 origen               arm     armv7       origen      samsung    exynos

29     set ${line}
origen               arm     armv7       origen      samsung    exynos
   $1                 $2       $3          $4          $5         $6             $#=6

57 CONFIG_NAME="${1%_config}"  去掉$1的尾部_config

BOARD_NAME = origen
61 arch="$2" = arm
62 cpu=`echo $3 | awk 'BEGIN {FS = ":"} ; {print $1}'`  <==> cpu = `echo $3 | cut -d ':' -f 1`
cpu = armv7

67     board="$4" = origen
69 [ $# -gt 4 ] && [ "$5" != "-" ] && vendor="$5" = samsung
70 [ $# -gt 5 ] && [ "$6" != "-" ] && soc="$6" = exynos

113     cd ./include
114     rm -f asm
115     ln -s ../arch/${arch}/include/asm asm

123     ln -s arch-exynos asm/arch             都是在include目录下

134 ( echo "ARCH   = ${arch}"
135     if [ ! -z "$spl_cpu" ] ; then
136     echo 'ifeq ($(CONFIG_SPL_BUILD),y)'
137     echo "CPU    = ${spl_cpu}"
138     echo "else"
139     echo "CPU    = ${cpu}"
140     echo "endif"
141     else
142     echo "CPU    = ${cpu}"
143     fi
144     echo "BOARD  = ${board}"
145
146     [ "${vendor}" ] && echo "VENDOR = ${vendor}"                                                                          
147     [ "${soc}"    ] && echo "SOC    = ${soc}"
148     exit 0 ) > config.mk

```

```
在include目录下创建了config.mk文件，存放了：
ARCH=arm  
CPU=armv7  
BOARD=origen  
VENDOR=samsung  
SOC=exynos  

Makefile makefile GNUmakefile .mk .AIX .Linux  

在include目录下创建了config.h头文件，内部存放当前平台移植时使用到的一些宏以及其它头文件
```

#### 总结:mkconfig脚本实现了3部分功能――3个软链接、include/config.mk、include/config.h ####

# u-boot编译 #
1. 如何编译? 顶层目录下执行make all

```C
u-boot启动
u-boot启动流程
1、进入arch/arm/cpu/armv7/start.S
2、 38 .globl _start
 39 _start: b   reset   u-boot启动的入口
 40     ldr pc, _undefined_instruction  初始化异常向量表入口，具有操作异常的能力
 41     ldr pc, _software_interrupt
 42     ldr pc, _prefetch_abort
 43     ldr pc, _data_abort
 44     ldr pc, _not_used
 45     ldr pc, _irq
 46     ldr pc, _fiq

 126 reset:                                                                                                                    
127     bl  save_boot_params   空跳转
128     /*
129      * set the cpu to SVC32 mode   设置特权模式
130      */
131     mrs r0, cpsr
132     bic r0, r0, #0x1f
133     orr r0, r0, #0xd3
134     msr cpsr,r0

1 #if !(defined(CONFIG_OMAP44XX) && defined(CONFIG_SPL_BUILD))   设置异常向量
142     /* Set V=0 in CP15 SCTRL register - for VBAR to point to vector */
143     mrc p15, 0, r0, c1, c0, 0   @ Read CP15 SCTRL Register
144     bic r0, #CR_V       @ V = 0
145     mcr p15, 0, r0, c1, c0, 0   @ Write CP15 SCTRL Register
146
147     /* Set vector address in CP15 VBAR register */
148     ldr r0, =_start
149     mcr p15, 0, r0, c12, c0, 0  @Set VBAR
150 #endif                                    

287 ENTRY(cpu_init_cp15)
288     /*
289      * Invalidate L1 I/D
290      */
291     mov r0, #0          @ set up for MCR
292     mcr p15, 0, r0, c8, c7, 0   @ invalidate TLBs 提高虚拟地址和物理地址映射效率的一种缓存
293     mcr p15, 0, r0, c7, c5, 0   @ invalidate icache 缓存指令
294     mcr p15, 0, r0, c7, c5, 6   @ invalidate BP array
295     mcr     p15, 0, r0, c7, c10, 4  @ DSB
296     mcr     p15, 0, r0, c7, c5, 4   @ ISB
297
298     /*
299      * disable MMU stuff and caches   关闭MMU――内核启动后才需要使用虚拟地址(MMU映射的)，但是现在还没有执行内核
300      */
301     mrc p15, 0, r0, c1, c0, 0
302     bic r0, r0, #0x00002000 @ clear bits 13 (--V-)
303     bic r0, r0, #0x00000007 @ clear bits 2:0 (-CAM)
304     orr r0, r0, #0x00000002 @ set bit 1 (--A-) Align
305     orr r0, r0, #0x00000800 @ set bit 11 (Z---) BTB                                                                       
306 #ifdef CONFIG_SYS_ICACHE_OFF
307     bic r0, r0, #0x00001000 @ clear bit 12 (I) I-cache
308 #else
309     orr r0, r0, #0x00001000 @ set bit 12 (I) I-cache
310 #endif

331     b   lowlevel_init       @ go setup pll,mux,memory

82     bl system_clock_init     初始化系统时钟
 83
 84     /* Memory initialize */
 85     bl mem_ctrl_asm_init    初始化内存控制器

  bl uart_asm_init  串口初始化(需要我们认为的自己修改了)

  158     bl  _main  
		==》 108     ldr sp, =(CONFIG_SYS_INIT_SP_ADDR) 初始化堆栈指针，为了后续执行c语言使用。

115     bl  board_init_f
303     for (init_fnc_ptr = init_sequence; *init_fnc_ptr; ++init_fnc_ptr) {
								||
								\/
							函数指针数组(数组内部存放了若干个初始化函数)
462     gd->relocaddr = addr;  计算出u-boot.bin重定位的地址
463     gd->start_addr_sp = addr_sp;
464     gd->reloc_off = addr - _TEXT_BASE;						

进入arch/arm/cpu/armv7/start.S
170 ENTRY(relocate_code)
171     mov r4, r0  /* save addr_sp */
172     mov r5, r1  /* save addr of gd */
173     mov r6, r2  /* save addr of destination */
174
175     adr r0, _start
176     cmp r0, r6
177     moveq   r9, #0      /* no relocation. relocation offset(r9) = 0 */
178     beq relocate_done       /* skip relocation */                                                                         
179     mov r1, r6          /* r1 <- scratch for copy_loop */
180     ldr r3, _image_copy_end_ofs
181     add r2, r0, r3      /* r2 <- source end address     */
............................
以上代码实现u-boot.bin的自搬移

```

#### 到此为止u-boot启动的第一阶段完成了(也就是我们所谓的bl2.bin) ####
---

```C
清零BSS段
167     ldr pc, =board_init_r

len = readline (CONFIG_SYS_PROMPT); 获取u-boot命令
rc = run_command(lastcommand, flag); 执行u-boot命令(其中包含了bootcmd――>bootm)
bootm命令会跳转到41000000运行内核镜像
```
