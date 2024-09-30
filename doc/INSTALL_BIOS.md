# Arch installation guide covering the following topics
* MBR partition and BIOS Mode installation
* Full disk encryption
* LVM on LUKS partitioning scheme
* Minimal system configuration including intel-ucode updates

## Table of contents
1. Create bootable install medium
2. Create disk layout
3. Install base system
4. Install bootloader
5. Configure users

### Disk partition layout:
```
+----------------+------------------------+------------------------+
| Boot partition | Logical volume 1       | Logical volume 2       |
|                |                        |                        |
| /boot          | [SWAP]                 | /                      |
|                |                        |                        |
|                | /dev/mapper/vg0-swap   | /dev/mapper/vg0-root   |
|                +------------------------+------------------------+
| unencrypted    |             LUKS encrypted partition            |
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
Create partitions according to the partitioning scheme above. Use a mbr parition table.
```bash
$ fdisk /dev/sda
```

Create a partition table that looks like the following example. Don't forget to set the boot flag.
```
Disk /dev/sda: 477 GiB, 512110190592 bytes, 1000215216 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x971ef2ea

Device     Boot   Start        End   Sectors  Size Id Type
/dev/sda1  *       2048    2099199   2097152    1G 83 Linux
/dev/sda2       2099200 1000215215 998116016  476G 83 Linux LVM
```

Now create a filesystem on the /boot partition.
```bash
mkfs.ext4 -L boot /dev/sda1
```

Create an encrypted container containing the logical volumes /root and swap. Make sure to use a safe passphrase.
```bash
$ cryptsetup luksFormat --type luks2 -c aes-xts-plain64 -s 512 /dev/sda2
$ cryptsetup open /dev/sda2 cryptlvm
$ pvcreate /dev/mapper/cryptlvm
$ vgcreate vg0 /dev/mapper/cryptlvm
$ lvcreate -L 16G vg0 -n swap # This should be at least the size of your RAM if you want hybernation to work
$ lvcreate -l 100%FREE vg0 -n root
$ mkfs.ext4 -L root /dev/mapper/vg0-root
$ mkswap -L swap /dev/mapper/vg0-swap
```

Mount everything on the live system.
```bash
$ mount /dev/mapper/vg0-root /mnt
$ mount --mkdir /dev/sda1 /mnt/boot
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

## 3. Install base system

Install the base system, bootloader and some additional components using
pacstrap.
```bash
$ pacstrap -K /mnt base base-devel grub linux linux-firmware lvm2 vim
```

Generate fstab using UUIDs as representation.
```bash
$ genfstab -pU /mnt >> /mnt/etc/fstab
```

Chroot into your new base system.
```bash
$ arch-chroot /mnt
```

Set timezone, and hostname and set your hwclock to utc.
```bash
$ ln -sf /usr/share/zoneinfo/Europe/Zurich /etc/localtime
$ hwclock --systohc
```

Configure your locales. Omit the swiss german line if you don't need it.
```bash
$ echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
$ echo "de_CH.UTF-8 UTF-8" >> /etc/locale.gen
$ locale-gen
$ locale > /etc/locale.conf
```

Set a hostname, keymap and nice console font.
```bash
echo "myhostname" >> /etc/hostname
echo "127.0.1.1   myhostname.localdomain  myhostname" >> /etc/hosts
echo "KEYMAP=de_CH-latin1" >> /etc/vconsole.conf # Change to your locale
echo "FONT=lat9w-16" >> /etc/vconsole.conf
echo "FONT_MAP=8859-1_to_uni" >> /etc/vconsole.conf
```

Change mkinitcpio.conf to support lvm2 and encryption.
Edit `/etc/mkinitcpio.conf` and make sure the hook line looks like this:
```bash
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 resume filesystems fsck)
```

Regenerate the initrd image.
```bash
$ mkinitcpio -p linux
```

## 4. Install bootloader

Install microcode updates. These updates provide bug fixes that can be critical to the stability of your system. 
```bash
$ pacman -S intel-ucode
or
$ pacman -S amd-ucode
```

Edit the following lines in the `/etc/default/grub` config and generate the config file. The UUID of crypt partition can be retrieved using `blkid /dev/sda2 -s UUID -o value`.
```bash
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Arch Linux"
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
GRUB_CMDLINE_LINUX="rd.lvm.vg=vg0 rd.luks.uuid=UUID_OF_CRYPT_PARTITION resume=/dev/mapper/vg0-swap"
```

Install the bootloader. (where /dev/sda is a DISK not a PARTITION)
```bash
$ grub-install --target=i386-pc /dev/sda
```

Generate grub config.
```bash
$ grub-mkconfig -o /boot/grub/grub.cfg
```

## 5. Configure users

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
