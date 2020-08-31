## 1.系统环境：Virtualbox6.1 & Ubuntu-16.04

### [下载配置Virtualbox与Ubuntu-16.04](https://zhuanlan.zhihu.com/p/35619204)

- 网络设置为NAT上网模式
- 安装增强功能
- [打开嵌套 VT-x/AMD-V 功能](https://www.jianshu.com/p/8031924995a4)（为了支持qemu-kvm）
- [创建共享文件夹](https://jingyan.baidu.com/article/656db918cca831e381249cce.html) （Virtualbox6.1还支持自动挂载）

![image-20200624161810733](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/blob/master/pics/image-20200624161810733.png)

## 2.在Ubuntu-16.04上安装所需环境：

	1.sudo apt-get install gcc-5,sudo apt install g++-5
	2.sudo apt install python
	3.sudo apt install make
	4.sudo apt install git
	5.sudo apt install uuid-dev && 	sudo apt install nasm && sudo apt install acpica-tools
	6.sudo apt update
	7.(for HEADS)
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
## 3.安装编译QEMU

```
sudo apt-get install qemu ovmf
```

安装完qemu可以输入qemu-然后按几次tab键查看是否安装成功

```
bruce@bruce-VirtualBox:~$ qemu-
qemu-aarch64              qemu-ppc                  qemu-system-mipsel
qemu-alpha                qemu-ppc64                qemu-system-moxie
qemu-arm                  qemu-ppc64abi32           qemu-system-nios2
qemu-armeb                qemu-ppc64le              qemu-system-or1k
qemu-cris                 qemu-s390x                qemu-system-ppc
qemu-hppa                 qemu-sh4                  qemu-system-ppc64
qemu-i386                 qemu-sh4eb                qemu-system-ppc64le
qemu-img                  qemu-sparc                qemu-system-ppcemb
qemu-io                   qemu-sparc32plus          qemu-system-s390x
qemu-m68k                 qemu-sparc64              qemu-system-sh4
qemu-make-debian-root     qemu-system-aarch64       qemu-system-sh4eb
qemu-microblaze           qemu-system-alpha         qemu-system-sparc
qemu-microblazeel         qemu-system-arm           qemu-system-sparc64
qemu-mips                 qemu-system-cris          qemu-system-tricore
qemu-mips64               qemu-system-i386          qemu-system-unicore32
qemu-mips64el             qemu-system-lm32          qemu-system-x86_64
```

然后可以查看是否支持kvm，若不支持，则需要开启Virtualbox的嵌套 VT-x/AMD-V 功能。

Note: Virtualbox6.0只支持AMD的CPU，Virtualbox6.1才支持AMD与Intel的CPU。

```
bruce@bruce-VirtualBox:~$ sudo kvm-ok
INFO: /dev/kvm exists
KVM acceleration can be used
```

## 4.安装编译HEADS与Linuxboot

​	HEADS是一个可以给各种主板提供开源固件的项目，它旨在给系统提供更好的安全性与数据保护，它支持的主板中就有qemu-linuxboot，可以直接编译出能在qemu-system-x86上运行的Linuxboot固件，以及其所需的kernel与initramfs。

```
git clone https://github.com/osresearch/heads
cd heads
make BOARD=qemu-linuxboot
```

Note: 

- 若出现一些安装包因为网络问题下载不下来的情况，可先找到那个安装包的所在文件夹，然后在Windows主机上将目标文件下载到共享文件夹，Ubuntu虚拟机上再将安装包copy到指定文件夹，然后重新make BOARD=qemu-linuxboot便可继续运行。

- 最后可能会在MAKE linuxboot处出现一些缺少某个头文件的错误，此时可以暂时不管那些错误，先去查找是否成功生成了bzImage与cpio文件：

```
bruce@bruce-VirtualBox:/disk/heads$ sudo find -name "bzImage"
./build/qemu-linuxboot/bzImage

bruce@bruce-VirtualBox:/disk/heads$ sudo find -name "*.cpio*"
./build/qemu-linuxboot/initrd.cpio.xz
```

- 若找到上述两个文件，就去./build/linuxboot-git目录下编译Linuxboot，最后能找到linuxboot.rom即为编译成功。

  Note: Linuxboot编译过程中会编译EDK II,所以需要主机上装有gcc-5；[可以下载安装后将系统的默认版本改为gcc-5](https://blog.csdn.net/qq_31175231/article/details/77774971),若后期需要更高版本的gcc来编译其他工程可以用相同方法设置回来。

```
cd build/linuxboot-git/
make \
	BOARD=qemu \
	KERNEL=../qemu-linuxboot/bzImage \
	INITRD=../qemu-linuxboot/initrd.cpio.xz \
	config
make
...............................................................................
bruce@bruce-VirtualBox:/disk/heads/build/linuxboot-git$ find -name "*.rom*"
./boards/qemu/qemu.rom
./build/qemu/linuxboot.rom
```

​		如果配置出错或者想重新配置，可以先执行下面命令再重复上面的步骤

```
make clean
rm .config
```

## 5.尝试运行

```
cd ../..
make BOARD=qemu-linuxboot run
```

​		make run实际执行的命令为：

```
qemu-system-x86_64 \
	-machine q35,smm=on  \
	-global ICH9-LPC.disable_s3=1 \
	-global driver=cfi.pflash01,property=secure,value=on \
	-redir tcp:5555::22 \
	--serial /dev/tty \
	-drive if=pflash,format=raw,unit=0,file=/disk/heads/build/qemu-linuxboot/linuxboot.rom
```

​		Note:若关闭QEMU后终端无法正常运行，可盲打`stty sane`使其恢复正常。

- 可以看到终端上打印的信息里Linuxboot与Heads都成功启动了，但是却无法进一步boot真正的kernel，而会卡在recovery shell；
  从打印信息中我们可以发现是因为没有/dev/sda，即磁盘分区，所以也就无法将其挂载到/boot而进行进一步的boot。

```
***** Normal boot: /bin/generic-init
y) Default boot
n) TOTP does not match
r) Recovery boot
u) USB boot
m) Boot menu
2020-06-24 06:51:43 NO TPM: Boot mode[    6.945821] e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
2020-06-24 06:51:48 NO TPM: Boot mode
mount: mounting /dev/sda1 on /boot failed: No such file or directory
!!!!! Unable to mount /boot
!!!!! Starting recovery shell
~ # ls dev/sda*
ls: dev/sda*: No such file or directory
```

## 5.让Linuxboot使用USB boot模式去挂载ISO文件

- 下载[Fedora CoreOS的ISO文件](https://getfedora.org/en/coreos/download?tab=metal_virtualized&stream=stable)

  Note:这里下载Fedora CoreOS而不是Ubuntu或者CentOS是因为Linuxboot在boot真正的内核之前需要校验一下ISO文件，而Heads编译出来的initramfs本地自带的校验key只有三种: `fedora.key   qubes-4.key  tails.key`, 可以在recovery shell里看到：

```
~ # ls etc/distro/
S.gpg-agent          S.gpg-agent.ssh      pubring.kbx
S.gpg-agent.browser  keys                 pubring.kbx~
S.gpg-agent.extra    private-keys-v1.d    trustdb.gpg
~ # ls etc/distro/keys/
fedora.key   qubes-4.key  tails.key
```

​		所以推断只有这三种OS可以通过校验（实际上Ubuntu的ISO就没有通过gpg的校验）。

- 根据`make run`实际执行的[QEMU命令](https://wiki.gentoo.org/wiki/QEMU/Options#Hard_drive)进行修改，指定Fedora CoreOS的ISO文件为USB启动盘，并在boot选择界面按u选择USB boot。

```
sudo qemu-system-x86_64 -machine q35,smm=on \
		-global ICH9-LPC.disable_s3=1 \
		-global driver=cfi.pflash01,property=secure,value=on \
		--serial /dev/tty \
		-drive if=pflash,format=raw,unit=0,file=/disk/heads/build/qemu-linuxboot/linuxboot.rom  \
		-usb -drive id=usbflash,unit=1,file=/disk/system/fs/fedora-coreos-32.20200601.3.0-live.x86_64.iso,if=none,boot=on,cache=writeback -device usb-storage,drive=usbflash \
		--enable-kvm \
		-m 4G   		#Provide 4G Size Memory to fedora-coreos for loading kernel and other components
```

- 可以观察到Linuxboot开始boot真正的内核，稍等一会就能在QEMU界面上观察到开机界面。


```
***** Normal boot: /bin/generic-init
y) Default boot
n) TOTP does not match
r) Recovery boot
u) USB boot
m) Boot menu
2020-06-24 07:06:47 NO TPM: Boot mode[    6.945026] e1000: eth0 NIC Link is Up 1000 Mbps Full Duplex, Flow Control: RX
2020-06-24 07:07:02 NO TPM: [   21.428586] random: fast init done
u2020-06-24 07:07:04 NO TPM: Boot mode
[   23.748246] ehci_hcd: USB 2.0 'Enhanced' Host Controller (EHCI) Driver
[   23.828274] ehci-pci: EHCI PCI platform driver
[   23.834876] ehci-pci 0000:00:1d.7: EHCI Host Controller
[   23.845103] ehci-pci 0000:00:1d.7: new USB bus registered, assigned bus number 1
[   23.852723] ehci-pci 0000:00:1d.7: irq 10, io mem 0x91021000
[   23.880359] ehci-pci 0000:00:1d.7: USB 2.0 started, EHCI 1.00
[   23.885787] hub 1-0:1.0: USB hub found
[   23.895258] hub 1-0:1.0: 6 ports detected
[   24.232703] usb 1-1: new high-speed USB device number 2 using ehci-pci
[   26.156231] usb-storage 1-1:1.0: USB Mass Storage device detected
[   26.163119] scsi host0: usb-storage 1-1:1.0
[   26.172318] usbcore: registered new interface driver usb-storage
[   27.212097] scsi 0:0:0:0: Direct-Access     QEMU     QEMU HARDDISK    2.5+ PQ: 0 ANSI: 5
[   27.216793] sd 0:0:0:0: Attached scsi generic sg0 type 0
[   27.232322] NOHZ: local_softirq_pending 202
[   27.250688] NOHZ: local_softirq_pending 202
[   27.250688] NOHZ: local_softirq_pending 202
[   27.250688] NOHZ: local_softirq_pending 202
[   27.270994] NOHZ: local_softirq_pending 202
[   27.270994] NOHZ: local_softirq_pending 202
[   27.270994] NOHZ: local_softirq_pending 202
[   27.289820] NOHZ: local_softirq_pending 202
[   27.299259] NOHZ: local_softirq_pending 202
[   27.308022] NOHZ: local_softirq_pending 202
[   27.342569] sd 0:0:0:0: [sda] 1421312 512-byte logical blocks: (728 MB/694 MiB)
[   27.370812] sd 0:0:0:0: [sda] Write Protect is off
[   27.385667] sd 0:0:0:0: [sda] Write cache: enabled, read cache: enabled, doesn't support DPO or FUA
[   27.441062]  sda: sda1
[   27.484142] sd 0:0:0:0: [sda] Attached SCSI disk
!!! Could not find any ISO, trying bootable USB
+++ Scanning for unsigned boot options
Loading the new kernel:
kexec -l /media/images/vmlinuz --initrd=/media/images/initramfs.img --append="mitigations=auto,nosmt systemd.unified_cgroup_hierarchy=0 coreos.liveiso=fedora-coreos-32.20200601.3.0 ignition.firstboot ignition.platform.id=metal  "
Starting the new kernel
[   47.952309] sd 0:0:0:0: [sda] Synchronizing SCSI cache
[   47.976278] kexec_core: Starting new kernel
```

![image-20200624151103960](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/blob/master/pics/image-20200624151103960.png)

![image-20200624151151017](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/blob/master/pics/image-20200624151151017.png)

至此，Linuxboot在QEMU上成功启动了Fedora CoreOS。