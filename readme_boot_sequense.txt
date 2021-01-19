===================================================================
u-boot code flow (backward trace)
====================================================================
1 . main_loop calls 'autoboot_command' and control goes to
        'abortboot' where it waits for bootdelay
        and boots default command.
        common/autoboot.c

2 . u-boot commmand line
        main_loop => common/main.c

3 . main_loop is called from
        run_main_loop => common/board_r.c

4 . initcall_run_list  => common/board_r.c
        init_sequence_r =

5 . initcall_run_list is called by
        board_init_r => common/board_r.c

6 . board_init_f common/board_f.c
        initcall_run_list(init_sequence_f)

6 . _main calls board_init_r in arch/arm/lib/crt0.S

6 . _main calls board_init_f in arch/arm/lib/crt0.S

7 . board_init_r is called from
        _main => arch/arm/lib/crt0.S

8 . _main and lowlevel_init are called from
        arch/arm/cpu/(PROCESSOR_TYPE)/Start.S

9 . _start calls _main as defined in arch/arm/mach-bcm2835/u-boot.lds
        arch/arm/cpu/armv8/start.S


Board Initialisation Flow:
--------------------------

This is the intended start-up flow for boards. This should apply for both
SPL and U-Boot proper (i.e. they both follow the same rules).

Note: "SPL" stands for "Secondary Program Loader," which is explained in
more detail later in this file.

At present, SPL mostly uses a separate code path, but the function names
and roles of each function are the same. Some boards or architectures
may not conform to this.  At least most ARM boards which use
CONFIG_SPL_FRAMEWORK conform to this.

Execution typically starts with an architecture-specific (and possibly
CPU-specific) start.S file, such as:

        - arch/arm/cpu/armv7/start.S
        - arch/powerpc/cpu/mpc83xx/start.S
        - arch/mips/cpu/start.S

and so on. From there, three functions are called; the purpose and
limitations of each of these functions are described below.

lowlevel_init():
        - purpose: essential init to permit execution to reach board_init_f()
        - no global_data or BSS
        - there is no stack (ARMv7 may have one but it will soon be removed)
        - must not set up SDRAM or use console
        - must only do the bare minimum to allow execution to continue to
                board_init_f()
        - this is almost never needed
        - return normally from this function

board_init_f():
        - purpose: set up the machine ready for running board_init_r():
                i.e. SDRAM and serial UART
        - global_data is available
        - stack is in SRAM
        - BSS is not available, so you cannot use global/static variables,
                only stack variables and global_data

        Non-SPL-specific notes:
        - dram_init() is called to set up DRAM. If already done in SPL this
                can do nothing

        SPL-specific notes:
        - you can override the entire board_init_f() function with your own
                version as needed.
        - preloader_console_init() can be called here in extremis
        - should set up SDRAM, and anything needed to make the UART work
        - these is no need to clear BSS, it will be done by crt0.S
        - must return normally from this function (don't call board_init_r()
                directly)

Here the BSS is cleared. For SPL, if CONFIG_SPL_STACK_R is defined, then at
this point the stack and global_data are relocated to below
CONFIG_SPL_STACK_R_ADDR. For non-SPL, U-Boot is relocated to run at the top of
memory.

board_init_r():
        - purpose: main execution, common code
        - global_data is available
        - SDRAM is available
        - BSS is available, all static/global variables can be used
        - execution eventually continues to main_loop()

        Non-SPL-specific notes:
        - U-Boot is relocated to the top of memory and is now running from
                there.

        SPL-specific notes:
        - stack is optionally in SDRAM, if CONFIG_SPL_STACK_R is defined and
                CONFIG_SPL_STACK_R_ADDR points into SDRAM
        - preloader_console_init() can be called here - typically this is
                done by selecting CONFIG_SPL_BOARD_INIT and then supplying a
                spl_board_init() function containing this call
        - loads U-Boot or (in falcon mode) Linux

======================================================================
Linux boot sequense
======================================================================

