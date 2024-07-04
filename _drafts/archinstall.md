---
layout: post
title: Arch Install
---

This guide serves as a way for me to document my install and as such, provide other
people with a structured way to install Arch Linux so that we can say the phrase
**BTW, I use Arch**.

Head-up, this, the following guide goes through the manual install. I believe
that the automatic install is easy enough to not need a guide in the first place.
Also, the guide will follow closely the arch installation guide from [https://wiki.archlinux.org/title/installation_guide](https://wiki.archlinux.org/title/installation_guide).
Lastly, I am assuming that access to the internet is in place and the system is
succesfully booted from the USB stick. If not, please have a look at [Connect_to_the_internet](https://wiki.archlinux.org/title/installation_guide#Connect_to_the_internet).

## 0. Preparation

If you have a HiDPI screen, use:

```sh
setfont ter-132b
```

## 1. Disk Partitioning

Use [lsblk](https://wiki.archlinux.org/title/Device_file#lsblk) to find the system layout in its current state. In
a virtual machine, the partition is `vda`.
Once that is done, [fdisk](https://wiki.archlinux.org/title/fdisk), is going to be used for formatting the disk.
I am going to build a system with a standard partition table:

| Mount point | Partition                 | Partitio Type            | Size                |
| ----------- | ------------------------- | ------------------------ | ------------------- |
| /mnt/boot   | /dev/efi_system_partition | EFI system partition (1) | 1G                  |
| \[SWAP\]    | /dev/swap_partition       | Linux Swap (19)          | RAM size (TLDR)     |
| /mnt        | /dev/root_partition       | Linux                    | Remainder of device |

As a general guidance, swap has to be the same amount of RAM in order to allow the PC to hybernate.
My system has 64GB of RAM, however, I am most of the time using a quarter of it and sometimes reaching
half of it. So I will set my swap to be 32GB.

For the root partition an encryption will be used together with a BTRFS filesystem and I will use subvolumes for root, home.

## 2 Formatting

### 2.1. Format the EFI partition and SWAP

```sh
# format EFI partition
mkfs.fat -F 32 /dev/efi_system_partition

# format SWAP
mkswap /dev/swap_partition
# activate SWAP
swapon /dev/swap_partition
```

### 2.2. Format the root partition

I am using a `BTRFS` filesystem with encryption. This requires using [cryptsetup](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption)

1. Encrypt the partition with

```sh
cryptsetup luksFormat /dev/root_partition
```

2. Open the partition
   The partition requires a name. I call it `cryptroot` but you can call it however you want.

```sh
cryptsetup luksOpen /dev/root_partition cryptroot
```

3. Format the root partition

```sh
mkfs.btrfs /dev/mapper/cryptroot
```

## BTRFS

I am going for the following subvolume layout, inspired by [this](https://github.com/classy-giraffe/easy-arch?tab=readme-ov-file) and [this](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout). I am planning to use
[snapper](https://wiki.archlinux.org/title/Snapper) as my snapshot manager.
However, I am going to have my root directly at "@". Due to the nature of _BTRFS_,
this will impact the other subvolumes. In other words, rollbacks in root will not
rollback its subvolumes. See more info here

| Subvolume Number | Subvolume Name | Mountpoint            |
| ---------------- | -------------- | --------------------- |
| 1                | @              | /                     |
| 2                | @home          | /home                 |
| 3                | @snapshots     | /.snapshots           |
| 4                | @var_log       | /var/log              |
| 5                | @var_pkgs      | /var/cache/pacman/pkg |

To create subvolumes on the _btrfs_ filesystem, we first need to mount the root partition

1. Mount the openned partition and cd into it

```sh
mount /dev/mapper/cryptroot /mnt
cd /mnt
```

2. Create subvolumes

```sh
btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @var_log
btrfs subvolume create @var_pkgs
```

3. Umount

```sh
cd
umount /mnt
```

### Mounting

```sh
# create mnounting points
mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman/pkgs}
# mount the partitions
mount -o noatime,subvol=@ /dev/mapper/cryptroot /mnt
mount -o noatime,subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o noatime,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount -o noatime,subvol=@var_log /dev/mapper/cryptroot /mnt/var/log
mount -o noatime,subvol=@var_pkgs /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg

mount /dev/vda1 /mnt/boot
```

### Install essential packages (yes, vim is essential)

```sh
pacstrap -K /mnt base base-devel \
                 linux linux-headers linux-firmware \
                 linux-lts linux-lts-headers \
                 neovim git
```

### Generate fstab

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

### Essential Config

```sh
# enter the system
arch-chroot /mnt

# set the time
ln -sf /usr/share/zoneinfo/Region/City /etc/localtime
hwclock --systohc

# set locale
locale-gen
# uncomment the required locale in /etc/locale.gen
# add the same locale in /etc/locale.conf LANG=en_GB.UTF-8

# set root password
passwd

# make user
useradd -m -g users -G wheel tj
passwd tj
```

### Install rest of packages

```sh
# - Bootloader and file system tools: grub, grub-btrfs, efibootmgr, dosfstools
# - Networking essentials: networkmanager, network-manager-applet
# - Bluetooth utilities: bluez, bluez-utils
# - Printing system: cups
# - NVIDIA drivers: nvidia, nvidia-utils, nvidia-settings
# - Various utilities and fonts: sh-completion, openssh, reflector, flatpak, terminus-font
pacman -S grub grub-btrfs efibootmgr dosfstools\
          networkmanager network-manager-applet \
          bluez bluez-utils \
          cups \
          nvidia nvidia-utils nvidia-settings \
          sh-completion openssh reflector flatpak terminus-font snapper
```

### Edit `mkinitpio`

This step is required because I am using encryption

1. Edit `mkinitcpio.conf`

I added encrypt before fylesystems in hooks and btrfs in modules, see [here](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Unlocking_in_early_userspace) and [here](https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks)

```sh
# complete this bit
```

2. Regenerate the images

```sh
# regenerate images
mkinitcpio -p linux
```

## Setup GRUB

0. Install grub application

```sh
grub-install --target=x86_64-efi --efi-directory=<boot-directory> --bootloader-id=GRUB
```

1. copy UUID of the encrypted device (not the mapped one)

```sh
blkid
```

2. add the device to the default cmd:

UUID="539adfb2-ae29-4a11-a1b1-005437358cdc"

3. regenerate config

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

### Enable services
