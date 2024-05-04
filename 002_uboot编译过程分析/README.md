# uboot

学习三星S3C2440芯片的uboot，从编译流程，`makefile`，源码分析到移植过程。

## 编译

uboot文件包括2部分：
1. `u-boot-1.1.6.tar.bz2` : 标准uboot文件
2. `boot-1.1.6_jz2440.patch` : 2440补丁包

编译流程如下：

1. 解压boot，得到u-boot-1.1.6目录

```sh
tar xvjf u-boot-1.1.6.tar.bz2 
```

2. 打补丁

进入uboot源码目录，执行patch命令来打补丁，选项`-p1`为删除一层路径。

```sh
cd u-boot-1.1.6
patch -p1 < ../u-boot-1.1.6_jz2440.patch
```

3. 编译

编译分为3个流程：

1. 清理工程保持干净 ： `make distclean`
2. 配置工程 ： `make 100ask24x0_config `
3. 启动编译 ： `make`或`make all`

## mkconfig配置过程

```mk
100ask24x0_config	:	unconfig
	@$(MKCONFIG) $(@:_config=) arm arm920t 100ask24x0 NULL s3c24x0
```

实际上等价于：
```mk
mkconfig 100ask24x0 arm arm920t 100ask24x0 NULL s3c24x0
   $0        $1      $2    $3       $4      $5     $6
```

在linux脚本中，$0表示命令，$1表示第1个参数，$6表示第6个参数。$#表述参数的个数，当前为6。

下面是mkconfig脚本分析：

1. 给mkconfig传递参数

```sh
#!/bin/sh -e

# Script to create header files and links to configure
# U-Boot for a specific board.
#
# Parameters:  Target  Architecture  CPU  Board [VENDOR] [SOC]
#
# (C) 2002-2006 DENX Software Engineering, Wolfgang Denk <wd@denx.de>
#

APPEND=no	# Default: Create new config file
BOARD_NAME=""	# Name to print in make output

mkconfig 100ask24x0 arm arm920t 100ask24x0 NULL s3c24x0
   $0        $1      $2    $3       $4      $5     $6

# 如果BOARD_NAME为空，则BOARD_NAME=$1=100ask24x0
[ "${BOARD_NAME}" ] || BOARD_NAME="$1"

[ $# -lt 4 ] && exit 1
[ $# -gt 6 ] && exit 1

# 打印配置
echo "Configuring for ${BOARD_NAME} board..."
```

2. 在include目录下创建符号链接，使asm指向asm-arm

```sh
#
# Create link to architecture specific headers
#
    cd ./include
	rm -f asm
	ln -s asm-$2 asm    # 建立链接文件asm，使得asm指向asm-arm
```

我们查看下include目录，发现确实有符号链接。我们为什么要创建符号链接？

```sh
s -l include/asm
lrwxrwxrwx 1 ding ding 7  3月 29 22:19 include/asm -> asm-arm
```

我们看一下include目录下的文件内容。可以看到，有各种不同架构的asm文件夹，如asm-arm，asm-i386。

```sh
ls -d asm*
asm-arm  asm-avr32  asm-blackfin  asm-i386  asm-m68k  asm-microblaze  asm-mips  asm-nios  asm-nios2  asm-ppc
```

如果我们在源码里写这么一行代码：

```c
#include <asm/types.h>
```

当我们给arm架构编译时，他实际上相当于

```c
#include <asm-arm/types.h>
```

3. 生成配置文件

之前进入了include目录，创建了符号链接。现在是在include目录下，继续创建config.mk

```sh
#
# Create include file for Make
#
echo "ARCH   = $2" >  config.mk	#新建文件并写入
echo "CPU    = $3" >> config.mk
echo "BOARD  = $4" >> config.mk

[ "$5" ] && [ "$5" != "NULL" ] && echo "VENDOR = $5" >> config.mk

[ "$6" ] && [ "$6" != "NULL" ] && echo "SOC    = $6" >> config.mk
```

执行完成后，config.mk的文件内容如下：

```sh
cat include/config.mk 
ARCH   = arm
CPU    = arm920t
BOARD  = 100ask24x0
SOC    = s3c24x0
```

接下来继续在include目录下，创建config.h文件。并向config.h文件中写入内容`#include <configs/100ask24x0.h>`

```sh
#
# Create board specific header file
#
	> config.h		# Create new config file

echo "/* Automatically generated - do not edit */" >>config.h
echo "#include <configs/$1.h>" >>config.h

exit 0
```

所以，我们最终的配置文件就是100ask24x0.h，放在include/configs/目录下，要在里面配置各种我们需要支持的功能。

## make编译过程

1. 包含配置过程生成的config.mk文件

```mk
# load ARCH, BOARD, and CPU configuration
include $(OBJTREE)/include/config.mk
export	ARCH CPU BOARD VENDOR SOC
```

2. 选择编译工具链

如果没有定义`CROSS_COMPILE`交叉编译工具链，ARCH为arm架构的话，编译工具链就是arm-linux-

```mk
ifndef CROSS_COMPILE
ifeq ($(HOSTARCH),ppc)
CROSS_COMPILE =
else
ifeq ($(ARCH),ppc)
CROSS_COMPILE = powerpc-linux-
endif
ifeq ($(ARCH),arm)
CROSS_COMPILE = arm-linux-
endif
```

3. OBJS目标文件

OBJS实际的值就是：cpu/arm920t/start.o，这就是启动文件啊

```mk
OBJS  = cpu/$(CPU)/start.o
```

3. LIBS库文件

可以看到，`net/libnet.a`就是把net目录下所有文件编译好了之后，打包成`libnet.a`这样一个库。

