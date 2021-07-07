# 嵌入式Linux学习笔记（8）Arm Linux驱动开发的几个知识点

>作者：**80后老男孩**  
>写作时间：2021-05-06 
>本文已发表于csdn博客：[嵌入式Linux学习笔记（8）Arm Linux驱动开发的几个知识点](https://blog.csdn.net/weixin_41034400/article/details/116449536) 

注：本篇笔记参考《Arm Linux驱动开发》

## Linux驱动的分类

 Linux中的三大类驱动：字符设备驱动、块设备驱动和网络设备驱动。

其中字符设备驱动是占用篇幅最大的一类驱动，因为字符设备最多，从最简单的点灯到 I2C、 SPI、音频等都属于字符设备驱动的类型。

块设备和网络设备驱动要比字符设备驱动复杂，就是因为其复杂所以半导体厂商一般都给我们编写好了，大多数情况下都 是直接可以使用的。

所谓的块设备驱动就是存储器设备的驱动，比如 EMMC、 NAND、 SD卡和 U盘等存储设备，因为这些存储设备的特点是以存储块为基础，因此叫做块设备。

网络设备驱动就更好理解了，就是网络驱动，不管是有线的还是无线的，都属于网络设备驱动的范畴。一个设备可以属于多种设备驱动类型，比如 USB WIFI，其使用 USB接口，所以属于字符设备，但是其又能上网，所以也属于网络设备驱动。

在Linux 中一切皆为文件，驱动加载成功以后会在“/dev”目录下生成一个相应的文件，应用程序通过对这个名为“/dev/xxx”(xxx 是具体的驱动文件名字)的文件进行相应的操作即可实现对硬件的操作。比如现在有个叫做/dev/led 的驱动文件，此文件是led 灯的驱动文件。应用程序使用open 函数来打开文件/dev/led，使用完成以后使用close 函数关闭/dev/led 这个文件。open和close 就是打开和关闭led 驱动的函数，如果要点亮或关闭led，那么就使用write 函数来操作，也就是向此驱动写入数据，这个数据就是要关闭还是要打开led 的控制参数。如果要获取led 灯的状态，就用read 函数从驱动中读取相应的状态。

## 驱动开发中的常用命令

我们移植好三大系统（uboot、kernel、rootfs），配置好调试环境（从nfs启动rootfs）后就可以进入驱动开发阶段了，开发调试阶段的驱动不编译进内核，而是编译成模块的形式进行测试，最后没问题了才能编译进内核直接使用。

下载编译好的驱动文件（xxx.ko）和用户测试app后到目标根文件系统下的/lib/modules/4.1.5目录下，然后我们就可以加载模块和测试app了。需要用到以下一些常用命令：

depmod：加载前需要先生成modules.dep  
modprobe xxx.ko：加载驱动模块，加载后我们可以通过“cat /proc/devices”命令查看是否已经有了我们加载成功的驱动  
rmmod xxx.ko：卸载驱动模块  

## Linux设备树

高版本的liunux支持使用设备树的方式做驱动开发

在内核的arch/arm/boot/dts下保存着我们的xxx.dts文件，其中我们的xxx.dts文件中会有“include xxx.dtsi”的包含，dtsi文件是半导体厂商提供的跟芯片有关的所有设备树内容，其他的dts文件都要包含他，所以不要改动该dtsi文件。

如果我们使用 I.MX6ULL新做了一个板子，只需要新建一个此板子对应的 .dts文件，然后将对应的 .dtb文件名添加到 arch/arm/boot/dts/Makefile的dtb-$(CONFIG_SOC_IMX6ULL)下，这样在编译设备树的时候就会将对应的 .dts编译为二进制的 .dtb文件。

设备树编译命令是make dtbs

## pinctrl和gpio子系统

Linux内核提供了 pinctrl和 gpio子系统用于GPIO驱动。

pinctrl子系统主要工作内容如下：  
①、获取设备树中 pin信 息。  
②、根据获取到的 pin信息来设置 pin的复用功能  
③、根据获取到的 pin信息来设置 pin的电气特性，比如上 /下拉、速度、驱动能力等。  
对于我们使用者来讲，只需要在设备树里面设置好某个 pin的相关属性即可，其他的初始化工作均由 pinctrl子系统来完成， pinctrl子系统源码目录为drivers/pinctrl。  

pinctrl子系统重点是设置 PIN(有的 SOC叫做 PAD)的复用和电气属性，如果 pinctrl子系统将一个 PIN复用为 GPIO的话，那么接下来就要用到 gpio子系统了。 gpio子系统顾名思义，就是用于初始化 GPIO并且提供相应的 API函数，比如设置 GPIO为输入输出，读取 GPIO的值等。 gpio子系统的主要目的就是方便驱动开发者使用 gpio，驱动开发者在设备树中添加 gpio相关信息，然后就可以在驱动程序中使用 gpio子系统提供的 API函数来操作 GPIO Linux内核向驱动开发者屏蔽掉了 GPIO的设置过程，极大的方便了驱动开发者使用 GPIO。

一个例子

1、添加 pinctrl节点
I.MX6U-ALPHA开发板上的 LED灯使用了 GPIO1_IO03这个 PIN，打开 imx6ull-alientek-emmc.dts，在 iomuxc节点的 imx6ul-evk子节点下创建一个名为“ pinctrl_led”的子节点，节点
内容如下所示：
示例代码45.4.1.1 GPIO1_IO03 pinctrl节点

```
pinctrl_led: ledgrp { 
fsl,pins = < 
MX6UL_PAD_GPIO1_IO03__GPIO1_IO03 0x10B0 >; 
}; 
```

2、添加 LED设备节点
在根节点“ “/”下创建 LED灯节点，节点名为“ gpioled”，节点内容如下

```
gpioled { 
 #address-cells = <1>; 
 #size-cells = <1>; 
 compatible = "atkalpha-gpioled"; 
 pinctrl-names = "default"; 
 pinctrl-0 = <&pinctrl_led>; 
 led-gpio = <&gpio1 3 GPIO_ACTIVE_LOW>; 
 status = "okay"; 
}
```

**注：写完了设备树一定要检查pin是否被其他外设使用！！**

然后make dtbs命令，重新下载设备树文件到开发板，则会看到生成的/dev/gpioled，并在/proc/devicetree/base下看到根节点的内容

## Linx驱动中的其他一些注意点

1、必须解决并发与竞争问题，防止多个应用程序或者多个线程同时使用某个驱动。原子操作、自旋锁、信号量各自适合的应用场合有所不同。

2、Linux内核定时器，Linux使用的系统节拍默认100Hz，在menuconfig中配置-> Kernel Features -> Timer frequency可以看到其配置。在使用内核定时器的时候要注意一点，内核定时器并不是周期性运行的，超时以后就会自动关闭，因此如果想要实现周期性定时，那么就需要在定时处理函数中重新开启定时器。

3、Linux的阻塞和非阻塞IO：这里的“IO”并不是我们学习STM32 或者其他单片机的时候所说的“GPIO”(也就是引脚)。这里的IO 指的是Input/Output，也就是输入/输出，是应用程序对驱动设备的输入/输出操作。当应用程序对设备驱动进行操作的时候，如果不能获取到设备资源，那么阻塞式IO 就会将应用程序对应的线程挂起，直到设备资源可以获取为止。对于非阻塞IO，应用程序对应的线程不会挂起，它要么一直轮询等待，直到设备资源可以使用，要么就直接放弃。**默认的打开文件使用的都是阻塞IO**：

fd = open("/dev/xxx_dev", O_RDWR); /* 阻塞方式打开 */
……  
fd = open("/dev/xxx_dev", O_RDWR | O_NONBLOCK); /* 非阻塞方式打开 */

4、设备树下的platform驱动：latform驱动框架分为总线、设备和驱动，其中总线不需要我们这些驱动程序员去管理，这个是 Linux内核提供的，我们在编写驱动的时候只要关注于设备和驱动的具体实现即可。在 没有设备树的 Linux内核下，我们需要分别编写并 注册 platform_device和 platform_driver，分别代表设备和驱动。在使用设备树的时候，设备的描述被放到了设备树中，因此 platform_device就不需要我们去编写了，我们只需要实现 platform_driver即可。编写 platform驱动的时候要注意兼容属性compatible，因为 platform总线需要通过设备节点的 compatible属性值来 匹配驱动！

5、Linux自带一些常用的简单驱动，比如LED和按键输入驱动，我们只需要打开其内核驱动，并设置好设备树即可直接使用。

6、Linux input子系统，按键、鼠标、键盘、触摸屏等都属于输入 (input)设备， Linux内核为此专门做了一个叫做 input子系统的框架来处理输入事件。输入设备本质上还是字符设备，只是在此基础上套上了 input框架，用户只需要负责上报输入事件，比如按键值、坐标等信息， input核心层负责处理这些事件。注意我们获取的键值（inputenent.code）的定义可以参考系统的/usr/include/linux/input-event-coder中对各种键值的定义，比如KEY_0键值11，KEY_ENTER（回车键）键值28

