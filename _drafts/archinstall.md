---
layout: post
author: tudorjnu
title: |
  Personal Arch Install Guide
description: |
  I am using this guide to make ...
---
<!-- markdownlint-disable-file MD013 -->
<!-- markdownlint-disable-file MD033 -->

> [!WARNING]
> This is still under construction. Do not follow!

My Linux journey started around October 2021. I've dual booted PopOS and ended
up using Linux for most of the time. At that point, I ended up just erasing
everything and install PopOS as my stand alone Operating System. Few months
after, and I was hooked on the tiling feature of PopOS. I decided to have a look
at the Keyboard Shortcuts and noticed the `hjkl` being used as arrows. I don't
have to leave my keyboard to reach for the arrows? How cool ... Another few
months and Windows seemed rudimentary. I did not have to go out and install
software, I had the screen automatically adjust for me (tiling), and had
shortcuts for browser, terminal and so on.

I was constantly watching other videos based on Linux and noticed that they were
referring to `hjkl` keys as Vim keys. Next thing I knew, I ended up using
Neovim and wasted ages configuring it. Luckily this was during my masters, in
pandemic times, and I had no social life.

After about a year of playing with my Desktop Environment (DE) I was quite satisfied.
But there was one itch that I had to scratch. PopOS tiling is based on `i3`, so
it seemed only natural to give that one a try. I installed `i3` from Ubuntu's
official repo's, and it lacked a lot of modern features. Keep in mind that I was
using the 22.04 version as PopOS was now developing their own Rust based DE.

A lot of effort went into configuring `i3`, including wasting around 2 weeks
configuring the bar (`polybar`). A lot of things were not working due to the
outdated packages in the repositories, so it never actually felt right. However,
I managed to fix a few things that were frustrating me such as logging out, and
screen tearing. Also utilities like screenshots were relatively sorted. Things
were working ...

While I was pretty happy with the things, I decided I want something a little
bit more dynamic. So here comes `Qtile`. Qtile worked like a charm for me, and my
`python` familiarity was paying dividends. A config I can finally understand and
thinker with. I also started to explore different programs: `lf`, `ly` and the
likes. What I got from here was simple, the programs that I want to use are not
in Ubuntu's repositories ... So ... I decided to install arch and make my life
"_easier_". Also, it seemed like for everything I wanted to do, PopOS set it up
for me in the way they thought I wanted it done. Colors, xrandr and so on were
all set by System76 devs. It is the same with all the DE's, they make
"_sensible_" choices, and then it takes you hours to find out that your
"realsense" camera is not working because of `brltty`, a utility for blind
people. Very mindful of Ubuntu's devs, but for me? I don't need that stuff, and it
was getting in the way, causing a lot of debugging frustrations.

And this is how I ended up wanting to install Arch. I needed something that
allowed me to do things from a blank canvas. And if I f****d up, it was me, and I
could trace it back.