1 . zImage decompression :
    arch/arm/boot/compressed/head.S
	- decompressed kernel start from either uImage or zImage
	- kernel execution address

	- and always enters in ARM

	if r1 = arm
	Always enter in ARM state for CPUs that support the ARM ISA. As
	of today (2014) that's exactly the members of the A and R classes.

	AR_CLASS(      .arm    )
		start:

	- fills resgisters
	/*
	 *   r0  = delta
	 *   r2  = BSS start
	 *   r3  = BSS end
	 *   r4  = final kernel address (possibly with LSB set)
	 *   r5  = appended dtb size (still unknown)
	 *   r6  = _edata
	 *   r7  = architecture ID
	 *   r8  = atags/device tree pointer
	 *   r9  = size of decompressed image
	 *   r10 = end of this image, including  bss/stack/malloc space if non XIP
	 *   r11 = GOT start
	 *   r12 = GOT end
	 *   sp  = stack pointer
	 *

	 --------------------------------------------------------
	arch/arm/boot/compressed/head.S: start
	* First code executed, jumped to by the bootloader, at label "start" (108)
    	* save contents of registers r1 and r2 in r7 and r8 to save off architecture ID and atags pointer passed in by bootloader (118)
    	* execute arch-specific code (inserted at 146)
        	- arch/arm/boot/compressed/head-xscale.S or other arch-specific code file
        	- added to build in arch/arm/boot/compressed/Makefile
        	- linked into head.S by linker section declaration:  .section “start”
        	- flush cache, turn off cache and MMU
    	* load registers with stored parameters (152)
        	- sp = stack pointer for decompression code (152)
	        - r4 = zreladdr = kernel entry point physical address
    	* check if running at link address, and fix up global offset table if not (196)
    	* zero decompression bss (205)
    	* call cache_on to turn on cache (218)
        	- defined at arch/arm/boot/compressed/head.S (320)
        	- call call_cache_fn to turn on cache as appropriate for processor variant
            		= defined at arch/arm/boot/compressed/head.S (505)
            		= walk through proc_types list (530) until find corresponding processor
            		= call cache-on function in list item corresponding to processor (511)
                		for ARMv5tej core, cache_on function is __armv4_mmu_cache_on (417)
                    		call setup_mmu to set up initial page tables since MMU must be on for cache to be on (419)
                    turn on cache and MMU (426)
    	* check to make sure won't overwrite image during decompression; assume not for this trace (232)
    	* call decompress_kernel to decompress kernel to RAM (277)
    	* branch to call_kernel (278)
        	- call cache_clean_flush to flush cache contents to RAM (484)
        	- call cache_off to turn cache off as expected by kernel initialization routines (485)
        	- jump to start of kernel in RAM (489)
            		= jump to address in r4 = zreladdr from previous load
                		zreladdr = ZRELADDR = zreladdr-y
                		zreladdr-y specified in arch/arm/mach-vx115/Makefile.boot

2 . ARM-specific kernel code :
    arch/arm/kernel/head.S: stext (72)

    	* call __lookup_processor_type (76)
        	- defined in arch/arm/kernel/head-common.S (146)
        	- search list of supported processor types __proc_info_begin (176)
			= kernel may be built to support more than one processor type
            		= list of proc_info_list structs
                		defined in arch/arm/mm/proc-arm926.S (467) and other corresponding proc-*.S files
                		linked into list by section declaration:  .section ".proc.info.init"
            		= return pointer to proc_info_list struct corresponding to processor if found, or loop in error if not
    	* call __lookup_machine_type (79)
		- defined in arch/arm/kernel/head-common.S (194)
		- search list of supported machines (boards)
			= kernel may be built to support more than one board
			= list of machine_desc structs
                		machine_desc struct for boards defined in board-specific file vx115_vep.c
	                	linked into list by section declaration that's part of MACHINE_DESC macro
        	- return pointer to machine_desc struct corresponding to machine (board)
    	* call __create_page_tables to set up initial MMU tables (82)
    	* set lr to __enable_mmu, r13 to address of __switch_data (91, 93)
        	- lr and r13 used for jumps after the following calls
	        - __switch_data defined in arch/arm/kernel/head-common.S (15)
	* call the __cpu_flush function pointer in the previously returned proc_info_list struct (94)
		- offset is #PROCINFO_INITFUNC into struct
        	- this function is __arm926_setup for the ARM 926EJ-S, defined in arch/arm/mm/proc-arm926.S (392)
            		= initialize caches, writebuffer
            		= jump to lr, previously set to address of __enable_mmu
	* __enable_mmu (147)
        	- set page table pointer (TTB) in MMU hardware so it knows where to start page-table walks (167)
        	- enable MMU so running with virtual addresses (185)
        	- jump to r13, previously set to address of __switch_data, whose first field is address of __mmap_switched
            		= __switch_data defined in arch/arm/kernel/head-common.S (15)


3 . Kernel startup entry point.
    arch/arm/kernel/head-common.S: __mmap_switched

 This is normally called from the decompressor code.  The requirements
 * are: MMU = off, D-cache = off, I-cache = dont care, r0 = 0,
 * r1 = machine nr, r2 = atags or dtb pointer.
 *
 * This code is mostly position independent, so if you link the kernel at
 * 0xc0008000, you call this at __pa(0xc0008000).
 *
 * See linux/arch/arm/tools/mach-types for the complete list of machine
 * numbers for r1.
 *
 * We're trying to keep crap to a minimum; DO NOT add any machine specific
 * crap here - that's what the boot loader (or in extreme, well justified
 * circumstances, zImage) is for.
 */
        .arm

	- create page table
	- enable mmu
	- __mmap_switched


