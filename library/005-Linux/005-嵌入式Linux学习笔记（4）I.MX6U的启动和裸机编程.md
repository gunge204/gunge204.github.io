# 嵌入式Linux学习笔记（4）I.MX6U的启动和裸机编程

>作者：**80后老男孩**  
>写作时间：2021-04-10  
>本文已发表于csdn博客：[嵌入式Linux学习笔记（4）I.MX6U的启动和裸机编程](https://blog.csdn.net/weixin_41034400/article/details/115611490)  

[TOC]

**注：本章的笔记是从一个单片机开发人员的角度来观察arm芯片是如何启动起来的、用户代码是如何编辑、编译和烧写进去的。笔记综合了开发板教程第三章到第十三章的内容，所有笔记都能找到对应的章节，如有不明则去搜索查看**

以下内容先用一张流程图概括，包括硬件启动，以及如何编写、编译、烧写程序到SD的基本流程。  
![](assets/005/005-1618058268631.png)

## I.MX6U的启动方式

I.MX6U支持多种启动方式以及启动设备，比如可以从 SD/EMMC、 NAND Flash、 QSPI Flash等启动。用户可以根据实际情况，选择合适的启动设备。不同的启动方式其启动方式和启动要求也不一样，下边说的imxdownload工具的作用是从 SD卡启动就需要在 bin文件前面添加一个数据头，其它的启动设备也是需要这个数据头的，那么相应的imxdownload工具可能就需要变化。

### 启动方式的选择

BOOT的处理过程是发生在 I.MX6U芯片上电以后，芯片会根据 BOOT_MODE[1:0]的设置来选择 BOOT方式。      BOOT_MODE[1:0]的值是可以改变的，有两种方式，一种是改写 eFUSE(熔丝 )，一种是修改相应的 GPIO高低电平。第一种修改 eFUSE的方式只能修改一次，后面就不能再修改了，所以我们不使用。我们使用的是通过修改 BOOT_MODE[1:0]对应的 GPIO高低电平来选择启动方式，所有的开发板都使用的这种方式， I.MX6U有一个 BOOT_MODE1引脚和BOOT_MODE0引脚，这两个引脚对应这 BOOT_MODE[1:0]。

基本上我们会用到“串行下载”和“内部BOOT模式”两种启动方式。串行下载模式是让我们通过usb或者串口将程序烧写到板子上的外部存储。

而内部BOOT模式，芯片会执行内部的 boot ROM代码，这段 boot ROM代码会进行硬件初始化 (一部分外设，启动时钟等等 )，然后从 boot设备 (就是存放代码的设备、比如 SD/EMMC、 NAND)中将代码拷贝出来复制到指定的 RAM中，一般是 DDR。

### 启动设备

当 BOOT_MODE设置为内部 BOOT模式以后，可以从以下设备中读取我们的程序来启动：  
① 、接到 EIM接口的 CS0上的 16位 NOR Flash。  
② 、接到 EIM接口的 CS0上的 OneNAND Flash。  
③ 、接到 GPMI接口上的 MLC/SLC NAND Flash NAND Flash页大小支持 2KByte、 4KByte和 8KByte 8位宽。  
④ 、 Quad SPI Flash。  
⑤ 、接到 USDHC接口上的 SD/MMC/eSD/SDXC/eMMC等设备。  
⑥ 、 SPI接口的 EEPROM。  
这些启动设备如何选择呢？ I.MX6U同样提供了 eFUSE和 GPIO配置两种， eFUSE就不讲解了。我们重点看如何通过 GPIO来选择启动设备，因为所有的 I.MX6U开发板都是通过 GPIO来配置启动设备的。正如启动模式由 BOOT_MODE[1:0]来选择一样，启动设备是通过BOOT_CFG1[7:0]、 BOOT_CFG2[7:0]和 BOOT_CFG4[7:0]这 24个配置 IO，这 24个配置 IO刚好对应着 LCD的 24根数据线 LCD_DATA0~LCDDATA23，当启动完成以后这 24个 IO就可以作为 LCD的数据线使用。这 24根线和 BOOT_MODE1、 BOOT_MODE0共同组成了 I.MX6U的启动选择引脚。

## 裸机编程

### 编译流程

GCC编译器的编译流程是：预处理、编译 、汇编和链接。预处理就是展开所有的头文件、替换程序中的宏、解析条件编译并添加到文件中。编译是将经过预编译处理的代码编译成汇编代码，也就是我们常说的程序编译。汇编就是将汇编语言文件编译成二进制目标文件。链接就是将汇编出来的多个二进制目标 文件 链接在一起，形成最终的可执行文件，链接的时候还会涉及到静态库和动态库等问题。

#### 1、 arm-linux-gnueabihf-gcc编译文件  

我们是要编译出在 ARM开发板上运行的可执行文件，所以要**使用交叉编译器 arm-linux-gnueabihf-gcc来编译**。本试验就一个 led.s源文件，所以编译比较简单。先将 led.s编译为对应的 .o文件，在终端中输入如下命令：  
arm-linux-gnueabihf-gcc -g -c led.s -o led.o  
上述命令就是将 led.s编译为 led.o，其中 “-g”选项是产生调试信息 GDB能够使用这些调试信息进行代码调试。“ “-c”选项 是编译源文件，但是不链接。 “-o”选项是指定编译产生的文件名字，这里我们指定 led.s编译完成以后的文件名字为 led.o。

led.o文件并不是我们可以下载到开发板中运行的文件，一个工程中所有的 C文件和汇编文件都会编译生成一个对应的 .o文件，我们需要将这些 .o文件链接起来组合成可执行文件。

#### 2、 arm-linux-gnueabihf-ld链接文件

**arm-linux-gnueabihf-ld用来将众多的 .o文件链接到一个指定的链接位置**。我们在学习SMT32的时候基本就没有听过“链接”这个词，我们一般用 MDK编写好代码，然后点击“编译”， MDK或者 IAR就会自动帮我们编译好整个工程，最后再点击“下载”就可以将代码下载到开发板中。这是因为链接这个操作 MDK或者 IAR已经帮你做好了，打开一个 STM32的工程，然后编译一下，肯定能找到很多 .o文件，这些 .o文件肯定会被 MDK链接到某个地址去。0X08000000就是 STM32内部 ROM的起始地址，编译出来的指令肯定是要从 0X08000000这个地址开始存放的 。对于STM32来说 0X08000000就是它的链接地址，这些 .o文件就是从这个链接地址开始依次存放，最终生成一个可以下载的 hex或者 bin文件，我们可以打开 .map文件查看一下这些文件的链接地址。.map文件就详细的描述了各个 .o文件都是链接到了什么地址。

因此我们现在需要做的就是确定一下本试验最终的可执行文件其运行起始地址，也就是链接地址。这里我们要区分“存储地址”和“运行地址”这两个概念，“存储地址”就是可执行文件存储在哪里，可执行文件的存储地址可以随意选择。“运行地址”就是代码运行的时候所处的地址，这个我们在链接的时候就已经确定好了，代码要运行，那就必须处于运行地址处，否则代码肯定运行出错。比如 I.MX6U支持 SD卡、 EMMC、 NAND启动，因此代码可以存储到 SD卡、 EMMC或者 NAND中，但是要运行的话就必须将代码从 SD卡、 EMMC或者NAND中拷贝到 其运行地址 (链接地址 )处，“存储地址”和“运行地址”可以一样，比如STM32的存储起始地址和运行起始地址都是 0X08000000。

本教程所有的裸机例程都是烧写到 SD卡中，上电以后 I.MX6U的内部 boot rom程序会将可执行文件拷贝到链接地址处，这个链接地址可以在 I.MX6U的内部 128KB RAM中(0X900000~0X91FFFF)，也可以在外部的 DDR中。本教程所有裸机例程的链接地址都在 DDR中，链接起始地址为 0X87800000。 I.MX6U-ALPHA开发板的 DDR容量有两种： 512MB和256MB，起始地址都为 0X80000000，只不过 512MB的终止地址为 0X9FFFFFFF，而 256MB容量的终止地址为 0X8FFFFFFF。之所以选择 0X87800000这个地址是因为后面要讲的 Uboot其链接地址就是 0X87800000，这样我们统一使用 0X87800000这个链接地址，不容易记混。

确定了链接地址以后就可以使用 arm-linux-gnueabihf-ld来将前面编译出来的 led.o文件链接到 0X87800000这个地址，使用如下命令：  
arm-linux-gnueabihf-ld -Ttext 0X87800000 led.o -o led.elf     
上述命令中 -Ttext就是指定链接地址，“-o”选项指定链接生成的 elf文件名，这里我们命名为 led.elf。上述命令执行完以后就会在工程目录下多一个 led.elf文件。

#### 3、 arm-linux-gnueabihf-objcopy格式转换

ed.elf文件也不是我们最终烧写到 SD卡中的可执行文件，我们要烧写的 .bin文件，因此还需要将 led.elf文件转换为 .bin文件，这里我们就需要用到 arm-linux-gnueabihf-objcopy这个工具了。

**arm-linux-gnueabihf-objcopy更像一个格式转换工具，我们需要用它将 led.elf文件转换为led.bin文件**，命令如下：  
arm-linux-gnueabihf-objcopy -O binary -S -g led.elf led.bin   
上述命令中，“ ，“-O”选项指定以什么格式输出，后面的 binary”表示以二进制格式输出选项“ “-S”表示不要复制源文件中的重定位信息和符号信息 ，“-g”表示不复制源文件中的调试信息。

#### 4、 arm-linux-gnueabihf-objdump反汇编

大多数情况下我们都是用 C语言写试验例程的，有时候需要查看其汇编代码来调试代码，因此就需要进行反汇编，一般可以将 elf文件反汇编，比如如下命令：  
arm-linux-gnueabihf-objdump -D led.elf > led.dis   
上述代码中的“ “-D”选项表示反汇编所有的段，反汇编完成以后就会在当前目录下出现一个名为 led.dis文件

总结一下我们为了编译 ARM开发板上运行的 led.s这个文件使用了如下命令：
arm-linux-gnueabihf-gcc -g -c led.s -o led.o   
arm-linux-gnueabihf-ld -Ttext 0X87800000 led.o -o led.elf   
arm-linux-gnueabihf-objcopy -O binary -S -g led.elf led.bin   
arm-linux-gnueabihf-objdump -D led.elf > led.dis  

### 启动代码

类似于stm32开发中每个工程都必然有一个类似于与startup_stm32f10x_hd.s的启动文件，I.MX6U的裸机程序开发也需要一个启动文件，这个文件由汇编语言编写，因为 Cortex-A芯片一上电 SP指针还没初始化， C环境还没准备好，所以肯定不能运行 C代码，必须先用汇编语言设置好 C环境，比如初始化 DDR、设置 SP指针等等，当汇编把 C环境设置好了以后才可以运行 C代码。**启动代码也是用户程序的一部分，是在main之前最开始的那部分。**

下面是开发板的启动代码及注释：

```assembly
.global _start  		/* 全局标号 */
/*
 * 描述：	_start函数，程序从此函数开始执行，此函数主要功能是设置C运行环境。
*/
_start:		/*定义一个全局标号_start */

	/* 进入SVC模式（超级管理员模式） */
	mrs r0, cpsr
	bic r0, r0, #0x1f 	/* 将r0寄存器中的低5位清零，也就是cpsr的M0~M4 	*/
	orr r0, r0, #0x13 	/* r0或上0x13,表示使用SVC模式					*/
	msr cpsr, r0		/* 将r0 的数据写入到cpsr_c中 					*/

	/* 设置栈指针，
	 * 注意：IMX6UL的堆栈是向下增长的！
	 * 堆栈指针地址一定要是4字节地址对齐的！！！
	 * DDR范围:0X80000000~0X9FFFFFFF（512M)或者0X80000000~0X8FFFFFFF（256M)
	 * 0x80200000-0x80000000=0x200000=2MB，由于堆栈向下增长，固一共2MB字节
	 */

	ldr sp,=0X80200000	/* 设置用户模式下的栈首地址为0X80200000,大小为2MB	  	  */
	b main				/* 跳转到main函数 										*/
```

至此汇编部分程序执行完成，就几行代码，用来设置处理器运行到 SVC模式下、然后初始化 SP指针、最终跳转到 C文件的 main函数中。如果有玩过三星的 S3C2440或者 S5PV210的话会知道我们在使用 SDRAM或者 DDR之前必须先初始化 SDRAM或者 DDR。所以 S3C2440或者 S5PV210的汇编文件里面是一定会有 SDRAM或者 DDR初始化代码的。我们上面编写的start.s文件中却没有初始化 DDR3的代码，但是却将 SVC模式下的 SP指针设置到了 DDR3的地址范围中，这不会出问题吗？肯定不会的， **DDR3肯定是要初始化的，但是不需要在 start.s文件中完成。因为在后边生成最终的可烧写的.imx文件时，会在用户代码之前增加DCD数据部分，  DCD数据包含了 DDR配置参数**， I.MX6U内部的 Boot ROM会读取 DCD数据 中的 DDR配置 参数然后完成 DDR初始化的。

### Makefile文件

在项目规模巨大文件很多的时候，为便于处理以上述的编译流程，需编写Makefile文件，作为工程编译的脚本，下面是开发板的通用Makefile文件及注释（Makefile中没有直接使用链接命令，而是已经使用下边要讲的连接脚本的操作形式，本质上是一样的）：

```makefile
#编译器的名字
CROSS_COMPILE 	?= arm-linux-gnueabihf-
#目标文件名字，各项目自己修改
TARGET		  	?= bsp
#编译命令为arm-linux-gnueabihf-gcc，用于生成.o
CC 				:= $(CROSS_COMPILE)gcc
#链接命令为arm-linux-gnueabihf-ld，用于生成.elf
LD				:= $(CROSS_COMPILE)ld
#格式转换命令为arm-linux-gnueabihf-objcopy，用于生成.bin
OBJCOPY 		:= $(CROSS_COMPILE)objcopy
#反汇编命令为arm-linux-gnueabihf-objdump，用于生成.dis
OBJDUMP 		:= $(CROSS_COMPILE)objdump
#以下是整个工程的所有.h头文件目录，根据实际情况修改添加
INCDIRS 		:= imx6ul \
				   bsp/clk \
				   bsp/led \
				   bsp/delay 
#以下是整个工程的所有.c和.s源文件目录，根据实际情况修改添加				   			   
SRCDIRS			:= project \
				   bsp/clk \
				   bsp/led \
				   bsp/delay 				   
#通过函数patsubst给变量INCDIRS下描述的所有头文件目录都加上“-I”	，这是MakeFIle语法要求			   
INCLUDE			:= $(patsubst %, -I %, $(INCDIRS))
#保存工程中所有的.s汇编文件和.c文件（含绝对路径）
SFILES			:= $(foreach dir, $(SRCDIRS), $(wildcard $(dir)/*.S))
CFILES			:= $(foreach dir, $(SRCDIRS), $(wildcard $(dir)/*.c))
#保存工程中所有的.s汇编文件和.c文件，只有文件名本身，不含绝对路径
SFILENDIR		:= $(notdir  $(SFILES))
CFILENDIR		:= $(notdir  $(CFILES))
#.S和 .c文件编译以后对应的 .o文件目录，这里让所有.o文件都放到同一个叫obj文件夹下
SOBJS			:= $(patsubst %, obj/%, $(SFILENDIR:.S=.o))
COBJS			:= $(patsubst %, obj/%, $(CFILENDIR:.c=.o))
OBJS			:= $(SOBJS) $(COBJS)
#文件搜索目录，这里等同于SCDIRS
VPATH			:= $(SRCDIRS)
#指定一个伪目标clean，防止文件里边有个叫clean的造成错误
.PHONY: clean
#最终目标是生成.bin，依赖于各个.o文件，3条命令:	用链接脚本链接各个.o文件成.elf，.elf文件格式转换为.bin，.elf文件反汇编成dis文件
$(TARGET).bin : $(OBJS)
	$(LD) -Timx6ul.lds -o $(TARGET).elf $^				
	$(OBJCOPY) -O binary -S $(TARGET).elf $@			
	$(OBJDUMP) -D -m arm $(TARGET).elf > $(TARGET).dis	
#将所有.S文件编译成.o文件
$(SOBJS) : obj/%.o : %.S
	$(CC) -Wall -nostdlib -c -O2  $(INCLUDE) -o $@ $<
#将所有.c文件编译成.o文件
$(COBJS) : obj/%.o : %.c
	$(CC) -Wall -nostdlib -c -O2  $(INCLUDE) -o $@ $<
#清理编译结果	
clean:
	rm -rf $(TARGET).elf $(TARGET).dis $(TARGET).bin $(COBJS) $(SOBJS)	

```

### 链接脚本

在最初的链接操作中我们链接代码的时候使用如下语句：  
arm-linux-gnueabihf-ld -Ttext 0X87800000 -o ledc.elf $^   

上面语句中我们是通过“ -Ttext”来指定链接地址是 0X87800000的，这样的话所有的文件都会链接到以 0X87800000为起始地址的区域。但是有时候我们很多文件需要链接到指定的区域，或者叫做段里面，比如在 Linux里面初始化函数就会放到 init段里面。因此我们需要能够自定义一些段，这些段的起始地址我们可以自由指定，同样的我们也可以指定一个文件或者函数应该存放到哪个段里面去。要完成这个功能我们就需要使用到链接脚本，看名字就知道链接脚本主要用于链接的，用于描述文件应该如何被链接在一起形成最终的可执行文件。其主要目的是描述输入文件中的段如何被映射到输出文件中，并且控制输出文件中的内存排布。比如我们编译生成的文件一般都包含 text段、 data段等等。

链接脚本的语法很简单，就是编写一系列的命令，这些命令组成了链接脚本，每个命令是一个带有参数的关键字或者 一个对符号的赋值，可以使用分号分隔命令。像文件名之类的字符串可以直接键入，也可以使用通配符 “*”。最简单的链接脚本可以只包含一个命令 SECTIONS,我们可以在这一个“ SECTIONS”里面来描述输出文件的内存布局。我们一般编译出来的代码都包含在 text、 data、 bss和 rodata这四个段内。

开发板的链接脚本：

```assembly
SECTIONS{
	. = 0X87800000;
	.text :
	{
		obj/start.o 
		*(.text)
	}
	.rodata ALIGN(4) : {*(.rodata*)}     
	.data ALIGN(4)   : { *(.data) }    
	__bss_start = .;    
	.bss ALIGN(4)  : { *(.bss)  *(COMMON) }    
	__bss_end = .;
}
```

### 镜像烧写

那么有了led.bin这个文件后是不是就直接拷贝到SD上直接让arm运行就可以了呢？不行。我们需要一个镜像烧写的过程，镜像烧写过程的本质是在bin文件的前边再加入一些系统启动需要的内容生成一个全新的load.imx文件，这个文件写入SD卡中，才能被I.MX6U启动时正确读取和执行。

我们学习 STM32等其他的单片机的时候，编译完代码以后可以直接通过 MDK或者 IAR下载到内部的 flash中。但是 I.MX6U虽然内部有 96K的 ROM，但是这 96K的 ROM是 NXP自己用的，不向用户开放。所以相当于说 I.MX6U是没有内部 flash的，但是我们的代码得有地方存放啊，为此， I.MX6U支持从外置的 NOR Flash、 NAND Flash、 SD/EMMC、 SPI NOR Flash和 QSPI Flash这些存储介质中启动，所以我们可以将代码烧写到这些存储介质中中。在这些存储介质中，除了 SD卡以外，其他的一般都是焊接到了板子上的，我们没法直接烧写。但是 SD卡是活动的，是可以从板子上插拔的，我们可以将 SD卡插到电脑上，在电脑上使用软件将 .bin文件烧写到 SD卡中，然后再插到板子上就可以了。其他的几种存储介质是我们量产的时候用到的，量产的时候代码就不可能放到 SD卡里面了，毕竟 SD是活动的，不牢固，而其他的都是焊接到板子上的，很牢固。

因此，我们在调试裸机和 Uboot的时候是将代码下载到 SD中，因为方便嘛，当调 试完成以后量产的时候要将裸机或者 Uboot烧写到 SPI NOR Flash、 EMMC、 NAND等这些存储介质中的。那么，如何将我们前面编译出来的 led.bin烧写到 SD卡中呢？肯定有人会认为直接复制led.bin到 SD卡中不就行了，错！编译出来的可执行文件是怎么存放到 SD中的，存放的位置是什么？这个 NXP是有详细规定的！我们必须按照 NXP的规定来将代码烧写到 SD卡中，否则代码是绝对运行不起来的。

烧写所用工具是imxdownload软件，与bin文件放在同一个文件夹下，Shee中执行命令格式：使用 imxdownload向 SD卡烧写 led.bin文件，命令格式如下：  
./imxdownload <.bin file> <SD Card>  
其中 .bin就是 要烧写的 .bin文件 SD Card就是你要烧写的 SD卡 ，比如我的电脑使用如下命令烧写 led.bin到 /dev/sdd中（有sdd必有sdd1，sdd1是个分区1的意思）  
./imxdownload led.bin /dev/sdd

### imxdownload工具的作用

前面我们设置好 BOOT以后就能从指定的设备启动了，但是你的设备里面得有代码啊，在前面我们使用 imxdownload这个软件将 led.bin烧写到了 SD卡中。 imxdownload会在 led.bin前面添加一些头信息，重新生成一个叫做 load.imx的文件，最终实际烧写的是 laod.imx。那么 imxdownload究竟做了什么？ load.imx和 led.bin究竟是什么关系？本节我们就来详细的讲解一下 imxdownload是如何将 led.bin打包成 load.imx的。

学习 STM32的时候我们可以直接将编译生成的 .bin文件烧写到 STM32内部 flash里面，但是 I.MX6U不能直接烧写编译生成的 .bin文件，我们需要在 .bin文件前面添加一 些头信息构成满足 I.MX6U需求的最终可烧写文件， I.MX6U的最终可烧写文件组成如下：  
① 、 Image vector table，简称 IVT IVT里面包含了一系列的地址信息，这些地址信息在ROM中按照固定的地址存放着。  
② 、 Boot data，启动数据，包含了镜像要拷贝到哪个地址，拷贝的大小是多少等等。  
③ 、 Device configuration data，简称 DCD，设备配置信息，重点是 DDR3的初始化配置。  
④ 、用户代码可执行文件，比如 led.bin。  
可以看出最终烧写到 I.MX6U中 的程序其组成为： IVT+Boot data+DCD+.bin。所以 imxdownload所生成的 load.imx就是在 led.bin前面加上 IVT+Boot data+DCD。内部 Boot ROM会将 load.imx拷贝到 DDR中，用户代码是要一定要从 0X87800000这个地方开始的，因为链接地址为 0X87800000 load.imx在用户代码前面又有 3KByte的 IVT+Boot Data+DCD数据，下面会讲为什么是 3KByte，因此 load.imx在 DDR中的起始地址就是 0X87800000-3072=0X877FF400。

load.imx最前面的就是 IVT和 Boot Data IVT包含了镜像程序的入口点、指向 DCD的指针和一些用作其它用途的指针。内部 Boot ROM要求 IVT应该放到指定的位置，不同的启动设备位置不同，而 IVT在整个 load.imx的最前面，其实就相当于要求 load.imx在烧写的时候应该烧写到存储设备的指定位置去。

以 SD/EMMC为例， IVT偏移为 1Kbyte， IVT+Boot data+DCD的总大小为 4KByte-1KByte=3KByte。假如 SD/EMMC每个扇区为 512字节，那么 load.imx应该从第三个扇区开始烧写，前两个扇区要留出来。 load.imx从第 3KByte开始才是真正的 .bin文件。

复位以后， I.MX6U片内的所有寄存器都会复位为默认值，但是这些默认值往往不是我们想要的值，而且有些外设我们必须在使用之前初始化它。为此 I.MX6U提出了一个 DCD(Device Config Data)的概念，和 IVT、 Boot Data一样， DCD也是添加到 load.imx里面的，紧跟在 IVT和 Boot Data后面， IVT里面也指 定了 DCD的位置。 DCD其实就是 I.MX6U寄存器地址和对应的配置信息集合， Boot ROM会使用这些寄存器地址和配置集合来初始化相应的寄存器，比如开启某些外设的时钟、初始化 DDR等等。 DCD区域不能超过 1768Byte。

**注：imxdownload软件工具是正点原子提供的，其源代码是imxdownload.c和imxdownload.h，我们可以根据自己的实际情况改写后再编译生成自己所需要的烧写工具**。

上边的imxdownload工具的作用是从 SD卡启动就需要在 bin文件前面添加一个数据头，其它的启动设备也是需要这个数据头的，那么相应的imxdownload工具可能就需要变化。如何变化需要参看源码和正点原子开发板教程的《第九章 I.MX6U启动方式详解》



