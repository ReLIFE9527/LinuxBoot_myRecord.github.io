

## [U-ROOT](https://github.com/u-root/u-root)

U-Root是一种基于Go语言开发的initramfs,它可以实现一些基本的shell命令，与U-Root特有的boot命令，同时还支持直接将可执行文件编译进去，非常适合作为Linuxboot的initramfs.

1. [安装golang13以上版本](https://blog.csdn.net/Magic_Ninja/article/details/103213358)

2. 安装u-root

  ```
  go get -u github.com/u-root/u-root
  ```

使用说明可参考U-Root官方的README。

## [UTK](https://github.com/linuxboot/fiano)

在安装了golang的环境下：

```
go get -u github.com/linuxboot/fiano/cmds/utk
```

## QEMU

1. 先安装镜像源中的低版本：

```
sudo apt-get install qemu
```

2. 然后安装5.0版本以上的qemu-system-aarch64:

```
cd /disk/tools/src
git clone https://git.qemu.org/git/qemu.git
cd qemu
mkdir -p build-native
cd build-native
../configure --target-list=aarch64-softmmu --prefix=/usr/local
make install
```

3. 随后就能找到可执行文件`build-native/aarch64-softmmu/qemu-system-aarch64`，再将原来的`qemu-system-aarch64`替换掉即可

```
cp build-native/aarch64-softmmu/qemu-system-aarch64 /usr/bin/
cp build-native/aarch64-softmmu/qemu-system-aarch64 /usr/local/bin/
```



## aarch64-linux-gnu-

```
sudo apt-get install gcc-aarch64-linux-gnu
```

## EDK

1. 下载edk，编译BaseTools

```shell
mkdir /disk/tools/src
cd /disk/tools/src/
git clone https://github.com/tianocore/edk2
cd edk2
git submodule update --init
make -C BaseTools
```

​	Note: 如果`git submodule update --init`出现 `Could not resolve host:`错误，[可以修改`/etc/hosts`文件。](https://blog.csdn.net/weixin_41010198/article/details/89553879)

2. 设置环境变量

	Note: 如果想编译x86的UEFI，则无需创建`myexport.sh`，直接`source ./edksetup.sh`即可。

	2.1创建一个脚本`myexport.sh`，内容如下

```
export WORKSPACE=$PWD
export GCC5_AARCH64_PREFIX=aarch64-linux-gnu-
export PACKAGES_PATH=$PWD
export PYTHON_COMMAND=/usr/bin/python3
export EDK_TOOLS_PATH=$PWD/BaseTools
source ./edksetup.sh
```

​		2.2 执行脚本

```
chmod +x myexport.sh
source ./myexport.sh
```

3. 编译

```
For QEMU-ARM64:
build -p ArmVirtPkg/ArmVirtQemu.dsc -b DEBUG -a AARCH64 -t GCC5

For QEMU-x86_64:
build -p OvmfPkg/OvmfPkgX64.dsc -b DEBUG -a X64 -t GCC5
```



## GDB

For x86:

```
sudo apt-get install gdb
```

For arm64:

```
cd /disk/tools
wget https://ftp.gnu.org/gnu/gdb/gdb-9.2.tar.gz
cd gdb-9.2
mkdir build
cd build
../configure --target=aarch64-linux-gnu --program-prefix=aarch64-linux-gnu- --prefix=/usr/bin
make
make install
```

随后便可以在build/gdb中找到可执行文件:

```
./build/gdb/gdb --version

GNU gdb (GDB) 9.2
Copyright (C) 2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

## HEADS

1. 安装依赖

```
sudo apt install -y \
		build-essential \
		zlib1g-dev uuid-dev libdigest-sha-perl \
		libelf-dev \
		bc \
		bzip2 \
		bison \
		flex \
		git \
		gnupg \
		iasl \
		m4 \
		nasm \
		patch \
		python \
		wget \
		gnat \
		cpio \
		ccache \
		pkg-config \
		cmake \
		libusb-1.0-0-dev \
		pkg-config \
		texinfo \
```

2. 下载并编译HEADS

```
git clone https://github.com/osresearch/heads
cd heads
make BOARD=qemu-linuxboot
```
