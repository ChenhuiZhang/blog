# QEMU

*QEMU is a generic and open source machine emulator and virtualizer.*

## Boot the Linux

<!-- more -->

1. Download the kernel:

   `git clone https://github.com/torvalds/linux.git linux`

2. Install the tool-chain:

   `sudo apt-get install gcc-arm-linux-gnueabi`

3. Compile the kernel:

   `make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- vexpress_defconfig`

   `make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- menuconfig`

   `make ARCH=arm CROSS_COMPILE=arm-linux-gnueabi- -j8`

   > Tips:
   >
   > ** `qemu-system-arm -machine help` can list the support machines

   vexpress means: *express-a9          ARM Versatile Express for Cortex-A9*  and [spec](<http://infocenter.arm.com/help/topic/com.arm.doc.dui0448i/DUI0448I_v2p_ca9_trm.pdf>) here. We will run Linux on this ARM development board.

4. Install qemu:

   `sudo apt-get install qemu qemu-system-arm`

5. Prepare the rootfs

   There are many ways to choose rootfs, e.g. busybox, buildroot and some linux distribution. Here  we use buildroot as an example.

   a. Download the source code from: https://buildroot.org/downloads/buildroot-2019.02.2.tar.bz2

   b. Extra the file and run "make menuconfig"

   ```
   #Set to the proper processor if you use different CPU
   Target options --->
       Target Architecture (ARM (little endian))
       Target Binary Format (ELF)
       Target Architecture Variant (cortex-A9)
       Target ABI (EABI)
       Floating point strategy (Soft float)
       ARM instruction set (ARM)
   #I prefer this toolchain rather than buildroot's
   Toolchain --->
       Toolchain type (External toolchain)
       Toolchain (Sourcery CodeBench ARM 2014.05)
       Toolchain origin (Toolchain to be downloaded and installed)
   #This is the minimal setting one could use.
   System configuration --->
       Init system (BusyBox)
       /dev management (Dynamic using devtmpfs only)
       [*] Enable root login with password
       (root) Root password
           /bin/sh (busybox' default shell)
   #CPIO is more compatible and has readonly property when executing QEMU
   Filesystem images --->
       [*] cpio the root filesystem (for use as an initial RAM filesystem)
   ```

   The benefit to use buildroot is easy to deploy lots of tools and packages you want. E.g. dhcpcd, web server, python etc.

   c. Run "make" to build the image

   d. The final result will placed on buildroot/output/images/rootfs.cpio

6. Start the QEMU

   ```
   qemu-system-arm \
   -M vexpress-a9 \
   -m 1024 \
   -smp 4 \
   -dtb linux/arch/arm/boot/dts/vexpress-v2p-ca9.dtb \
   -kernel linux/arch/arm/boot/zImage \
   -nographic \
   -append "console=ttyAMA0" \
   -initrd ./buildroot-2019.02.2/output/images/rootfs.cpio
   ```

   Enjoy the Linux!

   > Tips:
   >
   > ** To quit the QEMU, Ctrl + a and x

## Make the network work

The network will auto setup when system start. Install one dhcp client in your rootfs then you will get the address from default dhcp server in QEMU. To allow SSH from host, also need to configure a port forward in QEMU, the whole command is:

```
-net user,hostfwd=tcp::10022-:22 -net nic
```

See this [link](<https://wiki.qemu.org/Documentation/Networking>) for detail.

## GDB with QEMU

There is a internal gdb-server in QEMU, enable it by pass below option:

```
-s
Shorthand for -gdb tcp::1234, i.e. open a gdbserver on TCP port 1234 (see gdb_usage).
```

and also with:

```
-S
Do not start CPU at startup (you must type ’c’ in the monitor).
```

Then run the gdb as a normal gdb client in your terminal:

```
gdb-multiarch --tui

(gdb) file linux/vmlinux
Reading symbols from linux/vmlinux...done.
(gdb) target remote:1234
Remote debugging using :1234
0x60000000 in ?? ()
(gdb)
```

Also run gdb in vscode, config the launch.json as below:

```
			...
                        "program": "${workspaceFolder}/vmlinux",
                        "miDebuggerPath": "/usr/bin/gdb-multiarch",
                        "miDebuggerServerAddress": "localhost:1234",
			...
```

## Share folder

To make it possible to share file between QEMU guest and host, we could mount a host folder to the QEMU guest by below command:

```
--fsdev local,id=kmod,path=$PWD/mnt,security_model=none -device virtio-9p-device,fsdev=kmod,mount_tag=share
```

So mnt folder in current host PWD will map to a share mount point in QEMU guest, then we could mount it to /mnt in QEMU guest system by editing the /etc/fstab like:


```
share           /mnt            9p      trans=virtio  0	0
```

## Kernel debug fs

Add below to /etc/fstab

```
debugfs    /sys/kernel/debug      debugfs  defaults  0 0
```
