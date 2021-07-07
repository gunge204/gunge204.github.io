# 嵌入式Linux学习笔记（7）Linux内核和根文件系统移植的几个知识点

>作者：**80后老男孩**  
>写作时间：2021-04-27 
>本文已发表于csdn博客：[嵌入式Linux学习笔记（7）Linux内核和根文件系统移植的几个知识点](https://blog.csdn.net/weixin_41034400/article/details/116191814) 

注：本篇笔记参考《第三十五章 Linux内核顶层Makefile详解》、《第三十六章 Linux内核启动流程》、《第三十七章 Linux内核移植》、《第三十八章 根文件系统构建》

## linux内核移植的步骤

①、在 Linux内核中查找可以参考的板子，一般都是半导体厂商自己做的开发板。
②、编译出参考板子对应的 zImage和 .dtb文件。  
③、使用参考板子的 zImage文件和 .dtb文件在我们所使用的板子上启动 Linux内核，看能否启动。
④、如果能启动的 话就万事大吉，如果不能启动那就悲剧了，需要调试 Linux内核。不过一般都会参考半导体官方的开发板设计自己的硬件，所以大部分情况下都会启动起来。启动Linux内核用到的外设不多，一般就 DRAM(Uboot都初始化好的 )和串口。作为终端使用的串口一般都会参考半导体厂商的 Demo板。  
⑤ 、修改相应的驱动，像 NAND Flash、 EMMC、 SD卡等驱动官方的 Linux内核都是已经提供好了，基本不会出问题。重点是网络驱动，因为 Linux驱动开发一般都要通过网络调试代码，所以一定要确保网络驱动工作正常。如果是处理器内部 MAC+外部 PHY这种网络方案的话，一般网络驱动都很好处理，因为在 Linux内核中是有外部 PHY通用驱动的。只要设置好复位引脚、 PHY地址信息基本上都可以驱动起来。  
⑥、Linux内核启动以后需要根文件系统，如果没有根文件系统的话肯定会崩溃，所以确定 Linux内核移植成功以后就要开始根文件系统的构建。  

Linux的编译过程基本和 uboot一样，都要先执行“ make xxx_defconfig”来配置一下，然后在执行“ make”进行编译。如果需要使用图形界面配置的话就执行 make menuconfig”。

**当执行了 make clean后.config并不会被清除，但是执行了 make distclean后.config就会被清除，如果你以前使用了图形配置，那么就白干了，全没了**

## Linux内核的启动

start_kernel通过调用众多的子函数来完成 Linux启动之前的一些初始化工作。

rest_init函数定义在文件 init/main.c中：  
调用函数 kernel_thread创建 kernel_init进程 ，也就是大名鼎鼎的 init内核进程。
init进程的 PID为 1。 init进程一开始是内核进程 (也就是运行在内核态 )，后面 init进程会在根文件系统中查找名为“ init”这个程序，这个 init程序处于用户态，通过运行这个 init程序， init进程就会实现从内核态到用户态的转变。  
调用函数 kernel_thread创建 kthreadd内核进程，此内核进程的 PID为 2。kthreadd进程负责所有内核进程的调度和管理。

在Linux终端中输入“ps -a”就可以打印出当前系统中的所有进程

kernel_init进程函数的定义也在 init/main.c中：  
如果存在“ “/init”程序的话就通过函数 run_init_process来运行此程序。
如果 ramdisk_execute_command为空的话就看 execute_command是否为空，反
正不管如何一定要在根文件系统中找到一个可运行的 init程序。execute_command的值是通过uboot传递，**在 bootargs中使用“ init=xxxx”就可以了，比如init=/linuxrc”表示根文件系统中的 linuxrc就是要执行的用户空间 init程序**。
如果 ramdisk_execute_command和 execute_command都为空，那么就依次
查找“ “/sbin/init”、 “/etc/init”、 “/bin/init”和 “/bin/sh”，这四个相当于备用 init 程序，如果这四个也不存在，那么 Linux启动失败！
如果以上步骤都没有找到用户空间的 init程序，那么就提示错误发生！

调用函数 prepare_namespace来挂载根文件系统。根文件系统也是由命令行参
数指定的，也就是 uboot的 bootargs环境变量。**比如“ root=/dev/mmcblk1p2 rootwait rw”就表示根文件系统在 /dev/mmcblk1p2中，也就是 EMMC的分区 2中。**

## 根文件系统的移植和编译

Linux中的根文件系统更像是一个文件夹或者叫做目录 (在我看来就是一个文件夹，只不过是特殊的文件夹 )，在这个目录里面会有很多的子目录。根目录下和子目录中会有很多的文件，这些文件是 Linux运行所必须的，比如库、常用的软件和命令、设备文件、配置文件等等。

根文件系统是 Linux内核启动以后挂载 (mount)的第一个文件系统，然后从根文件系统中读取初始化脚本，比如 rcS inittab等。根文件系统和 Linux内核是分开的，单独的 Linux内核是没法正常工作的，必须要搭配根文件系统。如果不提供根文件系统， Linux内核在启动的时候就会提示内核崩溃 (Kernel panic)的提示。

### busybox的编译

选项“Build static binary (no shared libs)”用来决定是静态编译 busybox还是动态编译，静态编译的话就不需要库文件，但是编译出来的库会很大。动态编译的话要求根文件系统中有库文件，但是编译出来的 busybox会小很多。这里我们不能采用静态编译！

## 用户开发板调试

当uboot、kernel、rootfs编译完成没有问题了以后，我们就可以将uboot和kernel、dtb烧写进去emmc或者sd卡，然后开发板采用nfs远程登录ubuntu主机的方式启动rootfs，这是产品出于调试阶段的通用做法，要达到次目的，需要再uboot阶段设置bootargs参数（前提是在uboot里已经设置好了网络相关的环境变量）：  
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.1.250: /home/zuozhongkai/linux/nfs/rootfs,proto=tcp rw ip=192.168.1.251:192.168.1.250:192.168.1.1: 255.255.255.0::eth0:off' //设置 bootargs   
saveenv //保存环境变量  
用这种方法，我们将本机ip设为192.168.1.251，远程主机设为nfsroot=192.168.1.250，将根文件系统放在了192.168.1.250: /home/zuozhongkai/linux/nfs/rootfs,那么每次启动后都会直接去ubuntu主机寻找我们的rootfs加载启动，我们直接在主机上修改用户程序而不必再费事下载到板子上即可执行了。



