# Arch installation guide covering the following topics
* GPT partition UEFI mode installation
* Full disk encryption 
* EFI boot using GRUB
* LVM on LUKS partition scheme
* Minimal system configuration including intel-ucode or amd-ucode update

## Table of contents
1. Create bootable install medium
2. Create disk layout
3. Install base system & minimal configuration
4. Install and configure bootloader
5. Configure users

### Disk partition layout:
```
+---------------+----------------+----------------+----------------+
|ESP partition: |Boot partition: |Volume 1:       |Volume 2:       |
|               |                |                |                |
|/boot/efi      |/boot           |root            |swap            |
|               |                |                |                |
|               |                |/dev/vg0/root   |/dev/vg0/swap   |
|/dev/sda1      |/dev/sda2       +----------------+----------------+
|unencrypted    |Unencrypted     |/dev/sda3 encrypted LVM on LUKS  |
+---------------+----------------+---------------------------------+
```
## 1. Create a bootable install medium

Get the latest iso and checksums from a fast mirror. 
```bash
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/archlinux-$(date +%Y.%m.%d)-x86_64.iso archlinux.iso
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/md5sums.txt
$ wget https://mirror.puzzle.ch/archlinux/iso/latest/sha1sums.txt
```

Validate the downloads.
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
Create partitions according to the partitioning scheme above. 
Use a gpt partition table. And do not forget to set the correct partition types.
```bash
$ fdisk /dev/sda
```

The partition table should look like the following example.
```
Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         1050623   512.0 MiB   EF00  EFI System
   2         1050624         5244927   2.0 GiB     8300  Linux filesystem
   3         5244928       976773133   463.3 GiB   8E00  Linux LVM
```

Create a fat32 filesystem on the EFI partition.
```bash
$ mkfs.fat -F32 -n EFI /dev/sda1
```

Create a ext2 filesystem on the boot parition
```bash
$ mkfs.ext2 -L boot /dev/sda2
```

Create an encrypted container containing the logical volumes /root and swap. Set a safe passphrase.
The default cipher for LUKS is nowadays aes-xts-plain64, i.e. AES as cipher and XTS as mode of operation. This should be changed only under very rare circumstances. The default is a very reasonable choice security wise and by far the best choice performance wise that can deliver between 2-3 GiB/s encryption/decryption speed on CPUs with AES-NI. XTS uses two AES keys, hence possible key sizes are -s 256 and -s 512.
```bash
$ cryptsetup luksFormat --type luks2 -c aes-xts-plain64 -s 512 /dev/sda3
$ cryptsetup open /dev/sda3 cryptlvm
```

Create the logical volumes and filesystems.
```bash
$ pvcreate /dev/mapper/cryptlvm
$ vgcreate vg0 /dev/mapper/cryptlvm
$ lvcreate -L 32G vg0 -n swap # This should be at least the size of your RAM if you want hybernation to work
$ lvcreate -l 100%FREE vg0 -n root
$ mkfs.ext4 /dev/mapper/vg0-root
$ mkswap /dev/mapper/vg0-swap
```

Mount everything on the live system.
```bash
$ mount /dev/mapper/vg0-root /mnt
$ mkdir /mnt/boot
$ mount /dev/sda2 /mnt/boot
$ mkdir /mnt/boot/efi
$ mount /dev/sda1 /mnt/boot/efi
```

Activate the swap partition.
```bash
$ swapon /dev/mapper/vg0-swap
```

Check all the filesystems.
```bash
$ lsblk
```

If the output looks like this you're good to go.
```
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
loop0            7:0    0 347.9M  1 loop  /run/archiso/sfs/airootfs
sdb              8:32   1   3.8G  0 disk  
├─sdb2           8:34   1    40M  0 part  
└─sdb1           8:33   1   797M  0 part  /run/archiso/bootmnt
sda              8:0    0 931.5G  0 disk  
├─sda1           8:1    0   512M  0 part  /mnt/boot/efi
├─sda2           8:2    0   200M  0 part  /mnt/boot
└──sda3          8:3    0   800G  0 part  
  └─cryptlvm   254:1    0   800G  0 crypt 
    ├─vg0-swap 254:2    0    16G  0 lvm   [SWAP]
    └─vg0-root 254:3    0   784G  0 lvm   /mnt
```

## 3. Install base system & minimal configuration

Install the base system and some further components using pacstrap.
```bash
$ pacstrap /mnt base base-devel grub-efi-x86_64 efibootmgr lvm2 linux linux-firmware vim
```

Generate fstab with UUID representation.
```bash
$ genfstab -pU /mnt >> /mnt/etc/fstab
```

Make /tmp a ramdisk (add the following line to /mnt/etc/fstab)
```bash
tmpfs   /tmp    tmpfs   defaults,noatime,mode=1777  0 0
```

Chroot into your new base system.
```bash
$ arch-chroot /mnt /bin/bash
```

Set timezone and set your hwclock to use utc format.
```bash
$ ln -sf /usr/share/zoneinfo/Europe/Zurich /etc/localtime
$ hwclock --systohc --utc
```

Configure your locales.
```bash
$ echo "en_US.UTF-8 UTF-8" >> /etc/locale.gen
$ echo "de_CH.UTF-8 UTF-8" >> /etc/locale.gen
$ locale-gen
$ echo "LANG=en_US.UTF-8" >> /etc/locale.conf
$ echo "LC_ALL=C" >> /etc/locale.conf
```

Set a hostname, keymap and nice console font.
```bash
$ echo "myhostname" > /etc/hostname
$ echo "KEYMAP=de_CH-latin1" >> /etc/vconsole.conf
$ echo "FONT=lat9w-16" >> /etc/vconsole.conf
$ echo "FONT_MAP=8859-1_to_uni" >> /etc/vconsole.conf
```

Set a strong root password.
```bash
$ passwd
```

Change mkinitcpio.conf to support encryption. You need to change the following lines.
```bash
MODULES=(ext4)
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 filesystems fsck)
```

Regenerate the initrd image. And check for errors.
```bash
$ mkinitcpio -p linux
```

## 4. Install and configure bootloader
Change or add the following lines to your grub bootloader config.
To determine the UUID of your root device use `blkid /dev/sda3 -s UUID -o value` as stated below.
But please enter a valid UUID. And **do not** confuse `blkid` with `blkdiscard`!
```bash
GRUB_ENABLE_CRYPTODISK=y" >> /etc/default/grub
GRUB_CMDLINE_LINUX_DEFAULT="quiet"
GRUB_CMDLINE_LINUX="cryptdevice=UUID=$(blkid /dev/sda3 -s UUID -o value):vg0 root=/dev/mapper/vg0-root resume=/dev/mapper/vg0-swap"
```

I strongly recomment to install microcode updates for security reasons.
Grub will automatically recognize the image so no further configuration is necessary.
Choose accordingly.
```bash
$ pacman -S intel-ucode # for intel processors
$ pacman -S amd-ucode # for amd processors
```

Install grub to your disk, install its efi magic and write its config.
If you want grub to detect other os'es on your system, install the `os-prober` package.
But be aware that grub in efi mode will **not** detect non-uefi systems. There is no workaround as of yet.
```bash
$ grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ArchLinux
$ grub-mkconfig -o /boot/grub/grub.cfg
```

## 5. Configure users
Create a new user and set its password.
```bash
$ useradd -m -g users -G wheel $YOUR_USER_NAME
$ passwd $YOUR_USER_NAME
```

Finally uncomment the string `%wheel ALL=(ALL) ALL` in `/etc/sudoers` to allow sudo for users of the group wheel.
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
