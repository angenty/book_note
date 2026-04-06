# uboot编译烧录使用过程

## 编译烧录过程

首先在 Ubuntu 中安装 ncurses 库，否则编译会报错，安装命令如下：

```shell
sudo apt-get install libncurses5-dev
```

**编译命令**：

```shell
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- distclean
make ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- mx6ull_14x14_ddr512_emmc_defconfig
make V=1 ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j12
```

**烧写与启动:**

```c
chmod 777 imxdownload  //给予 imxdownload 可执行权限，一次即可
./imxdownload u-boot.bin /dev/sdd //烧写到 SD 卡
```

其中imxdownload为烧录软件，会为bin文件条件imx头部
==SD卡设别不一定是/dev/sdd，具体拔插设备查看，烧录速度为几百KB/s为正常烧录==

## uboot常用命令

### 信息查询命令

| 命令       | 说明                                                |
| -------- | ------------------------------------------------- |
| bdinfo   | 可以看到 DRAM 的起始地址和大小、启动参数保存起始地址、波特率、sp(堆栈指针)起始地址等信息 |
| printenv | 用于输出环境变量信息（这个常用）                                  |
| version  | 用于查看 uboot 的版本号                                   |

### 环境变量操作命令

环境变量的操作涉及到两个命令：setenv 和 saveenv。

| 命令      | 说明             |
| ------- | -------------- |
| setenv  | 用于设置或者修改环境变量的值 |
| saveenv | 用于保存修改后的环境变量   |

一般环境变量是存放在外部 flash 中的，uboot 启动的时候会将环境变量从 flash 读取到 DRAM 中。所以使用命令 setenv 修改的是 DRAM中的环境变量值，修改以后要使用 saveenv 命令将修改后的环境变量保存到 flash 中，否则的话uboot 下一次重启会继续使用以前的环境变量值。

| 命令                   | 说明     |
| -------------------- | ------ |
| setenv  \[变量\]  \[值] | 修改环境变量 |
| setenv  \[变量\]  \[值] | 新建环境变量 |
| setenv  \[变量\]       | 删除环境变量 |

使用完后都要在后面后一句saveenv，保存到flash里

**环境变量 bootargs**

bootargs 保存着 uboot 传递给 Linux 内核的参数，bootargs 环境变量是由 mmcargs 设置的

```shell
mmcargs=setenv bootargs console=${console},${baudrate} root=${mmcroot}

#其中 console=ttymxc0，baudrate=115200，mmcroot=/dev/mmcblk1p2 rootwait rw，因此将
#mmcargs 展开以后就是：
mmcargs=setenv bootargs console= ttymxc0, 115200 root= /dev/mmcblk1p2 rootwait rw
```

常用的参数有：

**console**

console 用来设置 linux 终端(或者叫控制台)，也就是通过什么设备来和 Linux 进行交互，是
串口还是 LCD 屏幕？ttymxc0 后面有个“,115200”，这是设置串口的波特率console=ttymxc0,115200 综合起来就是设置 ttymxc0（也就是串口 1）作为 Linux 的终端，并且串口波特率设置为 115200。

**root**

root 用来设置根文件系统的位置，root=/dev/mmcblk1p2 用于指明根文件系统存放在mmcblk1 设备的分区 2 中。EMMC 版本的核心板启动 linux 以后会在/dev/mmcblk0、/dev/mmcblk1、/dev/mmcblk0p1、/dev/mmcblk0p2、/dev/mmcblk1p1和/dev/mmcblk1p2 这样的文件，其中/dev/mmcblkx(x=0~n)表示 mmc 设而/dev/mmcblkxpy(x=0~n,y=1~n)表示 mmc 设备x 的分区 y。在 I.MX6U-ALPHA 开发板中/dev/mmcblk1 表示 EMMC，而/dev/mmcblk1p2 表示EMMC 的分区 2。

root 后面有“rootwait rw”，rootwait 表示等待 mmc 设备初始化完成以后再挂载，否则的话mmc 设备还没初始化完成就挂载根文件系统会出错的。rw 表示根文件系统是可以读写的，不加rw 的话可能无法在根文件系统中进行写操作，只能进行读操作。

**rootfstype**

此选项一般配置 root 一起使用，rootfstype 用于指定根文件系统类型，如果根文件系统为ext 格式的话此选项无所谓。如果根文件系统是 yaffs、jffs 或 ubifs 的话就需要设置此选项，指定根文件系统的类型。

### 内存操作命令

自己看文档，不常用

### 网络操作命令

| 命令   | 说明                                                              |
| ---- | --------------------------------------------------------------- |
| dhcp | dhcp 用于从路由器获取 IP 地址                                             |
| ping | 验证网络能否使用，是否可以和服务器(Ubuntu 主机)进行通信                                |
| nfs  | nfs(Network File System)网络文件系统，通过 nfs 可以在计算机之间通过网络来分享资源         |
| tftp | tftp 命令的作用和 nfs 命令一样，都是用于通过网络下载东西到 DRAM 中，只是 tftp 命令使用的 TFTP 协议 |

