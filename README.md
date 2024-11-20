# Arch Linux Installation
with UEFI, full disk encryption using LUKS and LVM2 for volume management.

---

## Prepare your bootable USB drive
[Download](https://archlinux.org/download) Arch Linux and `dd` it onto your USB drive  
_# replace /dev/sdX with your USB Flash Drive letter (lsblk; fdisk -l)_

```sh
sudo dd bs=4M if=/archlinux-2024.04.01-x86_64.iso of=/dev/sdX && sync
```

---

# Installation

Boot from your USB drive, check if you have network connectivity and sync your system clock
```sh
ping -c 3 google.com
```

```sh
timedatectl set-ntp true
```

## Disk partitioning

Confirm your storage layout and note the drive you will be installing to
```sh
fdisk -l # or
lsblk
# /dev/nvme0n1
```

Partition the drive to have a 1GB partition for boot and then the rest do Linux Filesystem (as we will be encrypting it)
```sh
cfdisk /dev/nvme0n1

# Select Label Type: GPT
# 1G Partition /boot TYPE=EFI
# 100%FREE /root TYPE=LINUX_86_64
```

You should now have two `/dev/nvme0n1p1` & `/dev/nvme0n1p2` partitions

Format the first `1G` partition to `VFAT32`
```sh
mkfs.vfat -F32 /dev/nvme0n1p1
```
And now we can encrypt and setup the our partition

## Disk Encryption

Encrypt the full partition
```sh
cryptsetup luksFormat /dev/nvme0n1p2
```

After that succeeds we can open that encrypted partition to work with it
```sh
cryptsetup luksOpen /dev/nvme0n1p2 cryptroot
```
_# You can change "cryptroot" to whatever you like, but you will have to remember to use your name instead of cryptroot for the rest of the install_


## LVM Creation

A logical volume needs a volume group which in turn needs a physical volume. So lets set those up

### Create your physical volume
```sh
pvcreate /dev/mapper/cryptroot
```
### Create a volume group (I will call it "vg0")
```sh
vgcreate vg0 /dev/mapper/cryptroot
```

### Create the logical volumes (root, home, swap)  
_Notice -L and -l, one is for fixed size, the other is percentage_
```sh
lvcreate -L 32G vg0 -n swap # If you plan to use hybernation - set the same size as your RAM
lvcreate -L 120G vg0 -n root # Modify "120G" to what ever size you think fits your root setup
lvcreate -l 100%FREE vg0 -n home # Fill the rest of the volume group for home
lvreduce -L -256M vg0/home # Reduce home by 256Mb to allow e2scrub to run as per Arch wiki
```

### Format and mount the newly created volumes
```sh
mkfs.ext4 /dev/mapper/vg0-root
mkfs.ext4 /dev/mapper/vg0-home
mkswap /dev/mapper/vg0-swap

mount /dev/mapper/vg0-root /mnt
mount --mkdir /dev/mapper/vg0-home /mnt/home
mount --mkdir /dev/nvme0n1p1 /mnt/boot

swapon -s /dev/mapper/vg0-swap
```

## Install the system

Installs linux kernel, base dependencies and text editor
```sh
pacstrap -i /mnt base base-devel linux linux-firmware lvm2 nano efibootmgr networkmanager zsh git curl openssh sysstat intel-ucode wget curl
```
I usually install my other required packages now rather than after chroot'ing into the system  
_# If you're on AMD replace "intel-ucode" with "amd-ucode"_

## Generate fstab

```sh
genfstab -U /mnt >> /mnt/etc/fstab
```

Check if swap was written also
```sh
cat /mnt/etc/fstab
```
And if not find your `vg0-swap` UUID with `blkid /dev/mapper/vg0-swap` and add it at the end of the fstab
```cfg
UUID=<SWAP_UUID> none swap defaults 0 0
```

##  Chroot into your new system
```sh
arch-chroot /mnt
```

### Setup the bootloader (`systemd-boot`)

```sh
bootctl --path=/boot install
```

Get the partition UUID which the bootloader will need to load (it should be the partition you encrypted and not the actual LVM). Write it to a file to make writing the bootloader entry easier.
```
blkid /dev/nvme0n1p2 > /boot/loader/entries/arch.conf
```

Edit the entry file and add the required info
```sh
nano /boot/loader/entries/arch.conf
```
_# replace intel-ucode with amd-ucode if using AMD processor_  
_# replace <PARTITION_ID> with the UUID that we entered here with blkid in the previous step_

```conf
title Arch Linux
linux /vmlinuz-linux
initrd /intel-ucode.img
initrd /initramfs-linux.img
options cryptdevice=UUID=<PARTITION_ID>:vg0 root=/dev/mapper/vg0-root quiet splash rw
```

Save and exit with `CTRL+x` and update bootloader
```sh
bootctl update
```

### Add modules to mkinitcpio
```sh
nano /etc/mkinitcpio.conf
```
Update `HOOKS` to have `encrypt lvm2` between `keymap filesystems`
```conf
HOOKS=(base udev autodetect microcode modconf kms keyboard keymap consolefont block encrypt lvm2 filesystems fsck)
```
If you have an NVME drive like in this tutorial add it to the modules section
```conf
MODULES=(nvme)
```

Save the file with `CTRL+x` and update initramfs
```sh
mkinitcpio -p linux
```

### Miscellaneous configuration 

Enable NetworkManager
```sh
systemctl enable NetworkManager
```

Change your region and localtime, sync clock
```sh
ln -sf /usr/share/zoneinfo/<REGION>/<CITY> /etc/localtime
hwclock --systohc
```

Edit `/etc/locale.gen` and uncomment `en_GB.UTF-8 UTF-8` and other needed locales. Generate the locales by running:
```sh
locale-gen
```
Make language settings persistent by adding them to `/etc/locale.conf`
```sh
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
```

Setup your keyboard mapping and terminal font, and make them persistent by adding them to `/etc/vconsole.conf`
```sh
pacman -Sy terminus-font
echo KEYMAP=uk\\nFONT=ter-v14n\\n > /etc/vconsole.conf
```

Setup your hostname
```bash
echo <HOSTNAME> > /etc/hostname
```

Configure local name resolution
```sh
echo 127.0.0.1 <HOSTNAME>\\n::1 <HOSTNAME\\n127.0.1.1 <HOSTNAME>.<DOMAIN> <HOSTNAME>\\n > /etc/hosts
```

Setup root password:
```bash
passwd
```

Exit chroot, unmount partitions and reboot
```bash
exit #(ctrl+d)
umount -R /mnt
reboot
```

## After Booting up

Just in case, update everything:
```bash
pacman -Syy
pacman -Syu
```

Create another user (DO NOT USE ROOT FOR DAILY USE!)
```bash
useradd -m -g users -G wheel -s /bin/zsh <USERNAME>
passwd <USERNAME>
```

Add user to SUDOERS
```bash
visudo
```
Find where it says `#wheel ALL=(ALL) ALL` and remove the `#`.

## Setup SSH

```bash
# Whenever changing the configuration, use sshd in test mode before restarting the service to ensure it will be able to start cleanly. Valid configurations produce no output.
# use: sshd -t
```
Setup SSH Welcome Banner:
```bash
sudo vim /etc/ssh/sshd_config
# Uncomment # Banner /etc/issue
# :wq
```
```bash
sudo vim /etc/issue
# Add a welcome message
# :wq
```
```bash
sudo systemctl start sshd
sudo systemctl enable sshd
```
