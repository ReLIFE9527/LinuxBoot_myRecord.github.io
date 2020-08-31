# **What is LinuxBoot**

- LinuxBoot is a project that aims to replace specific firmware functionality with a Linux kernel and runtime. 
- The overarching goal is moving from obscure, complex firmware to simpler, open source firmware.
- The goal of LinuxBoot is to reduce the role of firmware to a small, fixed-function core whose only purpose is to get a flash-based Linux kernel started. This “bare essentials” firmware prepares the hardware and starts a Linux kernel and a userland environment will run on the machine. 

# About LinuxBoot

- LinuxBoot **started as *NERF* in January 2017 at Google**. The following are just a subset of the organizations and people who have become involved.

- **Organizations Involved:**

	- [Google](http://www.google.com/)

	-  [Facebook](http://www.facebook.com/)

	-  [Horizon Computing](http://www.horizon-computing.com/)

	-  [Two Sigma](http://www.twosigma.com/)

	-  [9elements Cyber Security](http://www.9elements.com/cyber-security)

- **Technical Steering Committee**

	-  Ron Minnich (Google)

	-  Jean-Marie Verdun (Horizon Computing)

	-  Philipp Deppenwiese (9elements Cyber Security)

- Currently supports **server mainboards**:

	 [Intel S2600WF](https://trmm.net/S2600wf)

	 [Dell R630](https://trmm.net/NERF)

	 Winterfell Open Compute node

	 Leopard Open Compute node

	 Tioga Pass Open Compute node

	 Monolake Open Compute node (not tested)

- Implementation on **switch**:

	- [Arista 7368X4 implemented by Facebook in 2019](https://engineering.fb.com/data-center-engineering/f16-minipack/)

		The Arista 7368X4 (x86 CPU) had a different microserver, but more important, Arista has historically used a custom coreboot implementation rather than an off-the-shelf UEFI BIOS, Facebook engaged with Arista and now have a solution based on **coreboot, u-root, and LinuxBoot** to handle the microserver imaging.

		![image-20200828105811361](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200828105811361.png)

# **Why LinuxBoot  is needed**

- LinuxBoot replaces **proprietary, closed-source, vendor-supplied firmware drivers** with **Linux drivers**.

- This enables engineers writing Linux drivers and engineers writing firmware drivers to **focus on one set of drivers**.
- Because the drivers are part of Linux, **standard industry coding infrastructure** can be used to improve them.
- The drivers will have **fewer bugs**.
- **Stop reinventing the wheel** by implementing drivers for firmware again and again.



# **Benefits of LinuxBoot over UEFI**

- Reliability
	- Improves boot reliability by replacing lightly-tested firmware drivers with hardened Linux drivers.

- Engineering Productivity

	- Write a driver once, not twice

		- Linux is **open, measurable, reproducible, updatable**

		- Linux already has drivers for almost everything

	- Kernel Engineers = Firmware Engineers

		- Many more Engineers know Linux than know UEFI

	- Testing and debugging

		- Easier to write tests using resources (like network) with Linux

		- Test with a kernel in QEMU

- Security

	- Pxeboot and diskboot do parsing and other logic in userspace.

	- Kernel security patches can apply to firmware

- Customization
	- Allows customization of the initrd runtime to support site-specific needs

# **What LinuxBoot does**

- LinuxBoot replaces many Driver Execution Environment (DXE) modules used by Unified Extensible Firmware Interface (UEFI) and other firmware, particularly the network stack and file system modules, with Linux applications.
- LinuxBoot brings up the Linux kernel as a DXE in flash ROM instead of the UEFI shell or BDSDxe. The Linux kernel, with a provided Go based userland, can then bring up the kernel that you want to run on the machine.

![image-20200816112047190](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816112047190.png)





# **LinuxBoot Implementation**

## PART 1 : Replace UEFI shell with Linux Kernel

- Step 1: Boot Linux via netboot / UEFI shell.
- Step 2: Replace Shell binary section with Linux kernel + u-root.
	- U-root is a Go based initramfs.
	- The Linux kernel + u-root is supposed to boot Runtime OS by kexec.

![image-20200816093337872](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816093337872.png)

![image-20200816093355845](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816093355845.png)

## PART 2: Remove BDS:

- Step 3: Remove as many DXEs as possible.
- Step 4: Replace BDS DXE with linuxboot.ffs.
	- The linuxboot.ffs is supposed to load and boot the first kernel at the end of DXE Core.

![image-20200816093408189](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816093408189.png)

![image-20200816093420384](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816093420384.png)



## PART 3: Final Goal

​	So far, the LinuxBoot Project is not able to remove DXE phase, but it is th final goal of this project.

![image-20200816093436886](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816093436886.png)

# **An example in [LinuxBoot Project](https://github.com/linuxboot/linuxboot)**

## Step 1: Extract UEFI rom

- Get a UEFI rom that can run on qemu-system-x86, name it qemu.rom.

- Extract ffs and fv files from qemu.rom.
	- Note: the following parts won't be modified:
		- 0x000000.fv NVRAM
		- 0.fv PEI
		- 0x7cc000.fv SEC

​	![image-20200816094011459](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816094011459.png)

## Step 2: Prepare other necessary parts

- Prepare necessary parts that will be merged:
	- $(EDK2_OUTPUT_DIR)/DxeCore.efi	:  a DxeCore built by EDK II.

	- boards/$(BOARD)/image-files.txt          :  a list of necessary DXEs in qemu.rom

		```
		Linux Project will filter out unnecesary DXEs according this list.
		```

	- $(MAKE) -C dxe linuxboot.ffs                :  make a linuxboot.ffs that can replace BDS Dxe

	  ```
	  linuxboot.c:
	  	efi_bds_arch_protocol.bds_main = efi_bds_main;		// register the BDS callback
	  # linuxboot.ffs can directly load and boot kernel at the end of DXE Core,
	  # due to it has registered the BDS callback while the orignal BDS DXE is excluded.
    ```
	
- bzImage                                                :  Linux kernel Image
	
	- initrd.cpio.xz                                          :  initramfs

## Step 3: Merge all of the parts

- Merge all of the parts:

	![image-20200816100141712](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200816100141712.png)

# **What does** **LinuxBoot** **need?**

## UEFI:

- Able to boot linux kernel in EFI stub mode.
- Large enough to contain a linux kernel with a initramfs.

## Linux Kernel:

- Able to be booted by EFI stub mode.
- Able to initialize necessary drivers, like SCSI, USB, Network, etc.
- Able to contain a initramfs.
- Support kexec.

## Initramfs:

- Able to load drivers and find boot options and then boot RuntimeOS by kexec. 



# **Implementation Details**

- Build Linuxboot  of QEMU-x86 -- HEADS -- USB boot.md

- Build Linuxboot  of QEMU-x86 -- U-ROOT -- localboot.md

- Build Linuxboot  of QEMU-arm64.md

	(TODO)



# **Problems met in Implementation**  on QEMU-ARM64

## Phenomenon:

- Succeed: UEFI(DT) ------> Kernel+U-root (Image) ------> Ubuntu
- Fail: UEFI(ACPI) ------> Kernel+U-root (Image) ---x---> Ubuntu
- Succeed: UEFI(ACPI) --grub--> Ubuntu ------> Kernel+U-root (Image) ------> Ubuntu

## Reason:

- When UEFI using ACPI to config kernel, kernel will generate an empty DT which contains some EFI mmap messages, commandline, physical address for linux etc. 

- However, when Ubuntu is booted by grub, the fdt it generated contains a node called "uefi-secure-boot", which cannot be generated by the kernel built by my own.

- Thus, when my linux kernel with u-root is booting Ubuntu by kexec, Ubuntu cannot find the "uefi-secure-boot" node from fdt, and consequently goes wrong.

	![image-20200828111959894](https://github.com/ReLIFE9527/LinuxBoot_myRecord.github.io/tree/master/pics/image-20200828111959894.png)

## Temporary solution:

- Comment the line blow in source code of Ubuntu kernel and Ubuntu will not try to find this node.

	​    `UEFI_PARAM("Secure Boot Enabled", "linux,uefi-secure-boot", secure_boot)`

- The true solution should be modifying the souce code of kernel built on my own, to generate a addtional "uefi-secure-boot" node.



# **Limitations of LinuxBoot**

- Currently only support to boot **Linux Runtime OS**.
- U-Root is **not multifunctional enough** to support a user interface for configuration, although it is still in development.
- There is **no specification** to clarify **how to pass parameters** like ACPI table between two kernels.
- **Incompatibility** between two kernels.

# **Requirement** **to make** **LinuxBoot** **fast**

- Target Runtime OS supports to be booted by kexec,
	- and of course to be compatible with the first kernel.
- First kernel is supposed to be small enough,
	- only including necessary drivers.
- Original UEFI has too many useless DXEs.
	- Otherwise the time for booting kernel will be longer than loading DXEs.
- The kernel vmlinuz and initrd of Target Runtime OS are supposed to be small enough.
	- Otherwise it will take longer time to kexec the Target Runtime OS.

# **Follow-up Work**

- **Remove BDS phase.**
- **Reduce the volume** of Linux kernel.
- Modify source code of Linux kernel or fdt file in U-Root for **generating the “uefi-secure-boot” node.**