3 . __mmap_switched
	start_kernel - arch/arm/kernel/head-common.S
	first c	 function

	- copy data segment to RAM (39)
	- zero BSS (45)
	- branch to start_kernel (55)

4 . Processor-independent kernel code
    start_kernel - init/main.c

	- it loads 'vmlinux' ... i.e uncompressed kernel in .S file
		OR

	- setup_arch - here it parses command line boot args and reads
	ATAG_CMDLINE. refer github repo u-boot-with-new-board
	'Read commnad line in kernel using Atags or DTB'.

	- parse_early_param, trap_init, IRQ_init,

	- rest_init
5 . rest_init - init/main.c
	kernel_thread(kernel_init) - create kernel thread

6 . kerel_init - kernel thread
	kernel_init_freeable
		do_basic_setup
			do_initcalls - init/main.c

	do_initcalls
		for all init calls it calls
		do_initcall_level
			do_initcall_level
				do_one_initcall(fn)

				=> it calls all inicalls as given below,
				which modules are included statically in
				kernel

				=> board file is registered at 4 level
				i.e arch_initcall

8 . kernel_thread(kthreadd)
	for each process, it creates kernel thread

9 . run_init_process - if tries to run init process from ramfs / file sys
	if not found, kernle panic else it boots accordingly.

===============================================================


#define pure_initcall(fn)               __define_initcall(fn, 0)

#define core_initcall(fn)               __define_initcall(fn, 1)
#define postcore_initcall(fn)           __define_initcall(fn, 2)
#define arch_initcall(fn)               __define_initcall(fn, 3)
#define subsys_initcall(fn)             __define_initcall(fn, 4)
#define fs_initcall(fn)                 __define_initcall(fn, 5)
#define rootfs_initcall(fn)             __define_initcall(fn, rootfs)
#define device_initcall(fn)             __define_initcall(fn, 6)
#define late_initcall(fn)               __define_initcall(fn, 7)
#define __initcall(fn) device_initcall(fn)

==============================================================================
Linux bootup sequnce
===============================================================================

ARM Linux Boot Sequence
The following traces the Linux boot sequence for ARM-based systems in the 2.6.18 kernel. It looks at just the earliest stages of the boot process, until the generic non-processor-specific start_kernel function is called. The line numbers of each statement are in parenthese at the end of the line; the kernel source itself can be conveniently browsed on the Linux Cross-Reference website.

zImage decompression
arch/arm/boot/compressed/head.S: start (108)
First code executed, jumped to by the bootloader, at label "start" (108)
save contents of registers r1 and r2 in r7 and r8 to save off architecture ID and atags pointer passed in by bootloader (118)
execute arch-specific code (inserted at 146)
arch/arm/boot/compressed/head-xscale.S or other arch-specific code file
added to build in arch/arm/boot/compressed/Makefile
linked into head.S by linker section declaration:  .section “start”
flush cache, turn off cache and MMU
load registers with stored parameters (152)
sp = stack pointer for decompression code (152)
r4 = zreladdr = kernel entry point physical address
check if running at link address, and fix up global offset table if not (196)
zero decompression bss (205)
call cache_on to turn on cache (218)
defined at arch/arm/boot/compressed/head.S (320)
call call_cache_fn to turn on cache as appropriate for processor variant
defined at arch/arm/boot/compressed/head.S (505)
walk through proc_types list (530) until find corresponding processor
call cache-on function in list item corresponding to processor (511)
for ARMv5tej core, cache_on function is __armv4_mmu_cache_on (417)
call setup_mmu to set up initial page tables since MMU must be on for cache to be on (419)
turn on cache and MMU (426)
check to make sure won't overwrite image during decompression; assume not for this trace (232)
call decompress_kernel to decompress kernel to RAM (277)
branch to call_kernel (278)
call cache_clean_flush to flush cache contents to RAM (484)
call cache_off to turn cache off as expected by kernel initialization routines (485)
jump to start of kernel in RAM (489)
jump to address in r4 = zreladdr from previous load
zreladdr = ZRELADDR = zreladdr-y
zreladdr-y specified in arch/arm/mach-vx115/Makefile.boot

