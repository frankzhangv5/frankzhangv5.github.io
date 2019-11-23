---
layout: post
title:  "QEMU模拟ARM Versatile Express Cortex A9四核处理器开发板"
date:   2019-11-23 14:31:18 +0800
categories: linux arm
---

环境配置：Ubuntu 19.04，Kernel 4.4.199
<br>
内容参考了：https://blog.csdn.net/nxcxl88/article/details/53244754


# 1. 下载kernel
从kernel下载：
wget https://www.kernel.org/pub/linux/kernel/v4.x/linux-4.4.199.tar.xz
<br>
使用代理：
https://mirror.tuna.tsinghua.edu.cn/kernel/v4.x/linux-4.4.199.tar.xz
>4.4.1版本源码编译会报错


# 2. 安装arm交叉编译链
arm交叉编译链有很多种，有的书上装了arm-linux-gnueabi 和 arm-linux-gnueabihf两种，安装方法如下：
```
sudo apt-get install gcc-arm-linux-gnueabi
```
会安装下面这些工具：
```
binutils-arm-linux-gnueabi cpp-8-arm-linux-gnueabi cpp-arm-linux-gnueabi
  gcc-8-arm-linux-gnueabi gcc-8-arm-linux-gnueabi-base gcc-8-cross-base
  gcc-9-cross-base gcc-arm-linux-gnueabi libasan5-armel-cross
  libatomic1-armel-cross libc6-armel-cross libc6-dev-armel-cross
  libgcc-8-dev-armel-cross libgcc1-armel-cross libgomp1-armel-cross
  libstdc++6-armel-cross libubsan1-armel-cross linux-libc-dev-armel-cross
```

# 3. 编译Linux内核
解压下载的kernel文件，进入解压后的文件夹，运行下面的指令编译。
```
export ARCH=arm
export CROSS_COMPILE=arm-linux-gnueabi-
make vexpress_defconfig
make zImage -j8
make modules -j8
make dtbs
```

随后就可以将生成的zImage和dtbs拷贝到单独的文件夹中，拷贝指令：
```
mkdir out
cp arch/arm/boot/zImage out/
cp arch/arm/boot/dts/*ca9.dtb out/
cp .config  out/
```

# 4. 安装qemu模拟器
```
sudo apt-get install qemu-system-arm
```

# 5. 制作根文件系统
根文件系统由基本的运行命令、库和字符设备构成。
基本的运行命令需要使用Busybox，这个库也需要进行交叉编译，因此可以运行下面的指令进行源代码下载。

wget http://www.busybox.net/downloads/busybox-1.25.1.tar.bz2

解压源代码，在源代码目录下运行：
```
make defconfig
make CROSS_COMPILE=arm-linux-gnueabi-
make install CROSS_COMPILE=arm-linux-gnueabi-
```
安装完成后，该目录下生成的_install，即是基本的运行指令。
然后就开始一步步建立根文件系统。

## 5.1 创建rootfs目录：
```
mkdir rootfs

mkdir rootfs/lib
```

## 5.2 拷贝busybox命令
```
cp -r _install/* rootfs/
```

## 5.3 从工具链中拷贝运行库到lib下
```
cp -P /usr/arm-linux-gnueabi/lib/* rootfs/lib/
```
>如果编译busybox是静态链接的话，就不需要这一步；busybox默认时动态链接。

## 5.4 创建根文件系统目录
```
cd rootfs
mkdir dev proc sys tmp root var mnt home etc
mkdir -p etc/init.d/
```

- 下载Embedded_Linux_Files
```
git clone https://github.com/mahadevvinay/Embedded_Linux_Files.git
```

- 拷贝自动挂载配置文件和init启动配置文件
```
cp ./Embedded_Linux_Files/fstab  ./rootfs/etc/
cp ./Embedded_Linux_Files/inittab  ./rootfs/etc/
cp ./Embedded_Linux_Files/rcS  ./rootfs/etc/init.d/
chmod 777 ./rootfs/etc/init.d/rcS
```


*注意：实际上还可以在这个镜像中插入其他的文件夹，可以参考[ARM Versatile Express Emulation On Qemu From Scratch: Part 1](https://learningfromyoublog.wordpress.com/2016/04/05/131/)，写的很详细。
## 5.5 生成镜像
```
dd if=/dev/zero of=a9rootfs.ext3 bs=1M count=32
```

## 5.6 格式化生成ext3文件系统
```
mkfs.ext3 a9rootfs.ext3
```

## 5.7 将文件拷贝到镜像中
```
mkdir tmpfs
sudo mount -t ext3 -o loop a9rootfs.ext3 tmpfs/
sudo cp -r rootfs/*  tmpfs/
sudo umount tmpfs
```

# 6. 启动qemu模拟系统
使用下面的指令启动，会另开一个窗口显示虚拟终端。
```
qemu-system-arm -M vexpress-a9 -m 512M -dtb out/vexpress-v2p-ca9.dtb -kernel out/zImage -append "root=/dev/mmcblk0 rw" -sd a9rootfs.ext3
```
使用下面的指令启动，会在当前窗口中显示qemu虚拟出的终端。
```
qemu-system-arm -M vexpress-a9 -m 512M -dtb out/vexpress-v2p-ca9.dtb -kernel out/zImage -nographic -append "root=/dev/mmcblk0 rw console=ttyAMA0" -sd a9rootfs.ext3
```
