## u-boot启动过程: ##
1. 顶层目录下有一个u-boot.lds（arch/arm/cpu/armv7/start.o）  
   进入到arch/arm/cpu/armv7/start.S文件  

2. b reset  
	 初始化异常向量表  

3. 保存u-boot的启动参数(当前环境没有)  
   设置了特权模式(初始化硬件，并且使用到协处理器指令)  

4. 关闭mmu并且关闭相关缓存

5. 初始化系统时钟、并且初始化内存控制器，后续可能需要我们初始化串口

6. 堆栈的初始化(为了后续执行c代码)

7. board_init_f（有一个函数指针数组，数组内部存放的都是一些初始化功能函数，计算出了u-boot搬移时需要的动态高位地址）

8. u-boot自搬移(bl0，bl1，bl2)
	bl0固化到IROM中最终将bl1加载到IRAM中运行，bl1会初始化一些片内的硬件并且将bl2加载到IRAM中运行,bl2是u-boot.bin的前14K(计算出搬移地址并且实现搬移工作)

9. 清零bss段，并且执行board_init_r,这个函数的内部会有大量的初始化函数(片外设备的初始化函数)，最终会调用bootcmd环境信息。

## 内核的配置: ##
一切的嵌入式平台的大型源码都会有配置方法。  
因为大型源码是支持多种架构或者厂家，我们自己在用的时候架构和厂家已经固定了，  
所以我们要通过配置的方法从多种架构和厂家中选择出适合自己平台的源码来进行后续的编译。  

## 查看README文件发现配置方法: ##
1. 顶层目录下执行make xxx_defconfig
2. cp arch/arm/configs/xxx_defconfig .config 习惯使用这种方法
   xxx代表芯片系列名称,不包含子系列

## make menuconfig
菜单中会涉及到各种选项，每个选项都可能对应内核中的某部分源码。

[]      
<>  
()  

## Kconfig语法：
source 引用其他路径的Kconfig  

config XXX   对应同级目录下Makefile中的CONFIG_XXX变量
bool 代表了[]  
tristate 代表了<>  
string 代表了() 内部存字符串  
int  
hex  
depends on XXX 如果XXX对应的选项没有选中，那么当前选项不会出现。  

例子:寻找网卡驱动选项位置
```c
1、在~/source/linux-3.14/drivers/net/ethernet/davicom/Makefile中找到obj-$(CONFIG_DM9000) += dm9000.o
2、驱动的同级目录下进入Kconfig寻找DM9000
3、通过DM9000找到tristate "DM9000 support"，由此可以知道我们最终需要选择的选项是DM9000 support
4、返回上一层目录，进入Kconfig文件，找到bool "Ethernet driver support"
5、再返回上一级目录，进入Kconfig文件，找到bool "Network device support"
6、再返回上一局目录，进入Kconfig文件，找到 menu "Device Drivers"
7、最终我们在menuconfig中的寻找顺序是Device Drivers->Network device support->Ethernet driver support->DM9000 support

```

## 设备树 device tree

## 设备和驱动的分离

早期的驱动是既包含了驱动逻辑又包含了设备的硬件信息。这种驱动移植性非常差。  
所以linux社区的大神就提出一种思想――分离思想。  

后面我们讲的驱动一部分被称为driver，一部分被称为device.  

在arm架构下device的设备信息被放在内核的指定目录下进行编译，编译之后的硬件信息最终都会被存放到uImage镜像中。  

#### 命名方式:
.dts设备树源文件   
.dtb设备树二进制文件   
.dtsi设备树头文件  

## 设备树由两大部分组成:节点、属性

节点:根节点、根节点的子节点   
   节点用来描述某种设备  
属性:通用属性、专有属性  

节点包含属性。所有的节点包含于根节点。  

```
/{
	子节点1{
		属性 = 属性值;
	};

	子节点2{
		属性 = 属性值;
	};

	子节点3@123456{
	}；

	子节点3@234567{
	}；
};

```
上面的节点3代表了同种设备，通过@后面的地址来具体区分同种设备中的不同的设备  

`model = "某款芯片的描述性语句";` 这个属性通常出现于根节点中,这个属性不重要。  
`compatible = "samsung,exynos4412","samsung,exynos"; `这个属性的后面接了若干字符串，每个字符串用逗号分成了两个部分，前面的部分包哈后面的部分。  
对于整体字符串来讲，后面的字符串兼容范围要大于前面的字符串。  
驱动如何能知道它需要使用哪个节点哪个属性呢？驱动和设备树如何匹配呢？通过compatible属性来进行匹配  

