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

	- The GPU starts executing the first stage bootloader, which is
	stored in ROM on the SoC. The first stage bootloader reads
	the SD card, and loads the second stage bootloader (bootcode.bin)
	into the L2 cache, and runs it.

		FSBL is used to mount the FAT32 boot partition on the SD card so that
	the second stage bootloader can be accessed. FSBL is programmed into the
	SoC itself during manufacture of the RPi and cannot be reprogrammed by a use

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