ARM-specific kernel code
arch/arm/kernel/head.S: stext (72)
call __lookup_processor_type (76)
defined in arch/arm/kernel/head-common.S (146)
search list of supported processor types __proc_info_begin (176)
kernel may be built to support more than one processor type
list of proc_info_list structs
defined in arch/arm/mm/proc-arm926.S (467) and other corresponding proc-*.S files
linked into list by section declaration:  .section ".proc.info.init"
return pointer to proc_info_list struct corresponding to processor if found, or loop in error if not
call __lookup_machine_type (79)
defined in arch/arm/kernel/head-common.S (194)
search list of supported machines (boards)
kernel may be built to support more than one board
list of machine_desc structs
machine_desc struct for boards defined in board-specific file vx115_vep.c
linked into list by section declaration that's part of MACHINE_DESC macro
return pointer to machine_desc struct corresponding to machine (board)
call __create_page_tables to set up initial MMU tables (82)
set lr to __enable_mmu, r13 to address of __switch_data (91, 93)
lr and r13 used for jumps after the following calls
__switch_data defined in arch/arm/kernel/head-common.S (15)
call the __cpu_flush function pointer in the previously returned proc_info_list struct (94)
offset is #PROCINFO_INITFUNC into struct
this function is __arm926_setup for the ARM 926EJ-S, defined in arch/arm/mm/proc-arm926.S (392)
initialize caches, writebuffer
jump to lr, previously set to address of __enable_mmu
__enable_mmu (147)
set page table pointer (TTB) in MMU hardware so it knows where to start page-table walks (167)
enable MMU so running with virtual addresses (185)
jump to r13, previously set to address of __switch_data, whose first field is address of __mmap_switched
__switch_data defined in arch/arm/kernel/head-common.S (15)

arch/arm/kernel/head-common.S: __mmap_switched (35)
copy data segment to RAM (39)
zero BSS (45)
branch to start_kernel (55)

Processor-independent kernel code
init/main.c: start_kernel (456)


===============
Early booting on RPI
===============

1 . components :
---------------
	- GPU core
	- fsbl stores in ROM on SOC
	- second stage boot loader (bootcode.bin + loader.bin)
	- start.elf
	- config.txt, kernel.img
	- sysmd / init

2 . process :
------------

	- When the Raspberry Pi is first turned on, the ARM core is off,
	and a small RISC core on the GPU is responsible for booting the SoC,
	therefore most of the boot components are actually run on the GPU code, not the CPU.
	At this point the SDRAM is disabled.

		FSBL is used to mount the FAT32 boot partition on the SD card so that
	the second stage bootloader can be accessed. FSBL is programmed into the
	SoC itself during manufacture of the RPi and cannot be reprogrammed by a user

	- The GPU starts executing the first stage bootloader, which is
	stored in ROM on the SoC. The first stage bootloader reads
	the SD card, and loads the second stage bootloader (bootcode.bin)
	into the L2 cache, and runs it.


	- bootcode.bin enables SDRAM, and reads the third stage bootloader
	(loader.bin) from the SD card into RAM, and runs it.
		This is used to retrieve the GPU firmware from the SD card,
	program the firmware, then start the GPU.

	- loader.bin reads the GPU firmware (start.elf). loader.bin doesn't do much.
	It can handle elf files, and so is needed to load start.elf at the
	top of memory (ARM uses SDRAM from address zero).

	- start.elf Once is loaded, this allows the GPU to start up the CPU.
	An additional file, fixup.dat, is used to configure the SDRAM partition
	between the GPU and the CPU. At this point, the CPU is release from
	reset and execution is transferred over.

		start.elf reads config.txt, cmdline.txt and kernel.img and it loads kernel.img.
	It then also reads config.txt,cmdline.txt and bcm2835.dtb
	If the dtb file exists, it is loaded at 0×100 & kernel @ 0×8000
	If disable_commandline_tags is set it loads kernel @ 0×0 Otherwise
	it loads kernel @ 0×8000 and put ATAGS at 0×100

	- Everything is run on the GPU until kernel.img is loaded on the ARM.

	- start kernel running.