说白了就是，我们需要去定义的这些路径下把所有的文件，一起编译打包成静态库。

```mk
LIBS  = lib_generic/libgeneric.a
LIBS += board/lib100ask24x0.a
LIBS += cpu/arm920t/libarm920t.a
LIBS += cpu/arm920t/s3c24x0/libs3c24x0.a
LIBS += net/libnet.a
```

4. elf文件

```mk
$(obj)u-boot:		depend version $(SUBDIRS) $(OBJS) $(LIBS) $(LDSCRIPT)
		UNDEF_SYM=`$(OBJDUMP) -x $(LIBS) |sed  -n -e 's/.*\(__u_boot_cmd_.*\)/-u\1/p'|sort|uniq`;\
		cd $(LNDIR) && $(LD) $(LDFLAGS) $$UNDEF_SYM $(__OBJS) \
			--start-group $(__LIBS) --end-group $(PLATFORM_LIBS) \
			-Map u-boot.map -o u-boot
```

展开就是以下内容：

```mk
UNDEF_SYM=`arm-linux-objdump -x lib_generic/libgeneric.a board/100ask24x0/lib100ask24x0.a cpu/arm920t/libarm920t.a cpu/arm920t/s3c24x0/libs3c24x0.a lib_arm/libarm.a fs/cramfs/libcramfs.a fs/fat/libfat.a fs/fdos/libfdos.a fs/jffs2/libjffs2.a fs/reiserfs/libreiserfs.a fs/ext2/libext2fs.a net/libnet.a disk/libdisk.a rtc/librtc.a dtt/libdtt.a drivers/libdrivers.a drivers/nand/libnand.a drivers/nand_legacy/libnand_legacy.a drivers/usb/libusb.a drivers/sk98lin/libsk98lin.a common/libcommon.a |sed  -n -e 's/.*\(__u_boot_cmd_.*\)/-u\1/p'|sort|uniq`;\
	cd /home/ding/uboot/u-boot-1.1.6 && arm-linux-ld -Bstatic -T /home/ding/uboot/u-boot-1.1.6/board/100ask24x0/u-boot.lds -Ttext 0x33F80000  $UNDEF_SYM cpu/arm920t/start.o \
		--start-group lib_generic/libgeneric.a board/100ask24x0/lib100ask24x0.a cpu/arm920t/libarm920t.a cpu/arm920t/s3c24x0/libs3c24x0.a lib_arm/libarm.a fs/cramfs/libcramfs.a fs/fat/libfat.a fs/fdos/libfdos.a fs/jffs2/libjffs2.a fs/reiserfs/libreiserfs.a fs/ext2/libext2fs.a net/libnet.a disk/libdisk.a rtc/librtc.a dtt/libdtt.a drivers/libdrivers.a drivers/nand/libnand.a drivers/nand_legacy/libnand_legacy.a drivers/usb/libusb.a drivers/sk98lin/libsk98lin.a common/libcommon.a --end-group  \
		-Map u-boot.map -o u-boot
```

这些变量的值如下表：

+ $(OBJDUMP) : arm-linux-objdump
+ $(LIBS) : 
	```
	lib_generic/libgeneric.a
	board/100ask24x0/lib100ask24x0.a
	cpu/arm920t/libarm920t.a 
	cpu/arm920t/s3c24x0/libs3c24x0.a 
	lib_arm/libarm.a 
	fs/cramfs/libcramfs.a 
	fs/fat/libfat.a 
	fs/fdos/libfdos.a 
	fs/jffs2/libjffs2.a 
	fs/reiserfs/libreiserfs.a 
	fs/ext2/libext2fs.a 
	net/libnet.a 
	disk/libdisk.a 
	rtc/librtc.a 
	dtt/libdtt.a 
	drivers/libdrivers.a 
	drivers/nand/libnand.a 
	drivers/nand_legacy/libnand_legacy.a 
	drivers/usb/libusb.a 
	drivers/sk98lin/libsk98lin.a 
	common/libcommon.a
	``` 
+ $(LNDIR) : /home/ding/uboot/u-boot-1.1.6
+ L$(DFLAGS ) : -Bstatic -T /home/ding/uboot/u-boot-1.1.6/board/100ask24x0/u-boot.lds -Ttext 0x33F80000
	```
	LDFLAGS += -Bstatic -T $(LDSCRIPT) -Ttext $(TEXT_BASE) $(PLATFORM_LDFLAGS)
	```
+ $(OBJS) : 
	```
	cpu/arm920t/start.o
	```

链接脚本：board/100ask24x0/u-boot.lds

```ld
OUTPUT_FORMAT("elf32-littlearm", "elf32-littlearm", "elf32-littlearm")
/*OUTPUT_FORMAT("elf32-arm", "elf32-arm", "elf32-arm")*/
OUTPUT_ARCH(arm)
ENTRY(_start)
SECTIONS
{
	. = 0x00000000;

	. = ALIGN(4);
	.text      :
	{
	  cpu/arm920t/start.o	(.text)
          board/100ask24x0/boot_init.o (.text)
	  *(.text)
	}

	. = ALIGN(4);
	.rodata : { *(.rodata) }

	. = ALIGN(4);
	.data : { *(.data) }

	. = ALIGN(4);
	.got : { *(.got) }

	. = .;
	__u_boot_cmd_start = .;
	.u_boot_cmd : { *(.u_boot_cmd) }
	__u_boot_cmd_end = .;

	. = ALIGN(4);
	__bss_start = .;
	.bss : { *(.bss) }
	_end = .;
}
```

第一个运行的文件是start.o，第二个运行的文件是boot_init.o。
