.. _adsp debug:

Using JTAG to Debug Early Linux Booting
=======================================

Debugging early boot problems with Embedded Linux can be difficult, especially
when the system isn't able to boot far enough to provide console logging over
a serial or network connection. One way to attain more visibility for early
boot is to build the kernel with debug symbols, and use a JTAG debugger with
GDB to step through the boot process.

Background (Build System)
~~~~~~~~~~~~~~~~~~~~~~~~~

The output of a typical build system for the Linux kernel includes a standard
ELF (exectuable linkable format) file called "vmlinux" which contains the
kernel binary and optionally debug symbols. While this file is not directly
bootable, the Yocto build system strips debug symbols from this file to load
to the device, along with the device tree blob and initial ram file system. The
actual files which get loaded to the target for the ADSP Yocto releases are
``zImage`` (compressed kernel), ``*.dtb`` (device tree blob) and ``*.cpio.gz``
(compressed initramfs). There are also ``fitImage`` files included which
combine the kernel, device tree blob and initramfs into a single file.

To check whether a given "vmlinux" file contains debug symbols, you can use the
``file`` or ``readelf`` utilities. The following example shows the ``file`` output
for a modified 5.0.1 ADI release with a 6.12.0 Linux kernel, but the
instructions on this page will work for any release.

.. code-block:: bash

   file -b tmp/work/adsp_sc589_mini-adi_glibc-linux-gnueabi/linux-adi/6.12/linux-adsp_sc589_mini-standard-build/vmlinux
   ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, BuildID[sha1]=d36ff404bbf4e711b3e6476f987c01747f7960fb, with debug_info, not stripped

The **"with debug info, not stripped"** indicates this file contains debug symbols.

To build a kernel with debug symbols and a debuggable configuration, some extra
debug symbols need to be set to "y" via Kconfig. Yocto provides many ways of
adding custom configuration which include, among others:

1. Altering sources directly, cleaning the shared state cache and rebuilding
2. Providing a custom meta-layer for experimental or patches
3. Modifying a source via devtool (either temporarily or committing to a patch)

To demonstrate the simplest example, we will alter the sources directly.

Build with a Debug Configuration
********************************

To start, follow one of the "Getting Started" guides to complete a kernel build
and populate an SDK. This will ensure that a kernel rebuild is expedited and
that an SDK is available to use OpenOCD and GDB for debugging with.

Once a kernel build is in place, the following Kconfig symbols should be
enabled:

- **CONFIG_DEBUG_KERNEL=y** to enable kernel debug info options
- **CONFIG_DEBUG_INFO_DWARF4=y** to set debug info format to DWARF4
- **CONFIG_FRAME_POINTER=y** to keep frame pointers for backtraces

Add the following file in the Yocto sources:

.. code-block:: kconfig
   :caption: sources/meta-adi/meta-adi-adsp-sc5xx/recipes-kernel/linux/linux-adi/feature/cfg/debug-symbols.cfg

   # Enable kernel debug symbols for JTAG/GDB debugging
   # Use DWARF4 format and keeps frame pointers for stack backtraces
   CONFIG_DEBUG_KERNEL=y
   CONFIG_DEBUG_INFO_DWARF4=y
   CONFIG_FRAME_POINTER=y

Then modify the linux-adi_%.bb file to include the new configuration:

.. code-block:: text
   :emphasize-lines: 8,13
   :caption: sources/meta-adi/meta-adi-adsp-sc5xx/recipes-kernel/linux/linux-adi_%.bb

   # ...
   LICENSE="GPL-2.0-only"
   LIC_FILES_CHKSUM="file://COPYING;md5=6bc538ed5bd9a7fc9398086aedcd7e46"

   DEPENDS += "u-boot-mkimage-native dtc-native"

   # Include kernel configuration fragments
   # NOTE: Added debug-symbols.cfg
   SRC_URI:append="\
         file://feature/cfg/nfs.cfg \
         file://feature/cfg/crypto.cfg \
         file://feature/cfg/tracepoints.cfg \
         file://feature/cfg/debug-symbols.cfg
   "
   # ...

Next, rebuild the kernel with the following steps:

.. code-block:: bash

   # Clean Yocto's shared state cache for the linux-adi recipe,
   bitbake -c cleansstate linux-adi

   # Rebuild linux-adi
   bitbake linux-adi

   # Copy vmlinux file to known location
   cp tmp/work/adsp_sc589_mini-adi_glibc-linux-gnueabi/linux-adi/6.12/linux-adsp_sc589_mini-standard-build/vmlinux \
      $HOME/workspace/sc589-mini-5.0.1/vmlinux-dbg

This will store a kernel image with debug symbols to a known location.
This file will most likely be well over 100 MiB, compared to the stripped
kernel loaded to the target which will likely be 10-15 MiB depending on the
enabled features.

Finally, rebuild the complete Linux image including the corresponding stripped
zImage file:

.. code-block:: bash

   # Clean sstate cache for adsp-sc5xx-minimal, then rebuild
   bitbake -c cleansstate adsp-sc5xx-minimal
   bitbake adsp-sc5xx-minimal

   # Copy stripped loadable image to target boot media, e.g. tftp server
   cp tmp/deploy/images/adsp-sc589-mini/fitImage /tftpboot/