nfs命令格式：

```shell
nfs [loadAddress] [[hostIPaddr:]bootfilename]
#例程
nfs 80800000 192.168.1.253:/home/zuozhongkai/linux/nfs/zImage
```

tftp服务器的搭建详见正点原子文档30.4节

**tftp命令格式**：

```shell
tftpboot [loadAddress] [[hostIPaddr:]bootfilename]
#例程
tftp 80800000 zImage
```

**网络相关环境变量**

| 变量        | 说明                                       |
| --------- | ---------------------------------------- |
| ipaddr    | 开发板 ip 地址，可以不设置，使用 dhcp 命令来从路由器获取 IP 地址。 |
| ethaddr   | 开发板的 MAC 地址，一定要设置。                       |
| gatewayip | 网关地址。                                    |
| netmask   | 子网掩码。                                    |
| serverip  | 服务器 IP 地址，也就是 Ubuntu 主机 IP 地址，用于调试代码。    |

### BOOT 操作命令

要启动 Linux，需要先将 Linux 镜像文件拷贝到 DRAM 中，如果使用到设备树的话也需要
将设备树拷贝到 DRAM 中。

可以从 EMMC 或者 NAND 等存储设备中将 Linux 镜像和设备树文件拷贝到 DRAM，也可以通过 nfs 或者 tftp 将 Linux 镜像文件和设备树文件下载到 DRAM 中。

不管用那种方法，只要能将 Linux 镜像和设备树文件存到 DRAM 中就行，然后使用 bootz 命令
来启动，bootz 命令用于启动 zImage 镜像文件
#### bootz 命令

**命令格式**：

```shell
bootz [addr [initrd[:size]] [fdt]]
```
addr 是 Linux 镜像文件在 DRAM 中的位置，initrd 是 initrd 文件在DRAM 中的地址，如果不使用 initrd 的话使用‘-’代替即可，fdt 就是设备树文件在 DRAM 中的地址。
#### bootm 命令

详见文档

#### boot 命令

boot 命令也是用来启动 Linux 系统的，只是 boot 会读取环境变量 bootcmd 来启动 Linux 系统

**例子**：

```shell
setenv bootcmd 'tftp 80800000 zImage; tftp 83000000 imx6ull-14x14-emmc-7-1024x600-c.dtb;bootz 80800000 - 83000000'
saveenv
boot
```

### 其他命令

不常用，详见正点原子文档30.4节

### UBOOT阶段配置使用过程

### 配置网络

```shell
setenv ipaddr 192.168.1.50           # 开发板 ip 地址
setenv ethaddr b8:ae:1d:01:00:00     # 开发板的 MAC 地址，一定要设置
setenv gatewayip 192.168.1.1         # 网关地址
setenv netmask 255.255.255.0         # 子网掩码
setenv serverip 192.168.1.253        # 服务器 IP 地址，也就是 Ubuntu 主机 IP 地址
saveenv                              # 保存
```

使用ping命令ping通算成功

### 搭建tftp服务

安装 tftp-hpa 和 tftpd-hpa

```shell
sudo apt-get install tftp-hpa tftpd-hpa
sudo apt-get install xinetd
```

创建一个文件夹来存放文件

```shell
mkdir /home/zuozhongkai/linux/tftpboot
chmod 777 /home/zuozhongkai/linux/tftpboot
```

配置 tftp

后新建文件/etc/xinetd.d/tftp，如果没有/etc/xinetd.d 目录的话自行创建，然后在里面输入如下内容：
```shell
示例代码 30.4.4.1 /etc/xinetd.d/tftp 文件内容
server tftp
{
 socket_type = dgram
 protocol = udp
 wait = yes
 user = root
 server = /usr/sbin/in.tftpd
 server_args = -s /home/zuozhongkai/linux/tftpboot/
 disable = no
 per_source = 11
 cps = 100 2
 flags = IPv4
}
```

完成以后启动 tftp 服务

```shell
sudo service tftpd-hpa start
```

打开/etc/default/tftpd-hpa 文件，将其修改为如下所示内容：

```shell
# /etc/default/tftpd-hpa

TFTP_USERNAME="tftp"
TFTP_DIRECTORY="/home/zuozhongkai/linux/tftpboot"
TFTP_ADDRESS=":69"
TFTP_OPTIONS="-l -c -s"
```

最后输入如下命令， 重启 tftp 服务器：

```shell
sudo service tftpd-hpa restart
```

将 zImage 镜像文件拷贝到 tftpboot 文件夹中，并且给予 zImage 相应的权限。
在板子上使用tftp 80800000 zImage验证

若ping不通则查看正点原子文档：03【正点原子】I.MX6U网络环境TFTP&NFS搭建手册V1.3.2.pdf，只能uboot ping通外面，外面ping不到uboot

usb要连接主机，不会连到虚拟机，不然识别不到