So here we go, this is my guide on installing arch. It is not really meant for
"you", the reader. It started as just a way for me to document my process of
installing it, while in a virtual machine (VM), so I can repeat the process with
ease on actual hardware. Also, archwiki is just amazing, but the information is
pretty scattered. So if you want to do anything, you have to read a few guides
and put the pieces together. So this is what I have done, I put a few bits and
pieces together from [archwiki](https://wiki.archlinux.org/) in a way I would
understand it whenever I wanted to install Arch. Anyone could do this by just
reading the f*****g manual (RTFM). I would also follow the [installation_guide](https://wiki.archlinux.org/title/installation_guide) closely. Be aware, if you break your PC, is on you! Now, let's start the installation.

## Pre-installation

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
loadkeys <chosen key> # loadkeys uk for example
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
connected or connect via wireless by using [iwctl](https://wiki.archlinux.org/title/Iwd#iwctl):

1. Enter in the interactive prompt: `iwctl`
2. Show the devices such as `wlan0`: `device list`
3. Search for networks: `station wlan0 get-networks`
4. Exit the prompt: `<CTRL+d>`
5. Connect: `iwclt --passpharese "<your_passphrase>" station <your_station>
   connect <network_name>`

Check for an ip address:

```sh
ip addr
```

Start the `sshd` if you want to connect from another pc. It makes the process a
lot easier:

```sh
systemctl start sshd
```

Set a password for the root (needed for the ssh):

```sh
passwd
```

### Disk partitioning

[Partitioning Scheme](assets/img/blog/archlinux-install/partitioning.png)

To make things simple and secured, I will create an [LVM on LUKS](https://wiki.archlinux.org/title/dm-crypt/Encrypting_an_entire_system#LVM_on_LUKS) setup. The idea is that a full partition would be encrypted, and then it can be altered as wished. Another benefit of this setup is that it allows for [suspend-to-disk](https://wiki.archlinux.org/title/Dm-crypt/Swap_encryption#With_suspend-to-disk_support) of the swap partition. This means that the device can be put to [hibernate](https://wiki.archlinux.org/title/Power_management/Suspend_and_hibernate#Hibernation) and all the information will be saved onto the encrypted [swap](https://wiki.archlinux.org/title/swap) partition. The overall partitioning can be viewed above, where the summary of invormation can be viewed bellow:

<table border="1">
  <tr>
    <th>Partition</th>
    <th>Partition Type</th>
    <th>Partition Size</th>
    <th>Encryption</th>
    <th>LVM Physical Volume</th>
    <th>Volume Group</th>
    <th>Logical Volume</th>
    <th>Logical Volume Size</th>
    <th>Filesystem</th>
    <th>Subvolume Number</th>
    <th>Subvolume Name</th>
    <th>Mountpoint</th>
  </tr>
  <tr>
    <td>/dev/sda1</td>
    <td>UEFI</td>
    <td>1G</td>
    <td>No</td>
    <td>No</td>
    <td>N/A</td>
    <td>N/A</td>
    <td>N/A</td>
    <td>N/A</td>
    <td>N/A</td>
    <td>N/A</td>
    <td>/boot</td>
  </tr>
  <tr>
    <td rowspan="6" style="text-align: center; vertical-align: middle">
      /dev/sda2
    </td>
    <td rowspan="6" style="text-align: center; vertical-align: middle">LUKS</td>
    <td rowspan="6" style="text-align: center; vertical-align: middle">
      Rest of disk
    </td>
    <td rowspan="6" style="text-align: center; vertical-align: middle">Yes</td>
    <td rowspan="6" style="text-align: center; vertical-align: middle">Yes</td>
    <td rowspan="6" style="text-align: center; vertical-align: middle">vg0</td>
    <td>lvmswap</td>
    <td>4G</td>
    <td>swap</td>
    <td>N/A</td>
    <td>N/A</td>
    <td>N/A</td>
  </tr>
  <tr>
    <td rowspan="5" style="text-align: center; vertical-align: middle">
      lvmroot
    </td>
    <td rowspan="5" style="text-align: center; vertical-align: middle">
      Rest of the space
    </td>
    <td rowspan="5" style="text-align: center; vertical-align: middle">
      btrfs
    </td>
    <td>1</td>
    <td>@</td>
    <td>/</td>
  </tr>
  <tr>
    <td>2</td>
    <td>@home</td>
    <td>/home</td>
  </tr>
  <tr>
    <td>3</td>
    <td>@snapshots</td>
    <td>/.snapshots</td>
  </tr>
  <tr>
    <td>4</td>
    <td>@var_log</td>
    <td>/var/log</td>
  </tr>
  <tr>
    <td>5</td>
    <td>@var_pkgs</td>
    <td>/var/cache/pacman/pkg</td>
  </tr>
</table>
#### Create `boot` and `LUKS` partitions

To find the device we want to partition use `lsblk`. This will list all
block devices. In my virtual machine, my block device is called `vda`.

Create the following partitions using `fdisk`:

* A boot partition (size 1G)
* A LUKS partition with the rest of the memory

For more information have a look at the [archwiki](https://wiki.archlinux.org/title/partitioning).

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

#### Make the [BTRFS](https://wiki.archlinux.org/title/btrfs) subvolumes

I am going for the following subvolumes layout, inspired by [this](https://github.com/classy-giraffe/easy-arch?tab=readme-ov-file) and [this](https://wiki.archlinux.org/title/Snapper#Suggested_filesystem_layout). I am planning to use [snapper](https://wiki.archlinux.org/title/Snapper) as my snapshot manager. A tip recommended by the wiki is to make a subvolume for things that you do not want to be includded in the snapshots.

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
mount -o subvol=@ /dev/vg0/lvmroot /mnt
```

Create the directories:

```sh

mkdir -p /mnt/{boot,home,.snapshots,var/log,var/cache/pacman/pkg}
```

And mount the subvolumes:

```sh
mount -o subvol=@home /dev/vg0/lvmroot /mnt/home
mount -o subvol=@snapshots /dev/vg0/lvmroot /mnt/.snapshots
mount -o subvol=@var_log /dev/vg0/lvmroot /mnt/var/log
mount -o subvol=@var_pkgs /dev/vg0/lvmroot /mnt/var/cache/pacman/pkg
```

Finally, we can mount the `boot` partition:

```sh
mount /dev/vda1 /mnt/boot
```

### Installation

#### Install essential packages

The basic installation is just `pacstrap -K /mnt base linux linux-firmware`. However,
there are a few things that are needed to be installed and are recommended as in
the [wiki](https://wiki.archlinux.org/title/installation_guide). The main things
we need are:

* CPU [microcode](https://wiki.archlinux.org/title/Microcode) updates: `amd-ucode` or `intel-ucode`;
* [utilities for filesystems](https://wiki.archlinux.org/title/File_systems):
`btrfs-progs`;
* LVM management: `lvm2`;
* firmware not included in `linux-firmware` (none on my end);
* [networking](https://wiki.archlinux.org/title/Network_configuration) software: `networkmanager`, `networkmanager-applet`, and `iwd`;
* [console text editor](https://wiki.archlinux.org/title/List_of_applications/Documents#Console): `neovim` (for me, for you might be `nano`);
* documentation accessing software: `man-db`, `man-pages`,`texinfo`,and `tldr`;

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

Now there are a few packages left. Feel free to adapt this to your case:

* Boot loader: `grub`, `grub-btrfs`;
* Filesystem tools: `efibootmgr` `dosfstools` `mtools` `os-prober`, and `sudo`;
* Display Drivers: `nvidia`, `nvidia-lts`, `nvidia-open` (if Touring+), `nvidia-utils`, `nvidia-settings`;
* Utilities: `bash-compretion`, `openssh`, `reflector`, `flatpak`, `lxappearance`;
* Snapshots Manager: `snapper`;
* Software packaging: `flatpak`;

```sh
sudo pacman -S \
  grub grub-btrfs \
  efibootmgr dosfstools mtools os-prober sudo \
  nvidia nvidia-lts nvidia-open nvidia-utils nvidia-settings \
  bash-completion openssh reflector flatpak lxappearance \
  snapper \
  flatpak \
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

I added encrypt before filesystem in hooks and `btrfs` in modules, see [here](https://wiki.archlinux.org/title/Dm-crypt/System_configuration#Unlocking_in_early_userspace) and [here](https://wiki.archlinux.org/title/Mkinitcpio#Common_hooks).

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

After a restart you should be greeted by [GDM](https://wiki.archlinux.org/title/GDM).

### The window manager route

<!-- TODO: fill up this section -->

If you are like me, however, and like to have a window manager, we got a bit
more work to do. First, we need to get the required packages. Yours might differ but here are my
choices:

* Compositor: `picom`
* Display Manager: `ly`
* Window Manager: `qtile`
* Wallpaper: `nitrogen`
* Bluetooth: `bluez`, `bluez-utils`;
* Sound System: `pipewire`;
* Printing: `cups`;
* Snapshots Manager: `snapper`;
* Software:
  * Browser: `firefox`
  * Terminal: `alacritty`
  * File Manager: `lf`
* Fonts: `terminus-font`, `ttf-ubuntu-nerd`, `ttf-ubuntu-mono-nerd`, `ttf-jetbrains-mono-nerd`;
* Display server: `xorg`, `xorg-xwayland`

```sh
sudo pacman -S picom ly qtile nitrogen
sudo pacman -S \
  picom \
  ly \
  qtile \
  bluez bluez-utils \
  pipewire \
  cups \
  snapper \
  firefox \
  alacritty \
  lf \
  xorg xorg-xwayland \
  terminus-font ttf-ubuntu-nerd ttf-ubuntu-mono-nerd ttf-jetbrains-mono-nerd
```

That should do it for now. You should now have a fully functional system

