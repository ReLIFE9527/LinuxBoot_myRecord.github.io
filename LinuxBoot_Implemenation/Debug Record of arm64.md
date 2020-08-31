# Debug Record of Linuxboot of QEMU-ARM64 

## Phenomenon:

```
Succeed:  UEFI(DT) ------> Kernel+U-root (Image) ---kexec---> Ubuntu

Fail: UEFI(ACPI) ------> Kernel+U-root (Image) ---x---> Ubuntu

Succeed: UEFI(ACPI) --grub--> Ubuntu --kexec--> Kernel+U-root (Image) --kexec--> Ubuntu
```



## Get Ubuntu's vmlinuz and vmlinux ( On ARM machine):

1. Run Ubuntu Cloud Image on qemu-kvm on ARM Machine, for fast compiling.

2. [Get Ubuntu linux-source-5.4.0 and install it](https://www.jianshu.com/p/c7e7775b174c):

  ```Shell
  Get linux kernel source code:
  sudo apt-get install linux-source
  sudo apt-get install libncurses5-dev
  
  Install kernel:
  mkdir kernel
  cp /usr/src/linux-source-5.4.0.tar.bz2 ./kernel
  cd kernel
  tar xf linux-source-5.4.0.tar.bz2
  cd linux-source-5.4.0
  cp /boot/config-5.4.0-42-generic .config
  make -j $(npro)
  make modules
  make modules_install
  make install
  ```

3. Upload the vmlinux and vmlinuz file to personal server, then download the files to host machine(x86):

  ```shell
  On ARM Machine:
  scp vmlinux bruce@xxx.xxx.xx.xxx:/home/Ubuntu_Files
  cd /boot
  scp vmlinuz-5.4.44 initrd.img-5.4.44 bruce@xxx.xxx.xx.xxx:/home/Ubuntu_Files
  
  On x86 Machine:
  cd /disk/system/ARM
  mkdir Ubuntu_Files
  scp bruce@xxx.xxx.xx.xxx:/home/Ubuntu_Files/*  ./Ubuntu_Files/
  ```

  

4. Create a virtual disk to store the vmlinuz and initrd:

	```Shell
	cd /disk/system/ARM
	qemu-img create -f raw disk.img 1G
	mkdir disk_file
	
	sudo mkfs.vfat -F 32 disk.img
	sudo mount -o loop disk.img disk_file
	sudo cp Ubuntu_Files/* ../disk_file/
	sudo umount disk_file
	```
	


## Get GDB for debug:

1. Install latest GDB-AARCH64

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



## Debug  Ubuntu's vmlinux:

1. qemu support for gdb:

	```
	sudo qemu-system-aarch64 -s \							# it will set a 1234 port for gdb to connect
						  -machine virt -cpu cortex-a57 \
	                        -nographic \
	                        -m 6G -smp 7 \
	                        -bios QEMU_EFI.fd \				# UEFI with ACPI
	                        -kernel Image_5.4 \              # Kernel with U-root
	                        -device virtio-net-pci,netdev=net0,romfile="" \
	                        -netdev type=user,id=net0 \
	                        -drive file=disk.img,format=raw			# Disk image Stores vmlinuz and initrd
	```
	
2. GDB connect and set Breakpoint

	```
	cd /disk/system/kernel/linux-source-5.4.0
	/disk/tools/gdb-9.2/build/gdb/gdb ./vmlinux_Ubuntu
	
	gdb:
	target remote localhost:1234
	b start_kernel
	......
	```

## Bug  Point:

`Cannot find available physical address in memblock`

```
start_kernel
	setup_arch
		(arch/arm64/mm/mmu.c:)
		paging_init
			map_kernel 701
				map_kernel_segment 656
					__create_pgd_mapping 550
						alloc_init_pud 359
							pud_phys = pgtable_alloc(PUD_SHIFT); 297
                            	 early_pgtable_alloc 95                  	 																	memblock_phys_alloc
										memblock_phys_alloc_range
                                    			memblock_alloc_range_nid
                                					found = memblock_find_in_range_node(mm/memblock.c:1373)
                                					# cannot find available physical address in memblock
                                 panic("Failed to allocate page table page\n");
```

## Detect when the memblock is set:

It seems that `memblock.memory.total_size` is changed  after  `efi_init();`

Besides, if UEFI using DT to config kernel, memblock will be initialized in `setup_machine_fdt()`.

```
In setup_arch:

291		setup_machine_fdt(__fdt_pointer);
(gdb) p memblock
$2 = {	bottom_up = false, 
		current_limit = 18446744073709551615, 
		memory = {cnt = 1, 
    				max = 128, 
    				total_size = 0, 
    				regions = 0xffff800011dd89c8 <memblock_memory_init_regions>, 
    				name = 0xffff8000112d96f0 "memory"}, 
    	reserved = {cnt = 1, 
    				max = 385, 
   					total_size = 0, 
   					regions = 0xffff800011dd95d0 <memblock_reserved_init_regions>, 
   					name = 0xffff8000112017d0 "reserved"}}
    
314		efi_init();
(gdb) p memblock
$8 = {	bottom_up = false, 
		current_limit = 18446744073709551615, 
		memory = {cnt = 1, 
    				max = 128, 
    				total_size = 0, 
    				regions = 0xffff800011dd89c8 <memblock_memory_init_regions>, 
    				name = 0xffff8000112d96f0 "memory"}, 
    	reserved = {cnt = 1, 
    				max = 385, 
    				total_size = 513, 
    				regions = 0xffff800011dd95d0 <memblock_reserved_init_regions>, 
    				name = 0xffff8000112017d0 "reserved"}}
    
315		arm64_memblock_init();
(gdb) p memblock
$9 = {	bottom_up = false, 
		current_limit = 18446744073709551615,
         memory = {	cnt = 7, 
    			   	max = 128, 
    			   	total_size = 6442450944, 
    			   	regions = 0xffff800011dd89c8 <memblock_memory_init_regions>, 
    			   	name = 0xffff8000112d96f0 "memory"}, 
    	reserved = {cnt = 6, 
    				max = 385, 
    				total_size = 530033, 
    				regions = 0xffff800011dd95d0 <memblock_reserved_init_regions>, 
    				name = 0xffff8000112017d0 "reserved"}}

```

## Explore  `efi_init();`

1. ### Read the code:

```
setup_arch
	efi_init
		if (!efi_get_fdt_params(&params))	# set params from fdt
			return;						  # if cannot get fdt, return	
		data = 	params 					   # set data by params
		efi_memmap_init_early(&data)
			__efi_memmap_init(data, false)
				efi.memmap = map;			#set efi.memmap by data
		reserve_regions()
			for_each_efi_memory_desc (md) == for_each_efi_memory_desc_in_map(&efi.memmap, md)
				paddr = md->phys_addr;
				npages = md->num_pages;
				size = npages << PAGE_SHIFT;
				early_init_dt_add_memory_arch(paddr, size);
					memblock_add(base, size);				# set the memblock by efi.memmap
```

2. ### Find bug point:

	```
	if (!efi_get_fdt_params(&params))
		return;			# program returned here
	```

	Explore `efi_get_fdt_params`:

	​	info.miss = "Secure Boot Enabled", ret =0, so this func will return 0.

	```
	int __init efi_get_fdt_params(struct efi_fdt_params *params)
	{
		struct param_info info;
		int ret;
	
		pr_info("Getting EFI parameters from FDT:\n");
	
		info.found = 0;
		info.params = params;
	
		ret = of_scan_flat_dt(fdt_find_uefi_params, &info);
		
		/*
	     * (gdb) p info
	     * $5 = {found = 5, params = 0xffff800011b03ec8, 
	     * missing = 0xffff8000115801c8 <fdt_params+360> "Secure Boot Enabled"}
	     * 
	     * (gdb) p info
	     * $6 ret = 0
		*/
		
		if (!info.found)
			pr_info("UEFI not found.\n");
		else if (!ret)
			pr_err("Can't find '%s' in device tree!\n",
			       info.missing);
	
		return ret;
	}
	```

	Explore `fdt_find_uefi_params`:

	​	Get uname from fdt, then compare is with `dt_params[i].name` .

	​	When the uname cannot be found in `dt_params[i]`, `info->missing` will be set.

	```
	static int __init fdt_find_uefi_params(unsigned long node, const char *uname,
					       int depth, void *data)
	{
		struct param_info *info = data;
		int i;
	
		for (i = 0; i < ARRAY_SIZE(dt_params); i++) {
			const char *subnode = dt_params[i].subnode;
	
			if (depth != 1 || strcmp(uname, dt_params[i].uname) != 0) {
				info->missing = dt_params[i].params[0].name;
				continue;
			}
			/*
			 * when cannot find the name in dt_params[i],
			 * info->missing will be set.
			 */
			if (subnode) {
				int err = of_get_flat_dt_subnode_by_name(node, subnode);
	
				if (err < 0)
					return 0;
	
				node = err;
			}
	
			return __find_uefi_params(node, info, dt_params[i].params);
		}
	
		return 0;
	}
	```

	Check `dt_params`:

	```
	static __initdata struct params fdt_params[] = {
		UEFI_PARAM("System Table", "linux,uefi-system-table", system_table),
		UEFI_PARAM("MemMap Address", "linux,uefi-mmap-start", mmap),
		UEFI_PARAM("MemMap Size", "linux,uefi-mmap-size", mmap_size),
		UEFI_PARAM("MemMap Desc. Size", "linux,uefi-mmap-desc-size", desc_size),
		UEFI_PARAM("MemMap Desc. Version", "linux,uefi-mmap-desc-ver", desc_ver),
		UEFI_PARAM("Secure Boot Enabled", "linux,uefi-secure-boot", secure_boot)
	};
	
	static __initdata struct params xen_fdt_params[] = {
		UEFI_PARAM("System Table", "xen,uefi-system-table", system_table),
		UEFI_PARAM("MemMap Address", "xen,uefi-mmap-start", mmap),
		UEFI_PARAM("MemMap Size", "xen,uefi-mmap-size", mmap_size),
		UEFI_PARAM("MemMap Desc. Size", "xen,uefi-mmap-desc-size", desc_size),
		UEFI_PARAM("MemMap Desc. Version", "xen,uefi-mmap-desc-ver", desc_ver)
	};
	static __initdata struct {
		const char *uname;
		const char *subnode;
		struct params *params;
	} dt_params[] = {
		{ "hypervisor", "uefi", xen_fdt_params },
		{ "chosen", NULL, fdt_params },
	};
	```

	

## Conclusion:

​	When UEFI using ACPI to config kernel, the kernel will generate an empty DT which contains some EFI map messages,  commandline, physical address for linux etc. However, when Ubuntu is booted by grub, the fdt it generated contains a node called "uefi-secure-boot", which cannot be generated by the kernel built by my own. Thus, when my linux with u-root is booting Ubuntu by kexec, Ubuntu cannot find the  "uefi-secure-boot" node from fdt, and consequently goes wrong.

## Temporary solution:

Comment the line blow in kernel code  and Ubuntu will not try to find this node.

```
UEFI_PARAM("Secure Boot Enabled", "linux,uefi-secure-boot", secure_boot)
```

True solution should be:

   Modifying the source code of Linux kernel built on my own, to generate an additional "uefi-secure-boot" node.