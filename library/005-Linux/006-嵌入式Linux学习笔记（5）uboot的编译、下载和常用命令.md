# 嵌入式Linux学习笔记（5）uboot的编译、下载和常用命令

>作者：**80后老男孩**  
>写作时间：2021-04-16  
>本文已发表于csdn博客：[嵌入式Linux学习笔记（5）uboot的编译、下载和常用命令](https://blog.csdn.net/weixin_41034400/article/details/115749224)  

[TOC]

注：本次笔记参考以下三篇：  
I.MX6U嵌入式Linux驱动开发指南V1.5，第三十章  
I.MX6U用户快速体验V1.7.2，第四章  
I.MX6U 开发板文件拷贝及固件更新参考手册V1.2，第二章  

## 交叉编译工具
I.MX6U的uboot教程里的交叉编译工具是通用的arm-linux-gnueabihf，给出的编译过程脚本文件是：

```
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_ddr512_emmc_defconfig 
make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j12
```

以上操作可以编译成功，再使用imxdownload工具下载到sd卡，进入uboot后大多出操作也都正常，但是在使用网络功能ping主机的时候造成了超时和看门狗重启。

因此最后改用“I.MX6U用户快速体验V1.7.2，第四章”介绍的Poky 交叉编译工具链arm-poky-linux-gnueabi，用uboot下的build.sh脚本执行编译过程，下载后所有操作成功。

## 下载更新uboot到sd卡的dd命令

符合nxp的规范的下载文件不是.bin文件，是生成的.imx文件，要把它烧写到sd卡上，使用的命令如下：

```
sudo dd if=u-boot-imx6ull-14x14-ddr512-emmc.imx of=/dev/sdb bs=1024 seek=1 conv=fsync
```

至此更新uboot成功，需要注意的是：如果你的SD卡曾经保存过其他uboot环境变量，你需要在SD卡时恢复一次环境变量再保存，才能使用新的环境变量启动！

## 在ubuntu上安装tftp服务

uboot是支持网络的，我们在移植 uboot的时候一般都要调通网络功能，因为在移植 linux kernel的时候需要使用到 uboot的网络功能做调试。 uboot支持大量的网络相关命令，比如 dhcp、ping、 nfs和 tftpboot

nfs(Network File System)网络文件系统，通过 nfs或者tftp可以在计算机之间通过网络来分享资源，比如我们将 linux 镜像和设备树文件放到 Ubuntu中，然后在 uboot中使用 nfs或tftp命令将 Ubuntu中的 linux 镜像和设备树下载到开发板的 DRAM中。

这样做的目的是为了方便调试 linux镜像和设备树，也就是网络调试，通过网络调试是 Linux开发中最常用的调试方法。原因是嵌入式 linux开发不像单片机开发，可以直接通过 JLINK或 STLink等仿真器将代码直接烧写到单片机内部的 flash中，嵌入式 Linux通常是烧写到 EMMC、 NAND Flash、 SPI Flash等外置 flash中，但是嵌入式 Linux开发也没有 MDK IAR这样的 IDE，更没有烧写算法，因此不可能通过点击一个download”按钮就将固件烧写到外部 flash中。虽然半导体厂商一般都会提供一个烧写固件的软件，但是这个软件使用起来比较复杂，这个烧写软件一般用于量产的。其远没有 MDK、 IAR的一键下载方便，在 Linux内核调试阶段，如果用这个烧写软件的话将会非常浪费时间，而这个时候网络调试的优势就显现出来了，可以通过网络将编译好的 linux镜像和设备树文件下载到 DRAM中，然后就可以直接运行。

我们一般使用 uboot中的 nfs或tftp命令将 Ubuntu中的文件下载到开发板的 DRAM。

tftp命令的作用和 nfs命令一样，都是用于通过网络下载东西到 DRAM中，只是 tftp命令使用的 TFTP协议， Ubuntu主机作为 TFTP服务器。因此需要在 Ubuntu上搭建 TFTP服务器，需要安装 tftp-hpa和 tftpd-hpa，命令如下  
sudo apt-get install tftp-hpa tftpd-hpa sudo apt-get install xinetd 

和 NFS一样， TFTP也需要一个文件夹来存放文件，在用户目录下新建一个目录，命令如下：  
mkdir /home/zuozhongkai/linux/tftpboot chmod 777   /home/zuozhongkai/linux/tftpboot   

这样我就在我的电脑上创建了一个名为 tftpboot的目录 (文件夹 )，路径为
/home/zuozhongkai/linux/tftpboot。  

注意！我们要给 tftpboot文件夹权限，否则的话 uboot不能从tftpboot文件夹里面下载文件。  
最后配置 tftp，安装完成以后新建文件 /etc/xinetd.d/tftp 如果没有 /etc/xinetd.d目录的话自行创建， 然后在里面输入如下内容：  

```
server tftp 
 { 
 socket_type = dgram 
 protocol = udp 
 wait = yes 
 user = root 
 server = /usr/sbin/in.tftpd 
 server_args = -s /home/zuozhongkai/linux/tftpboot/ 
 disable = no 
 per_source = 
 11 cps = 100 2 
 flags = IPv4 
 }
```

完了以后启动 tftp服务，命令如下：
sudo service tftpd-hpa start     

打开 /etc/default/tftpd-hpa文件，将其修改为如下所示内容：  

```
# /etc/default/tftpd-hpa 
 
 TFTP_USERNAME="tftp" 
 TFTP_DIRECTORY="/home/zuozhongkai/linux/tftpboot" 
 TFTP_ADDRESS=":69" 
 TFTP_OPTIONS="-l -c -s"
```

TFTP_DIRECTORY就是我们上面创建的 tftp文件夹目录，以后我们就将所有需要通过TFTP传输的文件都放到这个文件夹里面，并且要给予这些文件相应的权限。  
最后输入如下命令， 重启 tftp服务器：  
sudo service tftpd-hpa restart tftp  
服务器已经搭建好了，接下来就是使用了。将 zImage镜像文件拷贝到 tftpboot文件夹  
中，并且给予 zImage相应的权限，命令如下：  
cp zImage /home/zuozhongkai/linux/tftpboot/   
cd   /home/zuozhongkai/linux/tftpboot/   
chmod 777 zImage  

## 常用uboot命令

### help或者？

查看当前uboot支持的所有命令，uboot是可配置的，需要什么命令就在移植的时候使能什么命令。

### bdinfo、printenv、version

查询板子信息、打印环境变量、查询版本信息

### setenv   saveenv

修改环境变量、保存环境变量，例如设置网络相关的环境变量：

```
setenv ipaddr 192.168.1.50 
setenv ethaddr 00:04:9f:04:d2:35 
setenv gatewayip 192.168.1.1 
setenv netmask 255.255.255.0 
setenv serverip 192.168.1.250 
saveenv
```

### ping命令

用于测试网络是否通了，跟windows下一个用法ping 192.168.1.250

### tftp命令

将目标主机下的文件下载到开发板的dram里，如：

tftp 80800000 zImage，即将主机tftp文件夹下的zImage镜像下载到开发板的dram的80800000开始的地址。

### EMMC和SD卡操作

**mmc info**：输出当前选中的mmc设备信息  
**mmc rescan**：扫描当前开发板所有的MMC设备  
**mmc list**：列表显示开发板上的所有的mmc设备
**mmc dev**：切换当前设备，如mmc dev 1，切换到mmc1（SD卡）  
**mmc part**：查看分区，有时候 SD卡或者 EMMC会有多个分区，可以使用命令“ **mmc part**”来查看其分区，比如查看 EMMC的分区情况，输入如下命令：  
mmc dev 1 //切换到 EMMC   
mmc part //查看 EMMC分区  
如果 EMMC里面烧写了 Linux系统的话， EMMC是有3个分区的，第 0个分区存放 uboot，第 1个分区存放 Linux镜像文件和设备树，第 2个分区存放根文件系统。但是一般我们能查看看到只有两个分区，那是因为第 0个分区没有格式化，所以识别不出来，实际上第 0个 分区是存在的。一个 新 的 SD卡 默认只有一个分区，那就是分区 0，所以前面讲解的 uboot烧写 到 SD卡 ，其实就是将 u-boot.bin烧写 到了 SD卡 的分区 0里面。  
如果要将 EMMC的分区 2设置为当前 MMC设置，可以使用如下命令：  
mmc dev 1 2  
**mmc write**：将数据写到mmc设备里边，比如我们要将uboot从ubuntu主机的ftp服务下载到开发板的sdram，再从sdram直接烧写到sd中相应位置，操作如下：  
tftp 80800000 u-boot.imx  
下载完成后，可以看出， u-boot.imx大小为 416768字节，416768/512=814，所以我们要向 SD卡中写入814个块，如果有小数的话就要加 1个块。使用命令“ mmc write”从 SD卡分区 0第 2个块 (扇
区 )开始烧写，一共烧写 814(0x32E)个块，命令如下：  
mmc dev 0 0  
mmc write 80800000 2 32E  
如果要更新 EMMC中的 uboot也是一样的。同理，如果要在 uboot中更新 EMMC对应的 uboot，可以使用如下所示命令  
mmc dev 1 0 //切换到 EMMC分区 0   
tftp 80800000 u-boot.imx //下载 u-boot.imx到 DRAM   
mmc write 80800000 2 32E //烧写 u-boot.imx到 EMMC中  
mmc partconf 1 1 0 0 //分区配置， EMMC需要这一步！  
**注意：千万不要写 SD卡或者 EMMC的前两个块 (扇区 )，里面保存着分区表。**

### fat格式文件系统操作命令

**fatls**：查询fat格式设备的目录和文件信息，比如我们要查询 EMMC分区 1的文件系统信息，命令如下：  
fatinfo mmc 1:1  
**fstype**：查询mmc设备某个分区的文件系统格式，如开发板EMMC核心板上的 EMMC默认有 3个分区，我们来查看一下这三个分区的文件系统格式，输入命令：
fstype mmc 1:0  
fstype mmc 1:1   
fstype mmc 1:2  
**fatload**：用于将制定的文件读取到dram中，我们将 EMMC分区 1中的 zImage文件读取到 DRAM中的0X80800000地址处，命令如下：  
fatload mmc 1:1 80800000 zImage  

### BOOT操作命令

boot的本质工作是引导 Linux，所以 uboot肯定有相关的 boot(引导 )命令来启动 Linux。常用的跟 boot有关的命令有： bootz、 bootm和 boot。

假如ubuntu上用于下载的 Linux镜像文件和设备树都准备好了，我们先学习如何通过网络启动 Linux，使用 tftp命令将 zImage下载到 DRAM的 0X80800000地址处，然后将设备树 imx6ull-alientek-emmc.dtb下载到 DRAM中的 0X83000000地址处，最后命令 bootz启动，命令如下：  
tftp 80800000 zImage  
tftp 83000000 imx6ull-alientek-emmc.dtb  
bootz 80800000 83000000  
这就是开发板通过uboot的tftp和bootz命令来从网络启动Linux系统。

**boot命令**  
boot命令也是用来启动 Linux系统的，只是 boot会读取环境变量 bootcmd来启动 Linux系统， bootcmd是一个很重要的环境变量！其名字分为“ boot”和 cmd”，也就是“引导”和“命令”，说明这个环境变量保存着引导命令，其实就是启动的命令集合，具体的引导命令内容是可以修改的。比如我们要想使用 tftp命令从网络启动 Linux那么就可以设置 bootcmd为“ tftp 80800000 zImage; tftp 83000000 imx6ull-alientek-emmc.dtb; bootz 80800000 - 83000000”，然后使用 saveenv将 bootcmd保存起来。然后直接输入 boot命令即可从网络启动 Linux系统，命令如下：
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-alientek-emmc.dtb; bootz 80800000 - 83000000'   
saveenv   
boot

**run命令**
run命令用于运行环境变量中定义的命令，比如可以通过“ run bootcmd”来运行 bootcmd中的启动命令，但是 run命令最大的作用在于运行我们自定义的环境变量。在后面调试 Linux系统的时候常常要在网络启动和 EMMC/NAND启动之间来回切换，而 bootcmd只能保存一种启动方式，如果要换另外一种启动方式的话就得重写 bootcmd，会很麻烦。这里我们就可以通过自定义环境变量来实现不同的启动方式，比如定义环境变量 mybootemmc表示从 emmc启动，定义 mybootnet表示从网络启动，定义 mybootnand表示从 NAND启动。如果要切换启动方式的话只需要运行“ run mybootxxx(xxx为 emmc、 net或 nand)”即可。
说干就干，创建环境变量 mybootemmc、 mybootnet和 mybootnand，命令如下  
setenv mybootemmc 'fatload mmc 1:1 80800000 zImage; fatload mmc 1:1 83000000 imx6ull-alientek-emmc.dtb;bootz 80800000 - 83000000'   
setenv mybootnand 'nand read 80800000 4000000 800000;nand read 83000000 6000000 100000;bootz 80800000 - 83000000'   
setenv mybootnet 'tftp 80800000 zImage;  tftp 83000000 imx6ull-alientek-emmc.dtb;  
bootz 80800000 - 83000000'   
saveenv 创建环境变量成功以后就可以使用 run命令来运行 mybootemmc、 mybootnet或 mybootnand来实现不同的启动：  
run mybootemmc 或  
run mytoobnand 或  
run mybootnet  


