# [ArchInstall](archinstall.md) | A personal guide

<!-- markdownlint-disable-file MD013 -->

## Pre-installation

This guide serves as a way for me to document my installation, so I can make it repeatable. In this way, it also serves as a structured way to install Arch Linux for other people so that we can all say the phrase **BTW, I use Arch**. Head-up, the following guide goes through the manual installation. I believe that the automatic installation is easy enough to not need a guide in the first place. Also, the guide will follow closely the arch installation guide from [https://wiki.archlinux.org/title/installation_guide](https://wiki.archlinux.org/title/installation_guide). Lastly, I am assuming that access to the internet is in place, and the system is succesfully booted from the USB stick. If not, please have a look at [Connect_to_the_internet](https://wiki.archlinux.org/title/installation_guide#Connect_to_the_internet). For any issues you are encountering, check the friendly manual RTFM 󰱸.

Changed my mind. I will use luks -> lvm -> btrfs / swap ([link](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS))

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

Check boot mode
([link](https://wiki.archlinux.org/title/installation_guide#Verify_the_boot_mode)). If you get an output for the following, you should be good to go:

```sh
cat /sys/firmware/efi/fw_platform_size
```

Check for network connectivity:

```sh
ping archlinux.org
```

Start the sshd

```sh
systemctl start sshd
```

Set a password:

```sh
passwd
```

### Disk partitioning 2.0

In order to make things simple and secured, I will create an [LVM on LUKS](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) setup. The idea is that a full partition would be encrypted and then it can be altered as wished. Another benefit of this setup is that it allows for [suspend-to-disk](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption#With_suspend-to-disk_support) of the swap partition. This means that the device can be put to [hibernate](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation) and all the information will be saved onto the encrypted [swap](https://wiki.archlinux.org/title/swap) partition.

| Mount point | Partition                 | Partitio Type            | Size                |
| ----------- | ------------------------- | ------------------------ | ------------------- |
| /mnt/boot   | /dev/efi_system_partition | EFI system partition (1) | 1G                  |
| \[SWAP\]    | /dev/swap_partition       | Linux Swap (19)          | RAM size (TLDR)     |
| /mnt        | /dev/root_partition       | Linux                    | Remainder of device |

#### Create `boot` and `LUKS` partitions

In order to find the device we want to partition use `lsblk`. This will list all
block devices. In my virtual machine, my block device is called `vda`.

Create the following partitions using `fdisk`:

* A boot partition (size 1G)
* A LUKS partition with the rest of the memory

For more information look [here]([archwiki](https://wiki.archlinux.org/title/partitioning)).

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
of the typing.

```sh
cryptsetup open /dev/<partition> cryptlvm
````

#### Creating the [LVM](https://wiki.archlinux.org/title/LVM) volumes

Create a physical volume on top of the opened LUKS container:

```sh
pvcreate /dev/mapper/cryptlvm
```

Create a volume group (_i.e._ `vg`):

```sh
vgcreate vg /dev/mapper/cryptlvm
```

Create all logical volumes on the volume group. As mentioned before, the two
logical volumes I need are `root` and `swap`:

```sh
lvcreate -L 4G vg -n swap
lvcreate -l 100%FREE vg -n root
```

Format the logical volumes:

```sh
mkswap /dev/vg/swap
mkfs.btrfs /dev/vg/root
```

Enable swap:

```sh
swapon /dev/vg/swap
```

#### Make the `btrfs` subvolumes

I am going for the following subvolume layout, inspired by [this](https://github.com/classy-giraffe/easy-arch?tab=readme-ov-file) and [this](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout). I am planning to use [snapper](https://wiki.archlinux.org/title/Snapper) as my snapshot manager. A tip recommended by the wiki is to make a subvolume for things that you do not want to be includded in the snapshots.

| Subvolume Number | Subvolume Name | Mountpoint            |
| ---------------- | -------------- | --------------------- |
| 1                | @              | /                     |
| 2                | @home          | /home                 |
| 3                | @snapshots     | /.snapshots           |
| 4                | @var_log       | /var/log              |

In order to make the `btrfs` subvolumes, the partition has to be mounted:

```sh
mount /dev/vg/root /mnt
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

We can now mount the subvolumes

```sh
mount -o subvol=@ /dev/vg/root /mnt

# create mnounting points
mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman/pkg}

# mount the partitions
mount -o subvol=@home /dev/vg/root /mnt/home
mount -o subvol=@snapshots /dev/vg/root /mnt/.snapshots
mount -o subvol=@var_log /dev/vg/root /mnt/var/log
mount -o subvol=@var_pkgs /dev/vg/root /mnt/var/cache/pacman/pkg

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

# add yourself to sudoers file /etc/sudoers
echo "tj ALL=(ALL) ALL" >> /etc/sudoers.d/tj

# set the time
ln -sf /usr/share/zoneinfo/<Region>/<City> /etc/localtime
hwclock --systohc


# create the hostname file /etc/hostname
<yourhostname>

```

##### Set up the locale

Uncomment the required locale in `/etc/locale.gen` such as:

* en_GB.UTF-8
* en_US.UTF-8

Add the same locale in `/etc/locale.conf`

```sh
LANG=en_US.UTF-8
```

```sh
locale-gen
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
pacman -S grub grub-btrfs efibootmgr dosfstools mtools \
          networkmanager network-manager-applet os-prober sudo\
          bluez bluez-utils \
          cups \
          iw \
          nvidia nvidia-utils nvidia-settings \
          intel-ucode \
          bash-completion openssh reflector flatpak terminus-font snapper
```

Create your user

```sh
# make user
useradd -m -g users -G wheel tj
passwd tj
```

### Edit `mkinitpio` at `/etc/mkinitcpio.conf`

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

#### mkinitcpio hook (necessary for resuming from encrypted device)

## Setup Snapper ([link](https://wiki.archlinux.org/title/Snapper))

## Setup Window Manager

First, we need to get the rewquired packages. Yours might differ but here are my
choices:

* Browser: Firefox
* Terminal: Alacritty
* File Manager: lf
* Compositor: picom
* Wallppaper: nitrogen
* Display Manager: ly
* Extras:
  * Customization: lxappearance

```sh
sudo pacman -S qtile lxappearance nitrogen alacritty lf ly

```

## Troubleshoot

1. Some packages are not being downloaded:

```bash
pacman -Sy archlinux-keyring
```
