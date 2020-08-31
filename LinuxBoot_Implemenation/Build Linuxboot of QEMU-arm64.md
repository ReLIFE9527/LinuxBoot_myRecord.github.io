

# Build Linuxboot of QEMU-arm64

## 创建一个工作文件夹：

```
mkdir /disk/system /disk/system/ARM		# /disk是我的虚拟机的硬盘挂载目录
```

## [U-ROOT](https://github.com/u-root/u-root):

U-Root是一种基于Go语言开发的initramfs,它可以实现一些基本的shell命令，与U-Root特有的boot命令，同时还支持直接将可执行文件编译进去，非常适合作为Linuxboot的initramfs.

1. [安装golang13以上版本](https://blog.csdn.net/Magic_Ninja/article/details/103213358)

2. 安装u-root

  ```
  go get -u github.com/u-root/u-root
  ```

3. 下载kexec-tools

```shell
cd $GOPATH/src/github.com/u-root
git clone https://github.com/linuxboot/bins
```

4. 修改u-root代码：

```go
/home/bruce/Projects/go/src/github.com/u-root/u-root/cmds/boot/localboot/main.go:
Line 60-71: # 增加一个函数，调用kexec命令去启动系统
func kexec_cmd_exec(cfg jsonboot.BootConfig) {
	var stdout, stderr bytes.Buffer
	log.Printf("kexec -f %s --initrd %s --append %s -d", cfg.Kernel, cfg.Initramfs, cfg.KernelArgs)
	kexec_load_cmd := exec.Command("kexec", "-f", cfg.Kernel, "--initrd", cfg.Initramfs, "--append", cfg.KernelArgs, "-d")
	kexec_load_cmd.Stdout = &stdout
	kexec_load_cmd.Stderr = &stderr
	if err := kexec_load_cmd.Run(); err != nil {
		log.Printf("Execute Command failed:" + err.Error())
	}
	outStr, errStr := string(stdout.Bytes()), string(stderr.Bytes())
	log.Printf("out:\n%s\nerr:\n%s\n", outStr, errStr)
}

func BootGrubMode(devices block.BlockDevices, baseMountpoint string, guid string, dryrun bool, configIdx int):
Line 138-163: # 注释原来用系统调用启动系统的代码，换成kexec_cmd_exec
if configIdx > -1 {
    for n, cfg := range bootconfigs {
        if configIdx == n {
            if dryrun {
                debug("Dry-run mode: will not boot the found configuration")
                debug("Boot configuration: %+v", cfg)
                return nil
            }

            /*
			* Add Code
			*/
            kexec_cmd_exec(cfg)

            /*
			* Original Code
			*/
            /*
			if err := cfg.Boot(); err != nil {
				log.Printf("Failed to boot kernel %s: %v", cfg.Kernel, err)
			}*/
        }
    }
    log.Printf("Invalid arg -config %d: there are only %d bootconfigs available\n", configIdx, len(bootconfigs))
    return nil
}

Line 171-186: # 注释原来用系统调用启动系统的代码，换成kexec_cmd_exec
// try to kexec into every boot config kernel until one succeeds
for _, cfg := range bootconfigs {
    debug("Trying boot configuration %+v", cfg)

    /*
	* Add Code
	*/
    kexec_cmd_exec(cfg)

    /*
	 * Original Code
	 */
    /*if err := cfg.Boot(); err != nil {
			log.Printf("Failed to boot kernel %s: %v", cfg.Kernel, err)
		}*/
}
```

5. 使用u-root编译initramfs并压缩打包成xz格式。

```shell
cd $GOPATH/github.com/u-root/u-root	  			   #根据GOPATH跟GOROOT生成的安装路径
vim my_generate.sh	
# 创建一个shell脚本，内容为：
##############################################################
GOARCH=arm64 GOOS=linux u-root -build=bb  \
        -files "../bins/kexec-tools/arm64/kexec:bbin/kexec" \                                       
        github.com/u-root/u-root/cmds/core/{init,ls,shutdown,cat,elvish,uname,cp,mount} \
        github.com/u-root/u-root/cmds/boot/localboot
xz --check=crc32 -9 --lzma2=dict=1MiB --stdout /tmp/initramfs.linux_arm64.cpio | dd conv=sync bs=512     of=/tmp/initramfs.linux_arm64.cpio.xz
cp /tmp/initramfs.linux_arm64.cpio.xz /disk/system/ARM/
###############################################################
此shell脚本能将可执行文件bins/kexec-tools/arm64/kexec编译到u-root的bbin/kexec中，从而让u-root可以使用该文件。

chmod +x my_generate.sh
./my_generate.sh
```



## **Kernel:**

下载最新的5.0以上longterm版本的内核：

```shell
mkdir /disk/system/kernel
cd /disk/system/kernel		
wget https://cdn.kernel.org/pub/linux/kernel/v5.x/linux-5.4.60.tar.xz
tar xvf linux-5.4.60.tar.xz
cd linux-5.4.60
```

修改Makefile，配置编译选项：

```makefile
ARCH            ?= arm64                                                                
CROSS_COMPILE   ?= aarch64-linux-gnu-
```

配置内核：

```
make defconfig
make menuconfig

#######添加的选项#########
1.EFI,EFI_STUB
	Firmware Drivers -->
		EFI (Extensible Firmware Interface) Support -->
			EFI Variable Support via sysfs
			EFI Bootloader Control	
2.virtio device support
	Device Drivers->
		SCSI device support ->
		SCSI low-level drivers ->
			<*>   virtio-scsi support		
3.KEXEC
	Kernel Features->
		kexec system call
		kexec file based system call

4.磁盘相关
	Device Drivers --> 
		SCSI device support --> 
			SCSI CDROM support (BLK_DEV_SR [=y]) --> 
			<*> SCSI generic support
		Serial ATA and Parallel ATA drivers --> 
			AHCI SATA support

	File systems -->
		CD-ROM/DVD Filesystems -->
			<*> ISO 9660 CDROM file system support
		DOS/FAT/NT Filesystems -->
			<*> MSDOS fs support
5.TPM
	Device drivers -->
		Character devices -->
			TPM Hardware Support` (enable the relevant drivers)
		
6. U-root运行环境
	General setup -->
		Configure standard kernel features (expert users) -->
			Multiple users, groups and capabilities support

	Firmware Drivers -->
		Google Firmware Drivers -->
			<*>   Coreboot Table Access
			<*>   Firmware Memory Console
			<*>   Vital Product Data
7. 包含Initrd
	General setup -->
		Initial RAM filesystem and RAM disk (initramfs/initrd) support -->
			(/disk/system/ARM/initramfs.linux_arm64.cpio.xz) Initramfs source file(s)
8. ACPI	相关
	Boot options
		Enable support for the ARM64 ACPI parking protocol

#######删除的选项#########
	Device Drivers -->
		FPGA Configuration Framework
		Sound card support
		Multimedia support
		Remote Controller support
		Graphics support -->
			VGA Arbitration
			Direct Rendering Manager
			NVIDIA Tegra host1x driver
		SOC (System On Chip) specific Drivers
			TI SOC drivers support 
		LED Support
		Platform support for Chrome hardware(transitional) 
		Platform support for Chrome hardware
		
	Platform selection -->
		除了ARMv8 software model (Versatile Express)都关掉

```

编译内核：

```shell
make -j $(nproc)
cp arch/arm64/boot/Image /disk/system/ARM/Image_5.4
```



## **UEFI**:

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

	2.1创建一个脚本`myexport.sh`，内容如下

```
export WORKSPACE=$PWD
export GCC5_AARCH64_PREFIX=aarch64-linux-gnu-
export PACKAGES_PATH=$PWD
export PYTHON_COMMAND=/usr/bin/python3
export EDK_TOOLS_PATH=$PWD/BaseTools
source ./edksetup.sh
```

​		2.2 设置环境变量

```
chmod +x myexport.sh
source ./myexport.sh
```

3. 修改代码

```
#修改代码，把生成的QEMU_EFI.fd的大小改为32M
/disk/tools/src/edk2/ArmVirtPkg/ArmVirt.dsc.inc: Line 14 and Line 28
!if $(TARGET) != NOOPT
#	DEFINE FD_SIZE_IN_MB    = 2
    DEFINE FD_SIZE_IN_MB    = 32
###################################注释默认的大小2M，改成32

# 添加对大小为32M时的赋值处理
!if $(FD_SIZE_IN_MB) == 32
    DEFINE FD_SIZE          = 0x2000000
    DEFINE FD_NUM_BLOCKS    = 0x2000
!endif
###################################添加对大小为32M时的赋值处理

/disk/tools/src/edk2/ArmVirtPkg/ArmVirtQemu.fdf: Line 29
# 添加对大小为32M时的赋值处理
!if $(FD_SIZE_IN_MB) == 32
DEFINE FVMAIN_COMPACT_SIZE  = 0x1fff000
!endif
###################################添加对大小为32M时的赋值处理
```

4. 编译

```
build -p ArmVirtPkg/ArmVirtQemu.dsc -b DEBUG -a AARCH64 -t GCC5
cp /disk/tools/src/edk2/Build/ArmVirtQemu-AARCH64/DEBUG_GCC5/FV/QEMU_EFI.fd /disk/system/ARM/QEMU_EFI_DEF.fd
```



## 用Linux Kernel替换掉UEFI Shell, 并移除不需要的DXEs

1. 下载安装`utk`

```
go get -u github.com/linuxboot/fiano/cmds/utk
```

2. 创建一个脚本来执行utk命令：

```shell
vim utk_modify.sh
chmod +x utk_modify.sh
./utk_modify.sh
```

3. `utk_modify.sh`的内容如下：

```shell
utk QEMU_EFI_DEF.fd \
	remove VirtioBlkDxe \
	remove VirtioNetDxe \
	remove VirtioScsiDxe \
	remove VirtioRngDxe \
	remove IScsiDxe \
	remove ScsiBus \
	remove ScsiDisk \
	remove UsbBusDxe \
	remove UsbKbDxe \
	remove UsbMassStorageDxe \
	remove Dhcp4Dxe \
	remove Ip4Dxe \
	remove Udp4Dxe \
	remove Mtftp4Dxe \
	remove TcpDxe \
	remove UefiPxeBcDxe \
	remove NvmExpressDxe \
	remove UhciDxe \
	remove EhciDxe \
	remove XhciDxe \
	remove tftpDynamicCommand \
	remove LinuxInitrdDynamicShellCommand \
	remove DisplayEngine \
	remove MnpDxe \
	remove ArpDxe \
	remove RamDiskDxe \
	remove ConPlatformDxe \
	remove ConSplitterDxe \
	remove GraphicsConsoleDxe \
	remove DiskIoDxe \
	remove PartitionDxe \
	remove Fat \
	remove UdfDxe \
	remove DpcDxe \
	remove VlanConfigDxe \
	remove BootGraphicsResourceTableDxe \
	remove PciHostBridgeDxe \
	remove PciBusDxe \
	remove VirtioPciDeviceDxe \
	remove ArmPciCpuIo2Dxe \
	replace_pe32 Shell Image_5.4 save QEMU_EFI.rom
```

至此，`QEMU_EFI.rom`制作完毕。

原`QEMU_EFI_DEF.fd`中剩下的DXE:

```
Remain:
	DxeCore
	PcdDxe
	VirtioFdtDxe
	FdtClientDxe
	HighMemDxe
	ArmCpuDxe
	RuntimeDxe
	SecurityStubDxe
	CapsuleRuntimeDxe
	FaultTolerantWriteDxe
	VariableRuntimeDxe
	MonotonicCounterRuntimeDxe
	ResetSystemRuntimeDxe
	RealTimeClock
	MetronomeDxe
	HiiDatabase
	TerminalDxe
	SerialDxe
	ArmGicDxe
	ArmTimerDxe
	ArmVeNorFlashDxe
	WatchdogTimer
	EnglishDxe
	ReportStatusCodeRouterRuntimeDxe
	DevicePathDxe
	SetupBrowser
	DriverHealthManagerDxe
	BdsDxe
	UiApp
	QemuKernelLoaderFsDxe
	SmbiosDxe
	SmbiosPlatformDxe
	PlatformHasAcpiDtDxe
	AcpiTableDxe
	QemuFwCfgAcpiPlatform
	EbcDxe
	Virtio10
	LogoDxe
```



## Ubuntu（需要在ARM主机上进行）：

1. 下载`focal-server-cloudimg-arm64.img`：

```shell
mkdir ~/bruce ~/bruce/ARM
cd ~/kvm-test/
cp flash0.img flash1.img ~/bruce/ARM		# 将kvm启动需要的文件都copy过去（这些文件由陈朱叠进行kvm测试时创建）
wget https://cloud-images.ubuntu.com/focal/current/focal-server-cloudimg-arm64.img
qemu-img resize focal-server-cloudimg-arm64.img +50G	#给磁盘扩容，增加50G容量

```

2. 创建登录cloud-img所需的额外的文件，该文件设置了cloud-img默认的登录账户ubuntu的密码为asdfqwer：

```shell
sudo apt-get install cloud-image-utils

cat >user-data <<EOF
#cloud-config
password: asdfqwer
chpasswd: { expire: False }
ssh_pwauth: True
EOF

cloud-localds user-data.img user-data
```

3. 创建启动脚本

```shell
vim start-qemu.sh
chmod +x start-qemu.sh

####创建一个start-qemu.sh启动脚本，内容如下####
#!/bin/sh
sudo qemu-system-aarch64  -enable-kvm -machine virt,gic-version=host -cpu host -m 4096 \
       -device virtio-blk-device,drive=drv0\
       -drive if=none,file=focal-server-cloudimg-arm64.img,id=drv0 \
       -pflash flash0.img \
       -pflash flash1.img \
       -drive file=user-data.img,format=raw \
       --nographic \
       -netdev type=user,id=net0 \
       -device virtio-net-pci,netdev=net0,romfile="" \
       -nic user,hostfwd=tcp:0.0.0.0:80-:80
###################################################
```

4. 使用kvm启动Ubuntu Server 20.04 Cloud Image并登录

```
./start-qemu.sh

ubuntu login: ubuntu
Password:asdfqwer
```

5. 在Ubuntu Server 20.04 Cloud Image中：
  1. 下载本系统的源代码

  	```shell
  	sudo apt-get install linux-source
  	sudo apt-get install libncurses5-dev
  	mkdir ~/bruce
  	tar -jxf /usr/src/linux-source-5.4.0/linux-source-5.4.0.tar.bz2 -C ~/bruce/
  	cd ~/bruce/linux-source-5.4.0/
  	```

  2. 修改该Ubuntu的内核源码

  	```c
  	linux-source-5.4.0/drivers/firmware/efi/efi.c: 693
  	/***********注释掉uefi-secure-boot结点，并删除上一行末尾的 ','**********/
  	UEFI_PARAM("MemMap Desc. Version", "linux,uefi-mmap-desc-ver", desc_ver)
  	//UEFI_PARAM("Secure Boot Enabled", "linux,uefi-secure-boot", secure_boot)
  	```

  3. 编译安装（耗时很久，大概三四个小时，所以才需要在ARM主机上使用kvm来编译）

  	```shell
  	cp /boot/config-5.4.0-42-generic .config		#将原来的内核配置文件拷贝过来
  	make -j $(nproc)
  	make modules
  	make modules_install
  	make install
  	```

  4. 将编译出来的vmlinux,vmlinuz,以及initrd文件都copy到个人服务器上

    （因为暂时没找到kvm与宿主机之间如何文件共享，只能通过网络再绕一下路）
    
    ```shell
    scp vmlinux bruce@xxx.xxx.xx.xxx:/home/Ubuntu_Files
    cd /boot
    scp vmlinuz-5.4.44 initrd.img-5.4.44 bruce@xxx.xxx.xx.xxx:/home/Ubuntu_Files
    ```

  5. 关闭kvm

  	```shell
  	sudo poweroff
  	```

至此，我们在`focal-server-cloudimg-arm64.img`中重新编译安装了一个5.4.44版本的内核，且得到了它的vmlinux与vmlinuz文件，可以在x86的宿主机上进行测试，还可以用gdb进行调试。



## **测试运行**

1. 将自己编译出来的Ubuntu的vmlinux,vmlinuz,以及initrd文件从个人服务器上下载过来

	```shell
	cd /disk/system/ARM
	mkdir Ubuntu_Files
	scp bruce@xxx.xxx.xx.xxx:/home/Ubuntu_Files/* Ubuntu_Files/
	```

2. 制作一个磁盘用来存储上述文件

	```shell
	#创建磁盘镜像以及文件夹
	qemu-img create -f raw disk.img 1G
	mkdir disk_file
	
	sudo mkfs.vfat -F 32 disk.img 		    # 格式化磁盘镜像
	sudo mount -o loop disk.img disk_file	# 磁盘挂载到disk_file
	sudo cp Ubuntu_Files/* ../disk_file/	# 把相关文件都拷贝到disk_file里面
	sudo umount disk_file				   # 解除挂载，镜像制作完毕
	```

3. 运行`QEMU_EFI.rom`

	```
	sudo qemu-system-aarch64 -machine virt -cpu cortex-a57 \
	                        -nographic \
	                        -m 6G -smp 7 \
	                        -s \
	                        -bios QEMU_EFI.rom \
	                        -device virtio-net-pci,netdev=net0,romfile="" \
	                        -netdev type=user,id=net0 \
	                        -drive file=disk.img,format=raw
	```

4. 尝试使用kexec命令启动vmlinuz

	```
	mount -t vfat /dev/vda /mnt/vda
	kexec -f /mnt/vda/vmlinuz-5.4.44 --append "console=ttyAMA0,115200" -d
	```

	若能正常加载并启动内核，且最后是因为未能挂载到文件系统而卡死，说明`QEMU_EFI.rom`可以实现挂载磁盘再用kexec命令跳转到大内核的功能，下一步就可以使用`QEMU_EFI.rom`在ARM主机上启动真正的Ubuntu了。

5. 将`QEMU_EFI.rom`拷贝到ARM主机上并运行

	```
	scp  QEMU_EFI.rom user0@30.224.165.134:/home/user0/bruce/ARM/
	sudo qemu-system-aarch64 -machine virt -cpu cortex-a57 \
	                        -nographic \
	                        -m 4096 -smp 4 \
	                        -s \
	                        -bios QEMU_EFI.rom \
	                        -device virtio-net-pci,netdev=net0,romfile="" \
	                        -netdev type=user,id=net0 \
	                        -device virtio-blk-device,drive=drv0 \
	                        -drive if=none,format=qcow2,file=focal-server-cloudimg-arm64.img,id=drv0 \
	                        -drive file=user-data.img,format=raw \
	```

6. 进入U-Root界面后，使用localboot命令跳转到Ubuntu

	```
	localboot -grub
	```

	等待五六分钟后就能看到Ubuntu的启动信息，说明`QEMU_EFI.rom`最终实现了LinuxBoot的功能。

## 后续工作

- 移除BDS阶段

	- 遇到的问题：

	1. 尝试使用LinuxBoot Project下面的体系（create-ffs与create-fv）对qemu-arm64.rom (在qemu-arm64上运行的UEFI rom) 进行拆开重组。
		- 由于该UEFI rom的PEI部分会被拆成许多ffs文件而没有一个fv文件，重新将这些ffs文件与DXE部分组合在一起时会发现每个ffs文件之间的pad空间都变大了，从而导致偏移地址改变无法正常工作。
		- 此外，由于该项目中最关键的linuxboot.ffs是按照x86体系架构开发编译的，就算可以正常拆分重组，对该部分也需要进行修改，使其编译出arm64能用的linuxboot.ffs.

	2. 尝试使用LinuxBoot项目下官方提供的utk工具（uefi tool kit）直接往qemu-arm64.rom里面插入linuxboot.ffs以及内核与initrd.

		- 在对qemu-arm64.rom实施之前，先对qemu.rom (在qemu-x86_64上运行的UEFI rom)尝试了该方法。实验后发现，对qemu.rom使用utk直接插入linuxboot.ffs会导致运行出错，而使用create-ffs与create-fv进行拆分重组则没有问题。

		- 这里怀疑官方对utk工具的开发还未完善，对于Driver类型的ffs文件的插入还存在着问题。

	3. 现在的实施方向
		- 将LinuxBoot Project中dxe目录下的相关文件复制到到edk工程里，进行合适修改后直接在edk工程里将linuxboot.c作为一部分编译进去。

- 删去不必要的驱动，减小Linux kernel的体积。

- 修改Linux Kernel的源码，或者给U-Root开发一个程序使其可以对fdt文件进行修改，从而添加一个*“uefi-secure-boot”*结点。