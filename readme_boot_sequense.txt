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
1 . arch/arm/boot/compressed/head.S
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


2 . Kernel startup entry point.

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

4 . start_kernel - init/main.c

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
