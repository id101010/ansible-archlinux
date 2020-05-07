# Arch installation guide covering the following topics 
* MBR partition BIOS Mode installation
* Full disk encryption using dm-crypt/LUKS
* LVM on LUKS
* Lightweight Sysliux Bootloader
* Minimal system configuration including intel-ucode updates

## Table of contents
1. Create bootable install medium
2. Create disk layout
3. Install base system and bootloader
4. Chroot into the System and do a minimal configuration

### Disk partition layout:
```
+----------------+-------------------------------------------------+
| Boot partition | Logical volume 1       | Logical volume 2       |
|                |                        |                        |
| /boot          | [SWAP]                 | /                      |
|                |                        |                        |
|                | /dev/mapper/vg0-swap   | /dev/mapper/vg0-root   |
| (may be on     |_ _ _ _ _ __ _ _ _ _ _ _|__ _ _ _ _ _ _ _ _ _ _ _|
| other device)  |                                                 |
|                |             LUKS encrypted partition            |
| /dev/sda1      |                    /dev/sda2                    |
+----------------+-------------------------------------------------+
```
## 1. Create a bootable install medium

Get the latest iso and checksums from a mirror near you. The recommended mirror
below is maintained by me and located in a datacenter based in switzerland.
Since new isos are not built on a daily basis, you may need to choose the
newest iso yourself. 

```bash
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/archlinux-$(date +%Y.%m.%d)-x86_64.iso archlinux.iso
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/md5sums.txt
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/sha1sums.txt
```

Check if the download is valid.
```bash
$ md5sum --check md5sums.txt
$ sha1sum --check sha1sums.txt
```

Create a bootable usb flash drive, make sure /dev/sdX corresponds to the usb drive.
```bash
$ dd if=archlinux.iso of=/dev/sdX bs=1M status=progress && sync
```

Boot and check your internet connection, fix if necessary.
```bash
$ ping google.com
```

Enable network time synchronization.
```bash
$ timedatectl set-ntp true
```

Check if the time got synchronized.
```bash
$ timedatectl status
```

## 2. Create disk layout
Create partitions according to the partitioning scheme above. Use a mbr
parition table.
```bash
$ fdisk /dev/sda
```

Create a partition table that looks like the following example
```
Disk /dev/sda: 477 GiB, 512110190592 bytes, 1000215216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x971ef2ea

Device     Boot   Start        End   Sectors  Size Id Type
/dev/sda1  *       2048    2099199   2097152    1G 83 Linux
/dev/sda2       2099200 1000215215 998116016  476G 83 Linux
```

Now create a filesystem on the /boot partition. Syslinux needs the 64bit option
of ext4 to be disabled since it can only handle 32bit block sizes. You don't want a 16TiB boot partition anyway. Make sure to set the option right otherwise your
bootloader won't load.
```bash
mkfs.ext4 -L boot -O '^64bit' /dev/sda1
```

Create an encrypted container containing the logical volumes /root and swap. Make sure to use a safe passphrase.
```bash
$ cryptsetup luksFormat --type luks2 -c aes-xts-plain64 -s 512 /dev/sda2
$ cryptsetup open /dev/sda2 cryptlvm
$ pvcreate /dev/mapper/cryptlvm
$ vgcreate vg0 /dev/mapper/cryptlvm
$ lvcreate -L 16G vg0 -n swap # This should be at least the size of your RAM if you want hybernation to work
$ lvcreate -l 100%FREE vg0 -n root
$ mkfs.ext4 /dev/mapper/vg0-root
$ mkswap /dev/mapper/vg0-swap
```

Mount everything on the live system.
```bash
$ mkdir /mnt/boot
$ mount /dev/mapper/vg0-root /mnt
$ mount /dev/sda1 /mnt/boot
```

Activate the swap partition.
```bash
$ swapon /dev/mapper/vg0-swap
```

Check all the filesystem.
```bash
$ lsblk
```

If the output looks like this you're good to go.
```
NAME            MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda               8:0    0  477G  0 disk
├─sda1            8:1    0    1G  0 part  /mnt/boot
└─sda2            8:2    0  476G  0 part
  └─main        254:0    0  476G  0 crypt
    ├─vg0-swap  254:1    0   16G  0 lvm   [SWAP]
    └─vg0-root  254:2    0  440G  0 lvm   /mnt
```

## 3. Install base system and bootloader

I strongly recommend to select a fast mirror for the base installation. This
will greatly improve the download speed. Eighter uncomment the server in the
provided mirrorlist or use the following suggestion.