=================
Early bootup OMAP
=================
	There are several stages of bootloaders that perform different levels
	of initialization on an OMAP platform, in order to eventually load and
	run the filesystem. This figure shows the booting sequence of the ROM code,
	x-loader, u-boot, and kernel, with each stage performing enough
	configuration in order to load and execute the next.

(1) components :
----------------

	1 . when sys is first booted, CPU invokes reset vector to start the code
	at known location in ROM.

	=> ROM CODE :
		- performes minimal clocks, mem, peripheral configs
		- serches booting devices for valid booting image
		- load x-loader into sram (L1, L2, L3 cashes) and execute it.

	2 . X-loader :
		- setup pin configs
		- init clock and mem SDRAM
		- loads u-boot into SDRAM and execute it

	3 . u-boot :
		- performs some more platform inits .
		- setup boot arguments
		- passes controll to kernel

	4 . kernel :
		- decompressed kernel into SDram
		- setup peripheral like LCD, USB, SD, i2c, Spi,
		- mount linx fs that contains user libs and init / sysmd

(2)OMAP Boot Sequence :
---------------------
	1 . SYSBOOT Pins

		The internal ROM Code can attempt to boot from several different
peripheral and memory devices, including, but not limited to: serial (UART3),
SD Card, eMMC, NAND, and USB. The order in which these devices are searched
for a valid first-stage booting image (x-loader) is determine by a set of GPIO
configuration pins referred to as SYSBOOT. The TRM includes a table that shows
the booting device list that each combination of the SYSBOOT pins refers to.

	The SYSBOOT value can be read from physical address 0x480022f0,
	either using JTAG, or if you have linux running, use devmem2:

	=>
	# devmem2 0x480022f0 b
	/dev/mem opened.
	Memory mapped at address 0x40020000.
	Value at address 0x480022F0 (0x400202f0): 0x2F

	2 . First Stage Boot (x-loader) :
		The x-loader is a small first stage bootloader derived from
the u-boot base code. It is loaded into the internal static RAM by the OMAP
ROM code. Due to the small size of the internal static RAM, the x-loader is
stripped down to the essentials. The x-loader configures the pin muxing, clocks,
DDR, and serial console, so that it can access and load the second stage bootloader
(u-boot) into the DDR. This figure shows the code flow in the x-loader, beginning in start.S

		For example, the SYSBOOT pins could be set such that booting
device list consists of 1) serial (UART3), 2) SD card (MMC1), and 3) NAND flash.
In this case, the ROM code would first look for a valid x-loader over
the serial port, then in the SD card, then in the NAND flash. Whenever
it finds a valid x-loader, it proceeds with execution of that binary.

     PC			 _______________		 _______________
 __________		|		|		| NAND FLASH 	|
|   FSBL   |	Serial	|   Rom code 	|      NAND	|    FSBL	|
|  Serial  |----------->| Internal SRam	|-------------->|_______________|
|__________|	Loader	|_______________|	Loader	|_______________|
				|			|_______________|
				|			|_______________|
			    SD	| Loader		|_______________|
				|
			 _______v_______
			|    FAT32	|
			|   SD card	|
			|  FSBL (MLO)	|
			|_______________|

	A Serial Boot
		For serial boot, a simple ID is written out of the serial port.
	If the host responds correctly within a short window of time,
	the ROM will read from the serial port and transfer the data to the internal SRAM.
	Control is passed to the start of SDRAM if no errors are detected.
	UART3 is the only uart for which the ROM will attempt to load from.


	B SD Card Boot
		If MMC is included in the booting device list, the ROM looks for
	an SD Card on the first MMC controller. If a card is found, the ROM then
	looks for the first FAT32 partition within the partition table.
	Once the partition is found, the root directory is scanned for a special
	signed file called "MLO" (which is the x-loader binary with a header
	containing the memory location to load the file to and the size of the file).
	Assuming all is well with the file, it is transfered into the internal
	SRAM and control is passed to it. Both MMC1 and MMC2 can be used for booting.

	C NAND / eMMC Boot
		If NAND is included in the booting device list, the ROM attempts
	to load the first sector of NAND. If the sector is bad, corrupt, or blank,
	the ROM will try the next sector (up to 4) before exiting. Once a good
	sector is found, the ROM transfers the contents to SRAM and transfers
	control to it. (The same steps are performed for eMMC if eMMC is included
	in the booting device list instead of NAND.)

	3 . Second Stage Boot (u-boot) :

		The u-boot is a second stage bootloader that is loaded by the x-loader
