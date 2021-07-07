# 嵌入式Linux学习笔记（6）关于uboot移植的几个知识点

>作者：**80后老男孩**  
>写作时间：2021-04-21 
>本文已发表于csdn博客：[嵌入式Linux学习笔记（6）关于uboot移植的几个知识点](https://blog.csdn.net/weixin_41034400/article/details/115939648)  

## U-Boot的由来
Linux系统要启动就必须需要一个 bootloader程序，也就说芯片上电以后先运行一段bootloader程序。这段 bootloader程序会先初始化 DDR等外设，然后将 Linux内核从 flash(NAND NOR FLASH SD MMC等 )拷贝到 DDR中，最后启动 Linux内核。当然了， bootloader的实际工作要复杂的多，但是它最主要的工作就是启动 Linux内核， bootloader和 Linux内核的关系就跟 PC上的 BIOS和 Windows的关系一样， bootloader就相当于 BIOS。所以我们要先搞定bootloader，很庆幸，有很多现成的 bootloader软件可以使用，比如 U-Boot、 vivi、 RedBoot等等，其中以 U-Boot使用最 为广泛。

uboot的全称是 Universal Boot Loader uboot是一个遵循 GPL协议的开源软件， uboot是一个裸机代码，可以看作是一个裸机综合例程。现在的 uboot已经支持液晶屏、网络、 USB等高级功能。 uboot官网为 http://www.denx.de/wiki/U-BOOT/

但是我们一般不会直接用 uboot官方的 U-Boot源码的。 uboot官方的 uboot源码是给半导体厂商准备的，半导体厂商会下载 uboot官方的 uboot源码，然后将自家相应的芯片移植进去。也就是说半导体厂商会自己维护一个版本的 uboot，这个版本的 uboot相当于是他们定制的。既然是定制的，那么肯定对自家的芯片支持会很全，虽然 uboot官网的源码中一般也会支持他们的芯片，但是绝对是没有半导体厂商自己维护的 uboot全面。NXP就维护 的 2016.03这个版本的 uboot，下载地址为http://git.freescale.com/git/cgit.cgi/imx/uboot-imx.git/tag/?h=imx_v2016.03_4.1.15_2.0.0_ga&id=rel_imx_4.1.15_2.1.0_ga

**一般arm芯片的内置程序会将uboot读到DRAM内存的靠后位置，为启动linux内核留出前边足够的内存空间，当内核启动后，uboot也就寿终正寝了，空间释放为内核所用。**

## .config配置文件

uboot是可以配置的，这里设置配置文件为 .config，.config默认是没有的，需要使用命令“ make xxx_defconfig对 uboot进行配置，配置完成以后就会在 uboot根目录下生成 .config。默认情况下 .config和xxx_defconfig内容是一样的，因为 .config就是从 xxx_defconfig复制 过来的。如果后续自行调整了 uboot的一些配置参数，那么这些新的配置参数就添加到了 .config中，而不是 xxx_defconfig。相当于 xxx_defconfig只是一些初始配置，而 .config里面的才是实时有效的配置。

**当执行了 make clean后.config并不会被清除，但是执行了 make distclean后.config就会被清除，如果你以前使用了图形配置，那么就白干了，全没了。**

## BSP包

uboot的移植并不是说我们完完全全的从零开始将 uboot移植到我们现在所使用的开发板或者开发平台上。这个对于我们来说基本是不可能的，这个工作一般是半导体厂商做的， 半导体厂商负责将 uboot移植到他们的芯片上，因此半导体厂商都会自己做一个开发板，这个开发板就叫做原厂开发板，比如大家学习 STM32的时候听说过的 discover开发板就是 ST自己做的。半导体厂商会将 uboot移植到他们自己的原厂开发板上，测试好以后就会将这个 uboot发布出去，这就是大家常说的原厂 BSP包。我们一般做产品的时候就会参考原厂的开发板做硬件，然后在原厂提供的 BSP包上做修改，将 uboot或者 linux kernel移植到我们的硬件上。这个就是uboot移植的一般流程：  
①、在 uboot中找到 参考的开发平台，一般是原厂的开发板。  
②、参考原厂开发板移植 uboot到我们所使用的开发板上。   

实际上半导体厂商会移植uboot、kernel、rootfs，都搞一遍，我们下载下来，在它的基础上根据我们的开发板修改即可。

## 环境变量bootcmd

bootcmd即boot和command，启动命令，bootcmd保存着 uboot默认命令， uboot倒计时结束以后就会执行 bootcmd中的命令。这些命令一般都是用来启动 Linux内核的，比如读取 EMMC或者 NAND Flash中的 Linux内核镜像文件和设备树文件到 DRAM中，然后启动 Linux内核。可以在 uboot启动以后进入命令行设置 bootcmd环境变量的值。如果 EMMC或者 NAND中没有保存 bootcmd的值，那么 uboot就会使用默认的值，板子第一次运行 uboot的时候都会使用默认值来设置 bootcmd环境变量。比如：

```
mmc dev 1 //切换到 EMMC 
fatload mmc 1:1 0x80800000 zImage //读取 zImage到 0x80800000处
fatload mmc 1:1 0x83000000 imx6ull-14x14-evk.dtb //读取设备树到 0x83000000处
bootz 0x80800000 - 0x83000000 //启动 Linux
```

bootcmd的内容可以顺着打开文件 include/env_default.h中的定义去查找，文件 mx6ull_alientek_emmc.h（在include/configs文件夹下）自定义中的宏CONFIG_EXTRA_ENV_SETTINGS保存着这些环境变量的默认值.

## 环境变量bootargs

bootargs即boot和args，保存着 uboot传递给 Linux内核的参数，uboot在最后启动kernel后，由kernel继承这些参数。

bootargs常设置的选项就三个。同样，mx6ull_alientek_emmc.h（在include/configs文件夹下）自定义中的宏CONFIG_EXTRA_ENV_SETTINGS保存着这些环境变量的默认值。

bootargs环境变量是由 mmcargs设置的， mmcargs环境变量如下：  
mmcargs=setenv bootargs console=${console},${baudrate} root=${mmcroot}  
其中  
console=ttymxc0 baudrate=115200 mmcroot=/dev/mmcblk1p2 rootwait rw，因此将mmcargs展开以后就是：  
mmcargs=setenv bootargs console= ttymxc0, 115200 root= /dev/mmcblk1p2 rootwait rw  

1、 console  
console用来设置 linux终端 (或者叫控制台 )，也就是通过什么设备来和 Linux进行交互，是串口还是 LCD屏幕？如果是串口的话应该是串口几等等。一般设置串口作为 Linux终端，这样我们就可以在电脑上通过 SecureCRT来和 linux交互了。这里设置 console为 ttymxc0，因为 linux启动以后 I.MX6ULL的串口 1在 linux下的设备文件就是 /dev/ttymxc0，在 Linux下，一切皆文件。
ttymxc0后面有个“,115200”，这是设置串口的波特率 console=ttymxc0,115200综合起来就是设置 ttymxc0（也就是串口1）作为 Linux的终端，并且串口波特率设置为 115200。  

2、 root  
root用来设置根文件系统的位置， root=/dev/mmcblk1p2用于指明根文件系统存放在mmcblk1设备的分区 2中。 EMMC版本的核心板启动 linux以后会存在 /dev/mmcblk0、/dev/mmcblk1、 /dev/mmcblk0p1、 /dev/mmcblk0p2、 /dev/mmcblk1p1和 /dev/mmcblk1p2这样的文件，其中 /dev/mmcblkx(x=0~n)表示 mmc设备，而 /dev/mmcblkxpy(x=0~n,y=1~n)表示 mmc设备x的分区 y。在 I.MX6U-ALPHA开发板中 /dev/mmcblk1表示 EMMC，而 /dev/mmcblk1p2表示EMMC的分区2。  
root后面有“ rootwait rw rootwait表示等待 mmc设备初始化完成以后再挂载，否则的话mmc设备还没初始化完成就挂载根文件系统会出错的。 rw表示根文件系统是可以读写的，不加rw的话可能无法在根文件系统中进行写操作，只能进行读操作。

3、 rootfstype    
此选项一般配置 root一起使用， rootfstype用于指定根文件系统类型，如果根文件系统为ext格式的话此选项无所谓。如果根文件系统是 yaffs、 jffs或 ubifs的话就需要设置此选项，指定根文件系统的类型。

## uboot移植过程

简单总结一下 uboot移植的过程：  
①、不管是购买的开发板还是自己做的开发板，基本都是参考半导体厂商的dmeo板，而半导体厂商会在他们自己的开发板上移植好 uboot、 linux kernel和 rootfs等，最终制作好 BSP包提供给用户。我们可以在官方提供的 BSP包的基础上添加我们的板子，也就是俗称的移植。  
②、我们购买的开发板或者自己做的板子一般都不会原封不动的照抄半导体厂商的demo板，都会根据实际的情况来做修改，既然有修改就必然涉及到 uboot下驱动的移植。  
③、一般uboot中需要解决串口、 NAND、 EMMC或 SD卡、网络和 LCD驱动 ，因为 uboot的主要目的就是启动 Linux内核，所以不需要考虑太多的外设驱动。  
④、在uboot中添加自己的板子信息，根据自己板子的实际情况来修改 uboot中的驱动。