Load the Image
**************

Load the debug image to the target using your chosen boot media
(SD, eMMC, TFTP, etc.). Please consult the appropriate Getting Started Guide
for your target hardware for instructions on doing this.

When booting the target, stop the U-Boot bootloader by pressing "Enter" before
booting the kernel.

Connect OpenOCD
***************

In one terminal, connect OpenOCD with the following commands (can be
placed into a script for ease of use):

.. code-block:: bash
   :caption: openocd-kernel-dbg.sh

   #!/bin/bash
   export sdk_usr=/opt/adi-distro-glibc/5.0.1/sysroots/x86_64-adi_glibc_sdk-linux/usr/

   # Use skip_reset command to avoid resetting the target
   $sdk_usr/bin/openocd \
      -f $sdk_usr/share/openocd/scripts/interface/ice2000.cfg \
      -f $sdk_usr/share/openocd/scripts/target/adspsc58x.cfg \
      -c "skip_reset" \
      -c "init"

Connect GDB
***********

In a separate terminal, connect GDB with the following command (can be
placed into a script for ease of use):

.. code-block:: bash

   GDB="/opt/adi-distro-glibc/5.0.1/sysroots/x86_64-adi_glibc_sdk-linux/usr/bin/arm-adi_glibc-linux-gnueabi/arm-adi_glibc-linux-gnueabi-gdb"
   VMLINUX="$HOME/workspace/sc589-mini-5.0.1/vmlinux-dbg"
   LOAD_ADDR="0xC3000000"

   "$GDB" "$VMLINUX" \
     -ex "target extended-remote localhost:3333" \
     -ex "set print pretty on" \
     -ex "set pagination off" \
     -ex "add-symbol-file $VMLINUX $LOAD_ADDR"

A script can be used to simplify this step.

.. code-block:: bash
   :caption: gdb-kernel-dbg.sh

   #!/bin/bash
   # GDB helper script for SC589-MINI kernel debugging

   GDB="/opt/adi-distro-glibc/5.0.1/sysroots/x86_64-adi_glibc_sdk-linux/usr/bin/arm-adi_glibc-linux-gnueabi/arm-adi_glibc-linux-gnueabi-gdb"
   VMLINUX="$HOME/workspace/sc589-mini-5.0.1/vmlinux-dbg"
   LOAD_ADDR="0xC3000000"

   if [ ! -f "$VMLINUX" ]; then
      echo "Error: vmlinux not found at $VMLINUX"
      echo "Please build the kernel first with: bitbake linux-adi"
      exit 1
   fi

   if [ ! -f "$GDB" ]; then
      echo "Error: GDB not found at $GDB"
      exit 1
   fi

   echo "Starting GDB for SC589-MINI kernel debugging..."
   echo "GDB: $GDB"
   echo "vmlinux: $VMLINUX"
   echo "Load address: $LOAD_ADDR"
   echo ""

   "$GDB" "$VMLINUX" \
      -ex "target extended-remote localhost:3333" \
      -ex "set print pretty on" \
      -ex "add-symbol-file $VMLINUX $LOAD_ADDR"

Notes on Using GDB to Debug Early Boot
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that a GDB connection has been established, please be aware that there are
a couple potential obstacles when debugging early boot:

- The kernel boots using physical addresses until the MMU is enabled
- The vmlinux file contains virtual addresses, which may not easily translate

To mitigate these issues, use hardware breakpoints (``hbreak`` in gdb) to
capture early execution e.g. ``hbreak start_kernel`` to break on the start_kernel
function. Single stepping across MMU enabling code should also be avoided.

Useful GDB Commands
~~~~~~~~~~~~~~~~~~~

Below is an example of debugging the kernel at early boot time that shows some
useful commands to use. Within GDB, the "help" command can also be useful to
get more information about these commands. Other useful resources are the GDB
documentation and `ADI's GDB Command Reference <https://developer.analog.com/docs/codefusion-studio/latest/tutorials/gdb-tutorial/gdb-commands/>`_.

.. code-block:: bash

   (gdb) # Set kernel source code path
   (gdb) set substitute-path /usr/src/kernel <yocto-workspace>/build/tmp/work-shared/adsp-sc589-mini/kernel-source
   (gdb)
   (gdb) # hardware breakpoints at start_kernel
   (gdb) hbreak start_kernel
   Hardware assisted breakpoint 1 at 0xc0b00da0: file /usr/src/kernel/init/main.c, line 904.
   (gdb) c
   Continuing.

   Breakpoint 1, start_kernel () at /usr/src/kernel/init/main.c:905
   905             char *command_line;
   (gdb) l
   900     }
   901
   902     asmlinkage __visible __init __no_sanitize_address __noreturn __no_stack_protector
   903     void start_kernel(void)
   904     {
   905             char *command_line;
   906             char *after_dashes;
   907
   908             set_task_stack_end_magic(&init_task);
   909             smp_setup_processor_id();
   (gdb) bt
   #0  start_kernel () at /usr/src/kernel/init/main.c:905
   #1  0x00000000 in ?? ()
   (gdb)