`reg = <地址1 偏移量1 地址2 偏移量2 地址3 偏移量3，...，>; ` 专门描述寄存器信息的。  
在属性reg中地址是必须存在的，偏移量不一定非得存在。  

```C
#address-cells = <数字>;
#size-cells = <数字>;
adress代表地址
size代表了地址偏移量

cell 代表了32位无符号整数
```
这里没有严格要求，一般都会出现不符合条件的情况。  

例如:  
```c
#address-cells = <2>;
#size-cells = <1>;
当前节点的子节点中有2个地址，1个偏移量

节点1{
	#address-cells = <1>;
	#size-cells = <1>;

	节点2{
		reg = <地址1 偏移量1 地址2>;//子节点有能力修改reg中地址以及偏移量的个数
	};
}

节点{

	xxx:子节点1{
		一堆属性;
	};

	子节点2{
		属性 = <&xxx>;子节点2通过标号xxx引用了子节点1中的属性。
	}；
};

```

## 内核的编译
```c
make uImage
1、在顶层目录寻找uImage目标
	487 ifeq ($(mixed-targets),1) 条件不成立
	492 %:: FORCE
	493     $(Q)$(MAKE) -C $(srctree) KBUILD_SRC= $@

2、有可能引用了其他路径的Makefile文件
	504 include $(srctree)/arch/$(SRCARCH)/Makefile

3、进入arch/arm/Makefile寻找uImage目标文件
	299 BOOT_TARGETS    = zImage Image xipImage bootpImage uImage   
	304 $(BOOT_TARGETS): vmlinux(顶层目录下的)   
	305     $(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
	uImage可以存放在BOOT_TARGETS变量内作为目标文件。
	如果要生成uImage先生成vmlinux

4、在顶层目录的Makefile中寻找vmlinux目标
	 817 vmlinux: scripts/link-vmlinux.sh $(vmlinux-deps) FORCE

5、	 305     $(Q)$(MAKE) $(build)=$(boot) MACHINE=$(MACHINE) $(boot)/$@
	 boot = arch/arm/boot

6、进入到arch/arm/boot/Makefile寻找uImage目标 	 
	 78 $(obj)/uImage:  $(obj)/zImage FORCE
	 79     @$(check_for_multiple_loadaddr)
	 80     $(call if_changed,uimage)

7、先生成zImage，所以寻找zImage目标
	 54 $(obj)/zImage:  $(obj)/compressed/vmlinux FORCE                                                                           
	 55     $(call if_changed,objcopy)

8、必须要先生成arch/arm/boot/compressed/vmlinux,所以进入arch/arm/boot/compressed/Makefile寻找vmlinux目标
	185 $(obj)/vmlinux: $(obj)/vmlinux.lds $(obj)/$(HEAD) $(obj)/piggy.$(suffix_y).o \                                            
	186         $(addprefix $(obj)/, $(OBJS)) $(lib1funcs) $(ashldi3) \
	187         $(bswapsdi2) FORCE
	188     @$(check_for_multiple_zreladdr)
	189     $(call if_changed,ld)
	190     @$(check_for_bad_syms)

	 25 HEAD    = head.o
	 26 OBJS = misc.o decompress.o  

	 86 suffix_$(CONFIG_KERNEL_GZIP) = gzip 这就是gzip压缩命令

	 195 $(obj)/piggy.$(suffix_y).o:  $(obj)/piggy.$(suffix_y) FORCE

	 192 $(obj)/piggy.$(suffix_y): $(obj)/../Image FORCE
	 193     $(call if_changed,$(suffix_y))
	 生成piggy.gzip的条件是以Image文件为源文件调用gzip来进行压缩

	 压缩前要事先生成Image文件
	 47 $(obj)/Image: vmlinux FORCE
	 48     $(call if_changed,objcopy)
	 生成Image文件要依赖顶层目录下的vmlinux

9、调用gzip命令对Image压缩最终生成piggy.gzip.o文件，结合其他的.o文件调用链接器最终生成 arch/arm/boot/compressed/vmlinux

10、回到arch/arm/boot目录下调用objcopy以vmlinux为源文件生成zImage

11、在arch/arm/boot目录下调用uimage来生成uImage镜像文件。

```

## 内核的启动
在arch/arm/kernel/head.S内核启动入口
```c

1、设置特权模式、屏蔽所有中断
2、121     bl  __vet_atags  内核判断u-boot传递的参数类型(是不是传递了设备树)
3、128     bl  __create_page_tables 创建页表
4、144 1:  b   __enable_mmu 初始化MMU
					||
					\/
5、444     b   __turn_mmu_on 使能MMU

6、137     ldr r13, =__mmap_switched 跳转到 __mmap_switched标号地址处

7、104     b   start_kernel

```
