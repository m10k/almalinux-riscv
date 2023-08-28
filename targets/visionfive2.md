# AlmaLinux builds on StarFive VisionFive2

This page details how I use StarFive VisionFive2 boards to natively
build AlmaLinux 9 packages. Because RISC-V is not supported by any
AlmaLinux version yet, I am using Fedora 38 as build environment.

## Hardware

* StarFive VisionFive2 SBC
* 2TB m.2 SSD
* 64GB eMMC
* UART Serial (for debugging)
* 32GB microSD (for initial boot)


## Software

StarFive provides a Debian image[^1] which we will use as the starting
point. To build RPM packages, we need rpmbuild and mock, which we could
install on the Debian image, but we will make our life a lot easier by
using an RPM-based distribution. Fedora provides RISC-V builds[^2] for
a different board, but since we only need the userspace, we won't let
that bother us and download one of the Fedora 38 images.


To combine the Debian bootloader and kernel with the Fedora userspace,
we first need to write the Debian image to a microSD card. Assuming that
our card is `/dev/mmcblk0` and the image is called
`starfive-jh7110-202306-SD-minimal-desktop.img.bz2`, we can write the
image to the microSD card like this:

    # bzcat starfive-jh7110-202306-SD-minimal-desktop.img.bz2 > /dev/mmcblk0

This command created four partitions on the microSD, of which the fourth
one is the root partition containing the Debian userspace. What we're
going to do next is replace the Debian root partition with the one in the
Fedora image.


To get the root partition from the Fedora image, we first have to unpack
it and have a look at the partition table inside.

    # xz -d Fedora-Developer-38-20230519.n.0-nvme.raw.img.xz
    # fdisk -l Fedora-Developer-38-20230519.n.0-nvme.raw.img

This will output something like this:

```
Disk Fedora-Developer-38-20230519.n.0-nvme.raw.img: 12 GiB, 12884901888 bytes, 25165824 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: CF61CC1F-7D59-4BDE-A9DF-8C5AD00A039B

Device                                           Start      End  Sectors Size Type
Fedora-Developer-38-20230519.n.0-nvme.raw.img1      34  4194337  4194304   2G Linux filesystem
Fedora-Developer-38-20230519.n.0-nvme.raw.img2 4194338 25165790 20971453  10G Linux filesystem
```

The larger of the two partitions is the root partition, and we can copy it
over the Debian root partition by passing the start sector and number of
sectors to `dd`.

    # dd if=Fedora-Developer-38-20230519.n.0-nvme.raw.img skip=4194338 count=20971453 of=/dev/mmcblk0p4

Once this command is done, we have a bootable Fedora 38 image for the
StarFive VisionFive2 board.


### Rebuilding the kernel

There is one big caveat though: The configuration of the Debian kernel does
not support EXT4 security labels. This will cause mock to fail when it tries
to install shadow-utils in the build environment. The error message that it
emits looks something like this.

    cpio: cap_set_file failed - Operation not supported

We need to rebuild the kernel with the missing support. So we first need
to get the sources. We perform all of the following steps on the target.

    $ git clone https://github.com/starfive-tech/linux -b JH7110_VisionFive2_devel

When I rebuilt my kernel, HEAD was at `4964ce0a869e92df26331833894c9d0fd84d80f3`,
so if the kernel build fails for you, try checking out the same commit.

Once you have the sources, run `make menuconfig` to edit the configuration. The
required setting is in `File systems` and is called `Ext4 Security Labels`.
Enable this option by selecting it and hitting `y` or the space bar. Next, save
the configuration and exit. Or, in case you need the board to act as an NFS
server, you might also want to enable NFS server support. The option for this is
located in `File systems ---> Network File Systems` and is called
`NFS server support`.
Next, run the following command and wait around 30 minutes for it to complete.

    $ make -j 4

This will create the kernel image `arch/riscv/boot/Image.gz`. Copy this file to
`/boot` and rename it appropriately (the name is important because we need to use
dracut to create a new initramfs). If necessary, use `su` to switch to root. The
default root password on Fedora RISC-V images is `fedora_rocks!`.

    # cp arch/riscv/boot/Image.gz /boot/vmlinuz-5.15.0+

In the next step, we need to install the kernel modules so dracut can find them.

    # make modules_install

Now we can tell dracut to generate a new initramfs.

    # dracut /boot/initrd.img-5.15.0+ 5.15.0+

Finally, we need to edit the uboot configuration so it will boot the correct
kernel. We do that by modifying the default entry in
`/boot/extlinux/extlinux.conf` to looks something like this:

```
label l0
        menu label Fedora Linux 38 kernel 5.15.0+
        linux /vmlinuz-5.15.0+
        initrd /initrd.img-5.15.0+
        fdtdir /dtbs
        append root=/dev/mmcblk1p4 rw console=tty0 console=ttyS0,115200 earlycon rootwait stmmaceth=chain_mode:1 selinux=0
```

And that's it. Now we can reboot the board by issuing a `shutdown -r now` and
wait for it to come up again. If that does not happen, use the UART serial to
see what is going on.


[^1]: StarFive VisionFive2 Debian Image: https://onedrive.live.com/?authkey=%21AAAs7oQT2992Eg8&id=8DAB77C937089CE9%21316617&cid=8DAB77C937089CE9
[^2]: Fedora RISC-V images: http://fedora.riscv.rocks/koji/tasks?state=closed&view=flat&method=createAppliance&order=-id
