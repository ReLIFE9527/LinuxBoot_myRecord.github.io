# Build Linuxboot of QEMU-x86 -- U-ROOT localboot

# U-Root

​	U-Root是一种基于Go语言开发的initramfs,它可以实现一些基本的shell命令，与U-Root特有的boot命令，同时还支持直接将可执行文件编译进去，非常适合作为Linuxboot的initramfs.

1. [安装golang13以上版本](https://blog.csdn.net/Magic_Ninja/article/details/103213358)

2. 安装u-root

	go get -u github.com/u-root/u-root

3. 使用u-root编译initramfs并压缩打包成xz格式。

	```
	cd ~/Projects/go/src/github.com/u-root/u-root	   #根据GOPATH跟GOROOT生成的安装路径
	u-root -build=bb \				    			 #以busybox模式编译成binary文件
		   -uinitcmd=systemboot \					 #设置u-root的启动命令，也可不设置直接进入shell界面
		   core boot \				   				#包含cmds/core与cmds/boot下的所有命令
	xz --check=crc32 -9 --lzma2=dict=1MiB \
		--stdout /tmp/initramfs.linux_amd64.cpio \	   #u-root编译出来的文件默认放在/tmp下
		| dd conv=sync bs=512 \
		of=/tmp/initramfs.linux_amd64.cpio.xz
	```

# Kernel

Note: 以下步骤需要建立在HEADS工程已经编译出内核的基础上；

如果不想编译Heads,可以下载Heads后只取出它的内核配置文件，用它来编译自己准备的内核。

```
cd /disk
git clone https://github.com/osresearch/heads
cd /disk/system/kernel
wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.14.195.tar.xz
tar xvf linux-4.14.195.tar.xz
cd linux-4.14.195
cp /disk/heads/config/linux-linuxboot.config .config
```

1.从Heads的工程里把kernel文件夹以及相应的.config文件拷贝出来

```
cd /disk/system/kernel
cp -rf /disk/heads/build/linux-4.14.62 Heads-linux-4.14.62
cd Heads-linux-4.14.62
cp linux-linuxboot/.config .config
```

2.在Heads的内核配置文件的基础上再配置内核，加入需要的驱动，然后编译

```
make menuconfig
磁盘相关:
-> Device Drivers
	-> SCSI device support
		-> SCSI CDROM support (BLK_DEV_SR [=y])
	->Serial ATA and Parallel ATA drivers
		->AHCI SATA support

->Enable the block layer (BLOCK [=y])                                                  
    -> Partition Types                                                                    
		-> Advanced partition selection (PARTITION_ADVANCED [=y])                          
			-> PC BIOS (MSDOS partition tables) support (MSDOS_PARTITION [=y])
				[*] BSD disklabel (FreeBSD partition tables) support  
				[*] Minix subpartition support                        
				[*] Solaris (x86) partition table support             
				[*] Unixware slices support 
网络相关:
Device drivers ---> 
	Network device support ---> 
		Ethernet driver support ---> 
			Intel(R) PRO/1000 Gigabit Ethernet support

[*] Networking support  
		Networking options  ---> 
			[*] Packet socket                      		  //添加.配置CONFIG_PACKET
			[*]TCP/IP networking
				[*]IP: kernel level autoconfiguration
					[*] IP: DHCP support                  //添加
					[*] Network packet filtering--->      //添加，后面子选项可不选，配置CONFIG_NETFILTER
				[*]The IPv6 protocol
			[*] Network packet filtering framework (Netfilter)
make bzImage
```

# Linuxboot

1.安装Linuxboot

```
cd /disk/
git clone http://github.com/linuxboot/linuxboot
cd linuxboot
```

2.将所需的bzImage与initramfs拷贝过来并编译

```
mkdir components
cd components
cp /tmp/initramfs.linux_amd64.cpio.xz .
cp /disk/system/kernel/Heads-linux-4.14.62/arch/x86/boot/bzImage bzImage_4.14.62
cd ..
make \
    BOARD=qemu \
    KERNEL=./components/bzImage_4.14.62 \
    INITRD=./components/initramfs.linux_amd64.cpio.xz \
    config
make
```

4. 在QMEU上运行Linuxboot

  ```
  sudo qemu-system-x86_64 \
  		-machine q35,smm=on  \
  		-m 4G --enable-kvm \
  		-global ICH9-LPC.disable_s3=1 \
  		-global driver=cfi.pflash01,property=secure,value=on \
  		--serial stdio \
  		-drive if=pflash,format=raw,unit=0,file=/disk/linuxboot/build/qemu/linuxboot.rom \
  		-cdrom /disk/system/fs/fedora-coreos-32.20200601.3.0-live.x86_64.iso
  ```

启动到U-ROOT界面后使用`localboot -grub`命令即可启动FedoraCoreOS.

# 开启QEMU的网络功能

```
本机网卡名称: enp0s3				  #可将下面命令中的enp0s3都更换成自己的网卡名称

ifconfig enp0s3 down    		   # 首先关闭宿主机网卡接口
brctl addbr br0                     # 添加一座名为 br0 的网桥
brctl addif br0 enp0s3        	    # 在 br0 中添加一个接口
brctl stp br0 off                   # 如果只有一个网桥，则关闭生成树协议
brctl setfd br0 1                   # 设置 br0 的转发延迟
brctl sethello br0 1                # 设置 br0 的 hello 时间
ifconfig br0 0.0.0.0 promisc up     # 启用 br0 接口
ifconfig enp0s3 0.0.0.0 promisc up    # 启用网卡接口
dhclient br0                        # 从 dhcp 服务器获得 br0 的 IP 地址
brctl show br0                      # 查看虚拟网桥列表
brctl showstp br0                   # 查看 br0 的各接口信息

tunctl -t tap0 -u root              # 创建一个 tap0 接口，只允许 root 用户访问
brctl addif br0 tap0                # 在虚拟网桥中增加一个 tap0 接口
ifconfig tap0 0.0.0.0 promisc up    # 启用 tap0 接口
brctl showstp br0                   # 显示 br0 的各个接口

QEMU Command:
	-net nic -net tap,ifname=tap0,script=no,downscript=no	#完成上述更改后，运行QEMU时加入该命令即可开启网络功能
```