into DDR. It comes from Das U-Boot. The u-boot can perform CPU dependent
and board dependent initialization and configuration not done in the x-loader.
The u-boot also includes fastboot functionality for partitioning and flashing the eMMC.
The u-boot runs on the Master CPU (CPU ID 0), which is responsible for the
initialization and booting; at the same time, the Slave CPU (CPU ID 1) is held
in the “wait for event” state. This figure shows the code flow in the u-boot, beginning in start.S.

		It is the job of the x-loader to transfer the 2nd stage loader
into main memory, which we call the u-boot. Typically both the x-loader and
u-boot come from the same storage medium. For example, typically if the x-loader
is transferred via USB, the u-boot will also be transferred via USB, and if
the x-loader is transferred via SD card, the u-boot will also be transferred
via SD card. However, this is not required. For example, you could flash the serial
x-loader into the NAND. The ROM code will load the x-loader from NAND and transfer
control to the x-loader, which will wait for the u-boot to be downloaded from the serial port.

 _______________________
|			|
|    Second stage bl	|					 _______
| Serial(u-boot) kermit	|			       		|	|
| SD FAT32 (u-boot.bin) |------------->	X-LOADER -------------->| SDRAM	|
| NAND  u-boot 		|					|_______|
|_______________________|

	A Serial Boot
		In the case of loading both the u-boot over the serial port,
	the x-loader waits for the host to initiate a kermit connection to
	reliably transfer large files into main memory. Once the file transfer
	is completed successfully, control is transfered.

	B SD Card Boot
		In the case of loading the u-boot via the SD card, the SD card
	x-loader looks for a FAT32 partition on the first MMC controller and
	scans the top level directory for a file named "u-boot.bin".
	It then transfers the file into main memory and transfers control to it.


	C NAND / eMMC Boot
		In the case of a u-boot stored in NAND, the x-loader expects
	the u-boot to be located at the 5th sector (offset 0x00800000).
	It transfers the image from NAND into main memory and transfers control
	to it. In the case of a u-boot stored in eMMC, the x-loader expects
	the u-boot to be located at the offset 0x200.

===============
L1 vs L2 Cache :
===============

	Cache memory is a special memory used by the CPU (Central Processing Unit)
of a computer for the purpose of decreasing the average time required to access memory.
Cache memory is a relatively smaller and also a faster memory, which stores most
frequently accessed data of the main memory. When there is request for a memory read,
cache memory is checked to see whether that data exists in cache memory.
If that data is in the cache memory, then there is no need to access the main memory
(which takes longer time to be accessed), therefore making the average memory access
time smaller. Typically, there are separate caches for data and instructions.
Data cache is typically set up in a hierarchy of cache levels (sometimes called
multilevel caches). L1 (Level 1) and L2 (Level 2) are the top most caches in this
hierarchy of caches. L1 is the closest cache to the main memory and is the cache
that is checked first. L2 cache is the next in line and is the second closest
to main memory. L1 and L2 vary in access speeds, location, size and cost.

	L1 Cache :
		L1 cache (also known as primary cache or Level 1 cache) is the
	top most cache in the hierarchy of cache levels of a CPU.
	It is the fastest cache in the hierarchy. It has a smaller size and a smaller
	delay (zero wait-state) because it is usually built in to the chip.
	SRAM (Static Random Access Memory) is used for the implementation of L1.

	L2 Cache :
		L2 cache (also known as secondary cache or Level 2 cache) is the
	cache that is next to L1 in the cache hierarchy. L2 is usually accessed only
	if the data looking for is not found in L1. L2 is usually used to bridge the
	gap between the performance of the processor and the memory.
	L2 is typically implemented using a DRAM (Dynamic Random Access Memory).
	Most times, L2 is soldered on to the motherboard very close to the chip
	(but not on the chip itself), but some processors like Pentium Pro
	deviated from this standard.


	What is the difference between L1 and L2 Cache?

		Although both L1 and L2 are cache memories they have their key
	differences. L1 and L2 are the first and second cache in the hierarchy
	of cache levels. L1 has a smaller memory capacity than L2. Also,
	L1 can be accessed faster than L2. L2 is accessed only if the requested
	data in not found in L1. L1 is usually in-built to the chip,
	while L2 is soldered on the motherboard very close to the chip.
	Therefore, L1 has a very little delay compared to L2. Because L1 is implemented
	using SRAM and L2 is implemented using DRAM, L1 does not need refreshing,
	while L2 needs to be refreshed. If the caches are strictly inclusive,
	all data in L1 can be found in L2 as well. However, if the caches are
	exclusive, same data will not be available in both L1 and L2.