```bash
$ rm /etc/pacman.d/mirrorlist
$ echo 'Server = http://mirror.puzzle.ch/archlinux/$repo/os/$arch' >
/etc/pacman.d/mirrorlist
```

Install the base system, bootloader and some additional components using
pacstrap.
```bash
$ pacstrap /mnt base base-devel syslinux linux linux-firmware vim git
```

Install the syslinux bootloader.
```bash
$ syslinux-install_update -i -a -m -c /mnt
```

Edit the `/mnt/boot/syslinux/syslinux.cfg` bootloader configuration to support your cryptlvm. 
To do this you need to change the `APPEND` lines for the Arch and Archfallback targets. 
To make sure your system has the right keyboard layout when entering the LUKS key, append a location and language entry to the
kernel line. The example below uses the Swiss QWERTY layout. If you use an english QWERTZ layout you can omit these entries.
The Resume statement is used for hibernation. If you don't want this you can omit it as well.

```bash
...

LABEL arch
    MENU LABEL Arch Linux
    LINUX ../vmlinuz-linux
    APPEND cryptdevice=/dev/sda2:vg0 root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap rw lang=en locale=de_CH.UTF-8 quiet splash
    INITRD ../initramfs-linux.img

LABEL archfallback
    MENU LABEL Arch Linux Fallback
    LINUX ../vmlinuz-linux
    APPEND cryptdevice=/dev/sda2:vg0 root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap rw lang=en locale=de_CH.UTF-8 quiet splash
    INITRD ../initramfs-linux-fallback.img

...
```

Generate fstab using UUIDs as representation.
```bash
$ genfstab -pU /mnt >> /mnt/etc/fstab
```

## 4. Chroot into the System and do a minimal configuration

Chroot into your new base system.
```bash
$ arch-chroot /mnt
```

Set timezone, and hostname and set your hwclock to utc.
```bash
$ ln -sf /usr/share/zoneinfo/Europe/Zurich /etc/localtime
$ hwclock --systohc --utc
```

Configure your locales. Omit the swiss german line if you don't need it.
```bash
$ echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
$ echo "de_CH.UTF-8 UTF-8" >> /etc/locale.gen
$ locale-gen
$ echo "LANG=en_US.UTF-8" >> /etc/locale.conf
```

Set a hostname, keymap and nice console font.
```bash
echo "myhostname" >> /etc/hostname
echo "KEYMAP=de_CH-latin1" >> /etc/vconsole.conf # Change to your locale
echo "FONT=lat9w-16" >> /etc/vconsole.conf
echo "FONT_MAP=8859-1_to_uni" >> /etc/vconsole.conf
```

Change mkinitcpio.conf to support ext4, lvm2 and encryption.
You need to add the following:
* MODULES: ext4
* HOOKS: encrypt lvm2 resume

Eighter edit `/etc/mkinitcpio.conf` by hand or use the following sed commands.
```bash
$ sed -i "s/MODULES=.*/MODULES=(ext4)/g" /etc/mkinitcpio.conf
$ sed -i "s/HOOKS=.*/HOOKS=(base udev autodetect modconf keyboard block keymap encrypt lvm2 resume filesystems keyboard fsck shutdown)/g" /etc/mkinitcpio.conf
```

Regenerate the initrd image.
```bash
$ mkinitcpio -p linux
```
Install microcode updates. These updates provide bug fixes that can be critical to the stability of your system. You need to install the package first and then create a second initrd entry in the bootloader config.
```bash
$ pacman -S intel-ucode
or
$ pacman -S amd-ucode
``` 

Edit the `/boot/syslinux/syslinux.cfg` config file. There must be no spaces between the intel-ucode and initramfs-linux initrd files.
The period signs also do not signify any shorthand or missing code. The INITRD line must be exactly as illustrated below.
```bash
LABEL arch
    MENU LABEL Arch Linux
    LINUX ../vmlinuz-linux
    INITRD ../{intel||amd}-ucode.img,../initramfs-linux.img # Make sure to choose the right image!
    APPEND <your kernel parameters>
```

Set a strong root password.
```bash
$ passwd
```

Create a new user and set its password.
```bash
$ useradd -m -g users -G wheel $YOUR_USER_NAME
$ passwd $YOUR_USER_NAME
```

Uncomment string `%wheel ALL=(ALL) ALL` in `/etc/sudoers` to allow sudo for users of the group wheel.
```bash
$ vim /etc/sudoers
```

Exit from chroot, unmount system, shutdown, extract flash stick. You made it! Now you have fully encrypted system.
```bash
$ exit
$ umount -R /mnt
$ swapoff -a
$ reboot
```

Reboot into your new arch linux base system and begin installing software.
