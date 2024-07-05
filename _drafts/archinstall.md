# ArchInstall | A personal guide

<!-- markdownlint-disable-file MD013 -->

## Pre-installation

This guide serves as a way for me to document my installation, so I can make it repeatable. In this way, it also serves as a structured way to install Arch Linux for other people so that we can all say the phrase **BTW, I use Arch**. Head-up, the following guide goes through the manual installation. I believe that the automatic installation is easy enough to not need a guide in the first place. Also, the guide will follow closely the arch installation guide from [https://wiki.archlinux.org/title/installation_guide](https://wiki.archlinux.org/title/installation_guide). Lastly, I am assuming that access to the internet is in place, and the system is succesfully booted from the USB stick. If not, please have a look at [Connect_to_the_internet](https://wiki.archlinux.org/title/installation_guide#Connect_to_the_internet). For any issues you are encountering, check the friendly manual RTFM 󰱸.

I would be using a few key features:

1. BTRFS filesystem
2. Snapper for snapshots
3. Qtile

### Basics

```sh
# set the font for HiDPI screen
setfont ter-132b

# set the keyboard layout
localectl list-keymaps
loadkeys <chosen key>
```

### Check boot mode ([link](https://wiki.archlinux.org/title/installation_guide#Verify_the_boot_mode))

If you get an output for the following, you should be good to go:

```sh
cat /sys/firmware/efi/fw_platform_size
```

### Disk Partitioning

Use [lsblk](https://wiki.archlinux.org/title/Device_file#lsblk) to find the system layout in its current state. In a virtual machine, the partition is `vda`. Once that is done, [fdisk](https://wiki.archlinux.org/title/fdisk), is going to be used for formatting the disk. I am going to build a system with a standard partition table:

| Mount point | Partition                 | Partitio Type            | Size                |
| ----------- | ------------------------- | ------------------------ | ------------------- |
| /mnt/boot   | /dev/efi_system_partition | EFI system partition (1) | 1G                  |
| \[SWAP\]    | /dev/swap_partition       | Linux Swap (19)          | RAM size (TLDR)     |
| /mnt        | /dev/root_partition       | Linux                    | Remainder of device |

As a general guidance, swap has to be the same amount of RAM to allow the PC to hibernate. My system has 64GB of RAM, however, I am most of the time using a quarter of it, and sometimes reaching half of it. So I will set my swap to be 32GB.

For the root partition an encryption will be used together with a *BTRFS* filesystem, and I will use subvolumes for root, home.

More information can be found on [archwiki](https://wiki.archlinux.org/title/partitioning)

#### Disk Formatting

##### 2.1. Format the EFI partition and SWAP

```sh
# format EFI partition
mkfs.fat -F 32 /dev/efi_system_partition

# format SWAP
mkswap /dev/swap_partition

# activate SWAP
swapon /dev/swap_partition
```

##### 2.2. Format the root partition

As mentioned, I am going to use BTRFS with encryption. Encryption has to come
before formatting the partition, so it has to be done now. For this, I will be using [cryptsetup](https://wiki.archlinux.org/title/Dm-crypt/Device_encryption).

First, the partition has to be encrypted:

```sh
# encrypt the partition
cryptsetup luksFormat /dev/root_partition
```

Then it has to be opened. The partition requires a name. I call it `cryptroot`, but you can call it however you want.

```sh
cryptsetup luksOpen /dev/root_partition cryptroot
```

Now we can format the partition with `btrfs`:

```sh
mkfs.btrfs /dev/mapper/cryptroot
```

### BTRFS

I am going for the following subvolume layout, inspired by [this](https://github.com/classy-giraffe/easy-arch?tab=readme-ov-file) and [this](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout). I am planning to use [snapper](https://wiki.archlinux.org/title/Snapper) as my snapshot manager. A tip recommended by the wiki is to make a subvolume for things that you do not want to be includded in the snapshots.

| Subvolume Number | Subvolume Name | Mountpoint            |
| ---------------- | -------------- | --------------------- |
| 1                | @              | /                     |
| 2                | @home          | /home                 |
| 3                | @snapshots     | /.snapshots           |
| 4                | @var_log       | /var/log              |

To create subvolumes on the *btrfs* filesystem, we first need to mount the root partition

#### Mount the openned partition and cd into it

```sh
mount /dev/mapper/cryptroot /mnt
cd /mnt
```

#### Create subvolumes

```sh
btrfs subvolume create @
btrfs subvolume create @home
btrfs subvolume create @var_log
btrfs subvolume create @var_pkgs
```

#### Umount

```sh
cd
umount /mnt
```

### Mounting the partitions

```sh
# create mnounting points
mkdir /mnt/
mkdir -p /mnt/{boot,home,.snapshots}

# mount the partitions
mount -o noatime,subvol=@ /dev/mapper/cryptroot /mnt
mount -o noatime,subvol=@home /dev/mapper/cryptroot /mnt/home
mount -o noatime,subvol=@snapshots /dev/mapper/cryptroot /mnt/.snapshots
mount -o noatime,subvol=@var_log /dev/mapper/cryptroot /mnt/var/log
mount -o noatime,subvol=@var_pkgs /dev/mapper/cryptroot /mnt/var/cache/pacman/pkg

mount /dev/vda1 /mnt/boot
```

### Installation

#### Install essential packages (yes, vim is essential)

```sh
pacstrap -K /mnt base base-devel \
                 linux linux-headers linux-firmware \
                 linux-lts linux-lts-headers \
                 neovim git btrfs-progs  \
```

#### Generate fstab

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

#### Essential Config

```sh
# enter the system
arch-chroot /mnt

# set root password
passwd

# make user
useradd -m -g users -G wheel tj
passwd tj

# set the time
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
hwclock --systohc

# uncomment the required locale in /etc/locale.gen
# add the same locale in /etc/locale.conf LANG=en_GB.UTF-8
# set locale
locale-gen

# create the hostname file /etc/hostname
<yourhostname>

```

### Install rest of packages

TODO: add Wi-Fi

```sh
# - Bootloader and file system tools: grub, grub-btrfs, efibootmgr, dosfstools
# - Networking essentials: networkmanager, network-manager-applet 
# - Bluetooth utilities: bluez, bluez-utils
# - Printing system: cups
# - wifi: iw
# - NVIDIA drivers: nvidia, nvidia-utils, nvidia-settings
# - Various utilities and fonts: sh-completion, openssh, reflector, flatpak, terminus-font
pacman -S grub grub-btrfs efibootmgr dosfstools\
          networkmanager network-manager-applet \
          bluez bluez-utils \
          cups \
          iw \
          nvidia nvidia-utils nvidia-settings \
          intel-ucode \
          bash-completion openssh reflector flatpak terminus-font snapper
```

### Edit `mkinitpio`

This step is required because I am using encryption

I added encrypt before fylesystems in hooks and btrfs in modules, see [here](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Unlocking_in_early_userspace) and [here](https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks)

```md
MODULES=(btrfs)
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block *encrypt* lvm2 filesystems fsck)
```

Regenerate the images (needs to be done for every kernel)

```sh
# regenerate images
mkinitcpio -p linux
mkinitcpio -p linux-lts
```

### Setup GRUB

Install grub application

```sh
grub-install --target=x86_64-efi --efi-directory=<boot-directory> --bootloader-id=GRUB
```

Copy UUID of the encrypted device (not the mapped one)

```sh
blkid
```

Add the device to the default cmd:

```sh
# /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet cryptdevice=UUID=6ec313a9-cdca-4127-b64e-ed463a78fbff:crypptroot root=/dev/mapper/cryptroot"
```

Regenerate config:

```sh
grub-mkconfig -o /boot/grub/grub.cfg
```

Do a restart 󰒲

### Enable services

```sh
systemctl enable NetworkManager
systemctl enable bluetooth
systemctl enable cups.service
systemctl enable sshd
systemctl enable reflector.timer
```

At this point, we have a clean installation. From here, you have a clean slate to
work on. In the following parts, I will set up the Swap encryption and snapper.

### Swap Encryption ([link](https://wiki.archlinux.org/title/dm-crypt/Swap_encryption#With_suspend-to-disk_support))

#### Preparations

Disable the partition first:

```sh
swapoff /dev/device
```

Format the partition:

```sh
cryptsetup luksFormat /dev/device
```

Open the partition:

```sh
cryptsetup open /dev/device cryptswap
```

Create swap filesystem inside the mapped partition:

```sh
mkswap /dev/mapper/cryptswap
```

Add the mapped partition to `/etc/fstab` by adding the following line:

```sh
/dev/mapper/cryptswap swap swap defaults 0 0
```

Setup system to resume from `/dev/mapper/cryptswap`. Add kernel parameter by
appending it to `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub`:

```sh
resume=/dev/mapper/cryptswap
```

Example can be:

```sh
kernel /vmlinuz-linux cryptdevice=/dev/sda2:rootDevice root=/dev/mapper/rootDevice resume=/dev/mapper/swapDevice ro
```

#### mkinitcpio hook

## Setup Snapper ([link](https://wiki.archlinux.org/title/Snapper))

## Troubleshoot

1. Some packages are not being downloaded:

```bash
pacman -Sy archlinux-keyring
```
