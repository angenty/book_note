# uboot移植

## 添加自己开发板默认配置文件

即添加自己的开发板deconfig文件，make配置阶段使用自己的默认配置文件

## 添加开发板对应的头文件

在目录include/configs下添加自己开发板对应的头文件，例如复制include/configs/mx6ullevk.h，并重命名为 mx6ull_alientek_emmc.h

```shell
cp include/configs/mx6ullevk.h include/configs/mx6ull_alientek_emmc.h
```

mx6ull_alientek_emmc.h 里面有很多宏定义，这些宏定义基本用于配置 uboot，也有一些I.MX6ULL 的配置项目。如果我们自己要想使能或者禁止 uboot 的某些功能，那就在mx6ull_alientek_emmc.h 里面做修改即可。

## 添加开发板对应的板级文件夹

uboot 中每个板子都有一个对应的文件夹来存放板级文件，NXP 的 I.MX 系列芯片的所有板级文件夹都存放在 board/freescale 目录下，在这个目录下有个名为 mx6ullevk 的文件夹，这个文件夹就是 NXP 官方 I.MX6ULL EVK 开发板的板级文件夹。复制 mx6ullevk，将其重命名为 mx6ull_alientek_emmc，命令如下：

```shell
cd board/freescale/
cp mx6ullevk/ -r mx6ull_alientek_emmc
```

进 入 mx6ull_alientek_emmc 目 录 中 ， 将 其 中 的 mx6ullevk.c 文 件 重 命 名 为
mx6ull_alientek_emmc.c，命令如下：

```shell
cd mx6ull_alientek_emmc
mv mx6ullevk.c mx6ull_alientek_emmc.c
```

**修改 mx6ull_alientek_emmc 目录下的 Makefile 文件**

obj-y := mx6ullevk.o改为obj-y := mx6ull_alientek_emmc.o，因为文件名变了

**修改 mx6ull_alientek_emmc 目录下的 imximage.cfg 文件**

将PLUGIN board/freescale/mx6ullevk/plugin.bin 0x00907000改为PLUGIN board/freescale/mx6ull_alientek_emmc /plugin.bin 0x00907000

**修改 mx6ull_alientek_emmc 目录下的 Kconfig 文件**

主要是修改名称

例子：

```shell
if TARGET_MX6ULL_ALIENTEK_EMMC
 config SYS_BOARD
   default "mx6ull_alientek_emmc"
 
 config SYS_VENDOR
   default "freescale"

 config SYS_SOC
   default "mx6"

 config SYS_CONFIG_NAME
   default "mx6ull_alientek_emmc"
endif
```

**修改 mx6ull_alientek_emmc 目录下的 MAINTAINERS 文件**

也是因为名称修改了，所以要修改

```shell
MX6ULL_ALIENTEK_EMMC BOARD
M: Peng Fan <peng.fan@nxp.com>
S: Maintained
F: board/freescale/mx6ull_alientek_emmc/
F: include/configs/mx6ull_alientek_emmc.h
```

## 修改 U-Boot 图形界面配置文件

修改文件arch/arm/cpu/armv7/mx6/Kconfig(如果用的 I.MX6UL 的话，应该修改 arch/arm/Kconfig 这个文件)

加入

```shell
config TARGET_MX6ULL_ALIENTEK_EMMC
	bool "Support mx6ull_alientek_emmc"
	select MX6ULL
	select DM
	select DM_THERMAL
```
在最后一行的 endif 的前一行添加如下内容：
```shell
source "board/freescale/mx6ull_alientek_emmc/Kconfig"
```

==有时候要根据PCB的连接和外设芯片的更换修改驱动代码，才能移植成功==