reference :
rpi : https://raspberrypi.stackexchange.com/questions/10442/what-is-the-boot-sequence
      https://elinux.org/RPi_Software#Overview
omap : http://omappedia.org/wiki/Bootloader_Project
x86 : https://linoxide.com/booting/boot-process-of-linux-in-detail/


https://stackoverflow.com/questions/34805383/why-is-mlo-needed-in-boot-step
https://www.allinonescript.com/questions/22455153/what-is-the-need-of-second-stage-boot-loader-why-different-bootloaders-like-fi
(must) http://omappedia.org/wiki/Bootloader_Project
https://stackoverflow.com/questions/22455153/what-is-the-need-of-second-stage-boot-loader-why-different-bootloaders-like-fi
https://superuser.com/questions/515126/boot-loader-of-linux
https://superuser.com/questions/707050/primary-and-secondary-boot-loaders
https://linoxide.com/booting/boot-process-of-linux-in-detail/
https://stackoverflow.com/questions/15548004/why-do-we-need-a-bootloader-in-an-embedded-device


Omap
------------------------------------------
Let me explain it using OMAP platform as an example (just to provide some actual background rather than just theory or common knowledge). Take a look at some facts for starters:

On OMAP-based platforms the first program being run after power-on is ROM code (which is similar to BIOS on PC).
ROM code looks for bootloader (which must be a file named "MLO" and located on active first partition of MMC, which must be formatted as FAT12/16/32, -- but that's details)
ROM code copies content of that "MLO" file to static RAM (because regular RAM is not initialized yet). Next picture shows SRAM memory layout for OMAP4460 SoC:
SRAM memory layout on OMAP4460

SRAM memory is limited (due to physical reasons), so we only have 48 KiB for bootloader. Usually regular bootloader (e.g. U-Boot) binary is bigger than that. So we need to create some additional bootloader, which will initialize regular RAM and copy regular bootloader from MMC to RAM, and then will jump to execute that regular bootloader. This additional bootloader is usually referred as first-stage bootloader (in two-stage bootloader scenario).
So this first-stage bootloader is U-Boot SPL; and second-stage bootloader is regular U-Boot (or U-Boot proper). To be clear: SPL stands for Secondary Program Loader. Which means that ROM code is the first thing that loads (and executes) other program, and SPL is the second thing that loads (and executes) other program. So usually boot sequence is next: ROM code -> SPL -> u-boot -> kernel. And actually it's very similar to PC boot, which is: BIOS -> MBR -> GRUB -> kernel.

UPDATE

To make things absolutely clear, here is the table describing all stages of boot sequence (to clarify possible uncertainty in terminology used):

+--------+----------------+----------------+----------+
| Boot   | Terminology #1 | Terminology #2 | Actual   |
| stage  |                |                | program  |
| number |                |                | name     |
+--------+----------------+----------------+----------+
| 1      |  Primary       |  -             | ROM code |
|        |  Program       |                |          |
|        |  Loader        |                |          |
|        |                |                |          |
| 2      |  Secondary     |  1st stage     | u-boot   |
|        |  Program       |  bootloader    | SPL      |
|        |  Loader (SPL)  |                |          |
|        |                |                |          |
| 3      |  -             |  2nd stage     | u-boot   |
|        |                |  bootloader    |          |
|        |                |                |          |
| 4      |  -             |  -             | kernel   |
|        |                |                |          |
+--------+----------------+----------------+----------+
So I'm just using bootloader as synonym for U-Boot, and Program Loader as common term for any program that loads other program.

See also:

[1] SPL (at Wikipedia)

[2] TPL: SPL loading SPL - Denx

[3] Bootloader (at OSDev Wiki)

[4] Boot ROM vs Bootloader

