---
layout: post
author: tudorjnu
title: |
  Personal Arch Install Guide
description: |
  I am using this guide to make ...
---
<!-- markdownlint-disable-file MD013 -->
> [!WARNING]
> This is still under construction. Do not follow!

## Pre-installation

This guide serves as a way for me to document my installation, so I can make it repeatable. In this way, it also serves as a structured way to install Arch Linux for other people so that we can all say the phrase **BTW, I use Arch**. Heads-up, the following guide goes through the manual installation. I believe that the automatic installation is easy enough to not need a guide in the first place. Also, the guide will follow closely the arch installation guide from [https://wiki.archlinux.org/title/installation_guide](https://wiki.archlinux.org/title/installation_guide).  To connect to the internet, check [Connect_to_the_internet](https://wiki.archlinux.org/title/installation_guide#Connect_to_the_internet).

I would be using a few key features:

* [BTRFS filesystem](https://wiki.archlinux.org/title/btrfs)
* [Snapper](https://wiki.archlinux.org/title/snapper) for snapshots
* [LVM on LUKS](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS)

This should be secure enough while keeping things _"simple"_.

### Basics

If you have a [HiDPI](https://wiki.archlinux.org/title/HiDPI#Linux_console_(tty)) screen:

```sh
setfont ter-132b
```

Change the keyboard layout if you need to (the default is us):

```sh
localectl list-keymaps
loadkeys <chosen key>
```

Check boot mode ([link](https://wiki.archlinux.org/title/installation_guide#Verify_the_boot_mode)). If you get an output for the following, you should be good to go:

```sh
cat /sys/firmware/efi/fw_platform_size
```

Check for network connectivity:

```sh
ping archlinux.org
```

If you do not get a response to the request, make sure you have a cable
connected or connect via wireless by using [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl).

Start the sshd if you want to connect from another pc. It makes the process a
lot easier:

```sh
systemctl start sshd
```

Set a password for the root (needed for the ssh):

```sh
passwd
```

### Disk partitioning

[Partitioning Scheme](../assets/img/partitioning.svg)

To make things simple and secured, I will create an [LVM on LUKS](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) setup. The idea is that a full partition would be encrypted, and then it can be altered as wished. Another benefit of this setup is that it allows for [suspend-to-disk](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption#With_suspend-to-disk_support) of the swap partition. This means that the device can be put to [hibernate](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation) and all the information will be saved onto the encrypted [swap](https://wiki.archlinux.org/title/swap) partition.

| Mount point | Partition                 | Partitio Type            | Size                |
| ----------- | ------------------------- | ------------------------ | ------------------- |
| /mnt/boot   | /dev/efi_system_partition | EFI system partition (1) | 1G                  |
| /mnt        | /dev/root_partition       | Linux                    | Remainder of device |

#### Create `boot` and `LUKS` partitions

To find the device we want to partition use `lsblk`. This will list all
block devices. In my virtual machine, my block device is called `vda`.

Create the following partitions using `fdisk`:

* A boot partition (size 1G)
* A LUKS partition with the rest of the memory

For more information have a look at the [archwiki](https://wiki.archlinux.org/title/partitioning).

> [!NOTE]
> You won't find a LUKS partition type, leave it to `Linux`

With the two partitions created, we can now format them:

```sh
mkfs.fat -F 32 /dev/<efi_system_partition>
```

For example with `mkfs.fat -F 32 /dev/vda1`.

Next step is to encrypt the second partition:

```sh
cryptsetup luksFormat /dev/<partition>
```

Now we need to open it and map it to a name. Mine will be called **cryptlvm**
but feel free to call it **mysuperamazingpartition** if you want. But be aware
that you will have to type the name for quite a few times.

```sh
cryptsetup open /dev/<partition> cryptlvm
````

#### Creating the [LVM](https://wiki.archlinux.org/title/LVM) volumes

Create a physical volume on top of the opened LUKS container:

```sh
pvcreate /dev/mapper/cryptlvm
```

Create a volume group (_i.e._ `vg0`):

```sh
vgcreate vg0 /dev/mapper/cryptlvm
```

Create all logical volumes on the volume group. As mentioned before, the two
logical volumes I need are `lvmroot` and `lvmswap`:

```sh
lvcreate -L 4G vg0 -n lvmswap
lvcreate -l 100%FREE vg0 -n lvmroot
```

Format the logical volumes:

```sh
mkswap /dev/vg0/lvmswap
mkfs.btrfs /dev/vg0/lvmroot
```

Enable swap:

```sh
swapon /dev/vg0/lvmswap
```

#### Make the [btrfs](https://wiki.archlinux.org/title/btrfs) subvolumes

I am going for the following subvolume layout, inspired by [this](https://github.com/classy-giraffe/easy-arch?tab=readme-ov-file) and [this](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout). I am planning to use [snapper](https://wiki.archlinux.org/title/Snapper) as my snapshot manager. A tip recommended by the wiki is to make a subvolume for things that you do not want to be includded in the snapshots.

| Subvolume Number | Subvolume Name | Mountpoint            |
| ---------------- | -------------- | --------------------- |
| 1                | @              | /                     |
| 2                | @home          | /home                 |
| 3                | @snapshots     | /.snapshots           |
| 4                | @var_log       | /var/log              |
| 5                | @var_pkgs       | /var/cache/pacman/pkg              |

To make the `btrfs` subvolumes, the partition has to be mounted:

```sh
mount /dev/vg0/lvmroot /mnt
cd /mnt
```

Now the subvolumes can be created as following:

```sh
btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @var_log
btrfs subvolume create @var_pkgs
btrfs subvolume create @snapshots
```

Now we can mount the subvolumes. First `cd` out of the root:

```sh
cd
umount /mnt
```

We can now mount the root:

```sh
mount -o subvol=@ /dev/vg/root /mnt
```

Create the directories:

```sh

mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman/pkg}
```

And mount the subvolumes:

```sh
mount -o subvol=@home /dev/vg/root /mnt/home
mount -o subvol=@snapshots /dev/vg/root /mnt/.snapshots
mount -o subvol=@var_log /dev/vg/root /mnt/var/log
mount -o subvol=@var_pkgs /dev/vg/root /mnt/var/cache/pacman/pkg
```

Finally, we can mount the `boot` partition:

```sh
mount /dev/vda1 /mnt/boot
```

### Installation

#### Install essential packages (yes, vim is essential)

```sh
pacstrap -K /mnt base base-devel \
                 linux linux-headers linux-firmware \
                 linux-lts linux-lts-headers \
                 neovim git btrfs-progs lvm2 \
```

#### Generate fstab

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

#### Essential Config

Enter the system:

```sh
arch-chroot /mnt
```

Set root password:

```sh
passwd
```

Set the time

```sh
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
hwclock --systohc
```

Create the hostname file `/etc/hostname`:

```sh
<yourhostname>
```

##### Set up the locale

Uncomment the required locale in `/etc/locale.gen` such as:

* en_GB.UTF-8
* en_US.UTF-8

Add the main locale in `/etc/locale.conf` and since we are there, add a
fallback:

```sh
LANG=en_GB.UTF-8
LANGUAGE=en_US.UTF-8
```

Generate the locale:

```sh
locale-gen
```

### Install rest of packages

<!-- TODO: fix the following section -->

The packages required are as following:

```sh
# - Bootloader and file system tools: grub, grub-btrfs, efibootmgr, dosfstools
# - Networking essentials: networkmanager, network-manager-applet 
# - Bluetooth utilities: bluez, bluez-utils
# - Printing system: cups
# - wifi: iw
# - NVIDIA drivers: nvidia, nvidia-utils, nvidia-settings
# - Intel or AMD drivers: amd-ucode or intel-ucode
# - Various utilities and fonts: sh-completion, openssh, reflector, flatpak, terminus-font
pacman -S grub grub-btrfs efibootmgr dosfstools mtools \
          networkmanager network-manager-applet os-prober sudo\
          bluez bluez-utils \
          cups \
          iw \
          nvidia nvidia-utils nvidia-settings \
          amd-ucode \
          bash-completion openssh reflector flatpak terminus-font snapper
```

Create your user

```sh
# make user
useradd -m -g users -G wheel tj
passwd tj
```

Add yourself to sudoers file `/etc/sudoers`:

```sh
echo "tj ALL=(ALL) ALL" >> /etc/sudoers.d/tj
```

### Edit `mkinitpio` at `/etc/mkinitcpio.conf`

This step is required because I am using encryption

I added encrypt before fylesystems in hooks and btrfs in modules, see [here](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Unlocking_in_early_userspace) and [here](https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks).

Two changes are necessary here. Firstly, the `btrfs` module has to be loaded.
Next, I changed the `HOOKS` to `systemd` stuff as following:

<!-- TODO: replace this -->
```md
MODULES=(btrfs)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block *encrypt* lvm2 filesystems fsck)
```

Regenerate the images (needs to be done for every kernel):

```sh
mkinitcpio -p linux
mkinitcpio -p linux-lts
```

### Setup GRUB

Install grub application

```sh
grub-install --target=x86_64-efi --efi-directory=<boot-directory> --bootloader-id=GRUB
```

Copy UUID of the encrypted device (not the mapped one):

```sh
blkid
```

Add the device to the default cmd in `/etc/default/grub`:

```sh
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 rd.luks.name=<device-UUID>=cryptlvm root=/dev/vg0/root resume=/dev/vg0/swap quiet"
```

Regenerate config:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

### Enable services

```sh
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable cups.service
systemctl enable sshd
systemctl enable reflector.timer
```

Do a restart 󰒲:

```sh
exit
umount -a
reboot
```

At this point, we have a clean installation. From here, you have a clean slate to
work on. In the following parts, I will set up the Swap encryption and snapper.

## Snapper ([link](https://wiki.archlinux.org/title/Snapper))

### Prerequisites

Before setting up everything the packages `cron` and `snapper` are required:

```sh
pacman -S cron snapper
```

Now let's start the `cron` `systemctl` service:

```sh
systemctl enable cronie.service
systemctl start cronie.service
```

### Setup

I will assume that the subvolume for snapper was created as in [make-the-btrfs-subvolumes](#make-the-btrfs-subvolumes).

For the setup, we first have to delete the folder `/.snapshots`:

```sh
rm -rf /.snapshots
```

Generate the snapper config:

```sh
snapper -c root create-config /
```

Delete the folder created by `snapper`:

```sh
rm -rf /.snapper
```

Mount the snapper subvolume:

```sh
mount -o subvol=@snapshots /dev/vg/root /.snapshots
```

Make the mount permanent:

```sh
mount -a 
```

And give the folder 750 permissions:

```sh
chmod 750 /.snapshots
```

## General Setup

### The desktop environment route

If you are interested in a full desktop environment, you could just install one
and be done. For example:

```sh
sudo pacman -S gnome gnome-tweaks
sudo systemctl enable gdm
```

After a restart you should be greeted by [GDM]().

### The window manager route

<!-- TODO: fill up this section -->

If you are like me, however, and like to have a window manager, we got a bit
more work to do. First, we need to get the rewquired packages. Yours might differ but here are my
choices:

* Compositor: picom
* Display Manager: ly
* Window Manager: qtile
* Wallppaper: nitrogen
* Programs:
  * Browser: Firefox
  * Terminal: Alacritty
  * File Manager: lf
* Extras:
  * Customization: lxappearance
* Display drivers: nvidia, nvidia-lts, nvidia-settings, nvidia-utils
* Sound System: pipewire
* Fonts:  ttf-ubuntu-nerd, ttf-ubuntu-mono-nerd ttf-jetbrains-mono-nerd

```sh
sudo pacman -S ly picom qtile \
               lxappearance nitrogen alacritty lf ly \
               nvidia, nvidia-lts, nvidia-settings, nvidia-utils amd-ucode \
               pipewire pipewire-docs \
```

### Fonts

```sh
 sudo pacman -S ttf-ubuntu-nerd ttf-ubuntu-mono-nerd ttf-jetbrains-mono-nerd
```

### Display Server

They come in two flavours: 1) xorg, and 2) [wayland](https://wiki.archlinux.org/title/Wayland).

To install xorg:

```sh
pacman -S xorg
```

For wayland we will need a server that provides a compatibility layer with X11:

```sh
pacman -S xorg-xwayland
```

