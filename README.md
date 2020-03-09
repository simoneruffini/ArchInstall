---
author:
- Simone Ruffini
date: March 2020
title: Arch install for DELL XPS 9570
---

Read everitying while keeping an eye to [**Arch installation
Guide**](https://wiki.archlinux.org/index.php/Installation_guide) and
[GloriusEggroll](https://www.gloriouseggroll.tv/arch-linux-efi-install-guide/)

Pre-Installation
================

Create Bootable Media
---------------------

Create a bootable device with RUFUS with dd method () or by using dd.

-   Windows: RUFUS with dd

-   \*nix:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
sudo dd bs=4M if=<path/to/input>.iso of=/dev/sd<?> conv=fdatasync  status=progress
```

Boot in arch linux. Remember to disable Secure Boot and RAID intel
otherwise nvme devices wont be shown.

Verify the boot mode
--------------------

If [UEFI](https://wiki.archlinux.org/index.php/UEFI) mode is enabled on
an [UEFI](https://wiki.archlinux.org/index.php/UEFI) motherboard,
Archiso will [boot](https://wiki.archlinux.org/index.php/Boot) Arch
Linux accordingly via
[systemd-boot](https://wiki.archlinux.org/index.php/Systemd-boot). To
verify this, list the efivars directory:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
ls /sys/firmware/efi/efivars
```

Connect to the internet
-----------------------

To set up a network connection, go through the following steps: Ensure
your [network](https://wiki.archlinux.org/index.php/Network_interface)
interface is listed and enabled, for example with
[ip-link(8)](https://jlk.fjfi.cvut.cz/arch/manpages/man/ip-link.8):

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
ip link
```

[Connect to the wireless
LAN](https://wiki.archlinux.org/index.php/Wireless_network_configurationc).
Try with wifi-menu of
[netctl](https://wiki.archlinux.org/index.php/Netctl) package:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
wifi-menu
```

If somethig goes wrong you can modify the network profile with\
`vim /etc/netctl/<profile_name>`

The connection may be verified with
[ping](https://en.wikipedia.org/wiki/ping):

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
ping archlinux.org
```

Update the system clock
-----------------------

Use
[timedatectl(1)](https://jlk.fjfi.cvut.cz/arch/manpages/man/timedatectl.1)
to ensure the system clock is accurate:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
timedatectl set-ntp true
```

Partition the disks
-------------------

When recognized by the live system, disks are assigned to a [block
device](https://wiki.archlinux.org/index.php/Device_file#Block_devices)
such as `/dev/sda `or `/dev/nvme0n1`. To identify these devices, use
[lsblk](https://wiki.archlinux.org/index.php/Lsblk) or
[fdisk](https://wiki.archlinux.org/index.php/Fdisk).

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
lsblk
```

Results ending in `rom`, `loop` or `airoot` may be ignored. The
following [partitions](https://wiki.archlinux.org/index.php/Partition)
are **required** for a chosen device:

-   One partition for the root directory /.

-   If [UEFI](https://wiki.archlinux.org/index.php/UEFI) is enabled, an
    [EFI system
    partition](https://wiki.archlinux.org/index.php/EFI_system_partition).

Using [LVM](https://wiki.archlinux.org/index.php/LVM) and with a
graphical interface to partiton disk use
[cfdisk](https://jlk.fjfi.cvut.cz/arch/manpages/man/cfdisk.8).

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
cfdisk /dev/<device>
```

Inside cfdisk:

1.  Make a GPT partition table if not already exisiting

2.  New 512M partition for EFI, change partition type with `EFI System`

3.  Remaining of disk size (no option) with default type
    `Linux filesystem`

4.  write and quit

### LVM partitions

Visit [LVM ita](https://wiki.archlinux.org/index.php/LVM_(Italiano)).
**In this section the storage device will be sda**!

1.  Create physical volume and check with `pvdisplay`

    ``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
    pvcreate /dev/sda2 
    ```

2.  Create Volume Group and check with `vgdisplay`

    ``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
    vgcreate VolGrp0 /dev/sda2
    ```

3.  Create Logical Volumes and check with `lvldisplay`

    ``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
    lvcreate -L 20G VolGrp0 -n root
    lvcreate -C y -L 8G VolGrp0 -n swap
    lvcreate -l 100%FREE VolGrp0 -n home
    ```

Format the partitions
---------------------

Once the partitions have been created, each must be formatted with an
appropriate [file
system](https://wiki.archlinux.org/index.php/File_system).

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
mkfs.fat -F 32 /dev/<boot_partition> 
mkfs.f2fs -l root /dev/VolGrp00/root
mkfs.f2fs -l home /dev/VolGrp0/home
```

Initialize the swap

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
mkswap /dev/VolGrp0/swap
swapon /dev/VolGrp0/swap
```

The partitioning scheme and mount point of the EFI Sytem Partition (ESB)
are tided to the type of booloader used. In this guide
[systemd-boot](https://wiki.archlinux.org/index.php/Systemd-boot) will
be used (look at installation section). The mount point of the ESB
partition in Systemd-boot needs to contain kernel and initramfs files.
So boot is effectively the ESP.

Mount the file systems
----------------------

Non existen directories must be created first

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
mount /dev/VolGrp0/root /mnt
mkdir /mnt/home
mkdir /mnt/boot
mount /dev/VolGrp0/home /mnt/home
mount /dev/<boot_partition> /mnt/boot
```

Installation
============

Select mirrors
--------------

Packages to be installed must be downloaded from [mirror
servers](https://wiki.archlinux.org/index.php/Mirrors), which are
defined in\
`/etc/pacman.d/mirrorlist`. The higher a mirror is placed in the list,
the more priority it is given when downloading a package. This file will
later be copied to the new system by pacstrap, so it is worth getting
right.\
Make a backup of the mirror list:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist.backup
```

Install
[pacman-contrib](https://git.archlinux.org/pacman-contrib.git/about/)
package containing the `rankmirrors` script.

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
pacman -Sy
pacman -S pacman-contrib
```

If pacman is not working then probably all servers in the mirror list
are commented. Uncomment one by modyfing `mirrorlist` and after the
install recomment it.

To be shure that all servers are availble for ranking run tht command to
uncomment all the lines in `mirrorlist`: \|sed -i 's/\^\#Server/Server/'
/etc/pacman.d/mirrorlist.backup\| Now we will run `mirrors`, it can take
a wile and no output is made in the process so just wait and monitor on
another tty with `top`. The script will rank the first 6 best mirrors
delting the other ones.

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
rankmirrors -n 6 /etc/pacman.d/mirrorlist.backup > /etc/pacman.d/mirrorlist
```

Copy the remaining mirrors in the file.

Install essential packages
--------------------------

Use the
[pacstrap](https://projects.archlinux.org/arch-install-scripts.git/tree/pacstrap.in)
script to install the
[base](https://www.archlinux.org/packages/?name=base) package, Linux
[kernel](https://wiki.archlinux.org/index.php/Kernel) and firmware for
common hardware. The
[base](https://www.archlinux.org/packages/?name=base) package does not
include all tools from the live installation, so installing other
packages may be necessary for a fully functional base system. In
particular, consider installing:

-   userspace utilities for the management of [file
    systems](https://wiki.archlinux.org/index.php/File_systems) that
    will be used on the system,

-   utilities for accessing
    [RAID](https://wiki.archlinux.org/index.php/RAID) or
    [LVM](https://wiki.archlinux.org/index.php/LVM) partitions,

-   specific firmware for other devices not included in
    [linux-firmware](https://www.archlinux.org/packages/?name=linux-firmware),

-   software necessary for
    [networking](https://wiki.archlinux.org/index.php/Networking),

-   a [text editor](https://wiki.archlinux.org/index.php/Text_editor),

-   packages for accessing documentation in
    [man](https://wiki.archlinux.org/index.php/Man) and
    [info](https://wiki.archlinux.org/index.php/Info) pages:
    [man-db](https://www.archlinux.org/packages/?name=man-db),
    [man-pages](https://www.archlinux.org/packages/?name=man-pages) and
    [texinfo](https://www.archlinux.org/packages/?name=texinfo).

To [install](https://wiki.archlinux.org/index.php/Install) other
packages or package groups, append the names to the pacstrap command
above (space separated) or use
[pacman](https://wiki.archlinux.org/index.php/Pacman) while [chrooted
into the new
system](https://wiki.archlinux.org/index.php/Installation_guide#Chroot).
For comparison, packages available in the live system can be found in
[packages.x86\_64](https://projects.archlinux.org/archiso.git/tree/configs/releng/packages.x86_64).

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
pacstrap /mnt base base-devel linux linux-firmware vim man-db man-pages lvm2
```

Configure the system
====================

Generate Fstab
--------------

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
genfstab -U -p /mnt >> /mnt/etc/fstab
```

Check if there is a entry for every partition and the one for swap too
with `vim`.

Chroot
------

Now we are going to
[chroot](https://wiki.archlinux.org/index.php/Change_root) into our
newly installed system and begin to configure its booting, time, and
language:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
arch-chroot /mnt
```

Time zone
---------

Set the [time zone](https://wiki.archlinux.org/index.php/Time_zone):

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
 ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
```

Run [hwclock(8)](https://jlk.fjfi.cvut.cz/arch/manpages/man/hwclock.8)
to generate `/etc/adjtime`:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
hwclock --systohc --utc
```

Localization
------------

Uncomment `en_US.UTF-8 UTF-8` and other needed
[locale](https://wiki.archlinux.org/index.php/Locale) in
`/etc/locale.gen`, and generate them with:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
locale-gen
```

Create the
[locale.conf(5)](https://jlk.fjfi.cvut.cz/arch/manpages/man/locale.conf.5)
file, set the `LANG`
[variable](https://wiki.archlinux.org/index.php/Variable) as your
language:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
echo LANG=en_US.UTF-8 > /etc/locale.conf
export LANG=en_US.UTF-8
```

Network configuration
---------------------

### Hostname

Create the [hostname](https://wiki.archlinux.org/index.php/Hostname)
file:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
echo DellXPS > /etc/hostname
```

Add matching entries to
[hosts(5)](https://jlk.fjfi.cvut.cz/arch/manpages/man/hosts.5):

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
vim /etc/hosts
------------------------------------------------------------------------------------------
127.0.0.1   localhost
::1         localhost
127.0.1.1   DellXPS.localdomain DellXPS
```

### Network Manager

Install
[NetworkManager](https://wiki.archlinux.org/index.php/NetworkManager) as
our network manager. NetworkMangaer has and internal dhcp service, so if
DHCPCD is installed remove berforehand.\
Install
[networkmanager](https://wiki.archlinux.org/index.php/NetworkManager)
and enable it as service:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
pacman -S NetworkManager
systemctl enable NetworkManager.service
```

Enable trim support
-------------------

For safe, weekly
[TRIM](https://wiki.archlinux.org/index.php/Solid_state_drive#TRIM)
service on
[SSDs](https://wiki.archlinux.org/index.php/Solid_state_drive) and all
other devices that enable
[TRIM](https://wiki.archlinux.org/index.php/Solid_state_drive#TRIM)
support:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
systemctl enable fstrim.timer
```

Enabling multilib and Arch AUR
------------------------------

If you are running a 64bit system then you need to enable the [multilib
repository](https://wiki.archlinux.org/index.php/Official_repositories#multilib).\
Uncomment in `/etc/pacman.conf`:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
[multilib]
Include = /etc/pacman.d/mirrorlist
```

Update the sistem with pacman.

Boot loader
-----------

Install
[systemd-boot](https://wiki.archlinux.org/index.php/Systemd-boot)
[bootloader](https://wiki.archlinux.org/index.php/Boot_loader) Recheck
if the EFI variables are mounted

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
mount -t efivarfs efivarfs /sys/firmware/efi/efivars
```

With the ESP mounted to `/boot`, use
[bootctl(1)](https://jlk.fjfi.cvut.cz/arch/manpages/man/bootctl.1) to
install systemd-boot into the EFI system partition by running:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
bootctl --path=/boot install
```

This will copy the systemd-boot boot loader to the EFI partition, it
will then set systemd-boot as the default EFI application (default boot
entry) loaded by the EFI Boot Manager.\
Create configuration file to add an entry for Arch Linux:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
vim /boot/loader/entries/arch.conf
------------------------------------------------------------------------------------------
title Arch Linux
linux /vmlinuz-linux
initrd /initramfs-linux.img
options root=/dev/mapper/VolGrp0-root rw
```

The
[`option`](https://wiki.archlinux.org/index.php/Systemd-boot#Adding_loaders)
tag is the command line options to pass to the EFI program or [kernel
parameters](https://wiki.archlinux.org/index.php/Kernel_parameters). The
parameter
[`root`](https://wiki.archlinux.org/index.php/Kernel_parameters#Parameter_list)
tells ther kernel where the root file system partition is to be found.
`root` accepts [persistent block device
naming](https://wiki.archlinux.org/index.php/Persistent_block_device_naming)
but in our case since we are using LVM, all device blocks are persitent.

If `root` is not set properly the kernel will load but not finding the
root partition it will create a temporary one and put you in an
emergency shell.

If at reboot the bootloader is not shown you can either press `space` at
boot or change the `boot/loader/loader.conf` file adding `timeout 3`.

Add Intel microcode
-------------------

Install [microcode](https://wiki.archlinux.org/index.php/Microcode):

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
pacman -S intel-ucode
```

Update the arch systemd-boot loader file to load microcode:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
vim /boot/loader/entries/arch.conf
------------------------------------------------------------------------------------------
...
initrd /intel-ucode.img
initrd /initramfs-linux.img
...
```

Root password
-------------

Set the root [password](https://wiki.archlinux.org/index.php/Password):

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
passwd
```

User setup
----------

Add a default
[user](https://wiki.archlinux.org/index.php/Users_and_groups) with:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
useradd -m -g users -G wheel,storage,power -s /bin/bash simone
```

set a [password](https://wiki.archlinux.org/index.php/Password) for that
user:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
passwd simone 
```

Setting up sudoers
------------------

Edit the sudoers file to give this user
[sudo](https://wiki.archlinux.org/index.php/Sudo#Configuration)
privileges. The configuration file for sudo is in `/etc/sudoers`. It
should always be edited with the
[visudo(8)](https://jlk.fjfi.cvut.cz/arch/manpages/man/visudo.8)
command. `visudo` locks the `sudoers` file, saves edits to a temporary
file, and checks that file's grammar before copying it to
`/etc/sudoers`.\
Uncomment:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
visudo
------------------------------------------------------------------------------------------
#%wheel ALL=(ALL) ALL
```

Make sudoers require typing the root password instead of their own
password by adding:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
visudo
------------------------------------------------------------------------------------------
Defaults rootpw
```

LVM checks
----------

Create `lvm2`, `udev` hooks and enable `dm_mod` module in
[mkinitcpio](https://wiki.archlinux.org/index.php/Mkinitcpio):

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
vim /minted/mkinitcpio.conf
------------------------------------------------------------------------------------------
HOOKS="base udev ... lvm2 filesystems"
...
MODULES="dm_mod..."
```

This is needed otherwise the kernel is unable to find the lvm
partitions. Pay attention to the order in HOOKS, this is ther order in
which the kernel loads the modules so any wrong changes could dtop the
loading of the kernel.\
If changes to the file are made, `initramfs` has to be
[rebuilt](https://wiki.archlinux.org/index.php/Mkinitcpio#Installation)
with:

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
mkminitcpio -p linux
```

Install additional Packages
---------------------------

``` {.bash fontsize="\\small" bgcolor="ArchCode" frame="single"}
pacman -S bash-completion vi
```

check this things

-   add systemd pacman hook to update systemd

-   [xdg-user-dirs-update.service](https://wiki.archlinux.org/index.php/General_recommendations#User_directories)
