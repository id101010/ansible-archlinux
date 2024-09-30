# Arch installation guide covering the following topics
* GPT partition and UEFI mode installation
* Full disk encryption
* LVM on LUKS partition scheme
* Minimal system configuration including intel-ucode or amd-ucode update

## Table of contents
1. Create bootable install medium
2. Create disk layout
3. Install base system
4. Install bootloader
5. Configure users

### Disk partition layout:
```
+----------------+-----------------+-----------------+
| EFI partition: | Volume 1:       | Volume 2:       |
|                |                 |                 |
| /boot/efi      | swap            | /               |
|                |                 |                 |
|                | /dev/vg0/swap   | /dev/vg0/root   |
| /dev/sda1      +-----------------+-----------------+
| unencrypted    | /dev/sda2 encrypted LVM on LUKS   |
+----------------+-----------------------------------+
```
## 1. Create bootable install medium

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

Enable network time synchronization and check if the time got synchronized.
```bash
$ timedatectl set-ntp true
$ timedatectl status
```

Change keyboard layout and increase font size if needed.
```bash
$ setfont sun12x22
$ loadkeys de_CH-latin1
```

Check if your system is running in uefi mode.
```bash
$ ls /sys/firmware/efi/efivars
$ efibootmgr
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
   2         1050624       976773133   465.3 GiB   8E00  Linux LVM
```

Create an encrypted container containing the logical volumes /root and swap. Set a safe passphrase.
The default cipher for LUKS is nowadays aes-xts-plain64, i.e. AES as cipher and XTS as mode of operation.
This should be changed only under very rare circumstances.
The default is a very reasonable choice security wise and by far the best choice performance wise that can deliver between 2-3 GiB/s encryption/decryption speed on CPUs with AES-NI. XTS uses two AES keys, hence possible key sizes are -s 256 and -s 512.
```bash
$ cryptsetup luksFormat --type luks2 -c aes-xts-plain64 -s 512 /dev/sda2
$ cryptsetup open /dev/sda2 cryptlvm
```

Create a physical volume and a volume group inside the luks container.
```bash
$ pvcreate /dev/mapper/cryptlvm
$ vgcreate vg0 /dev/mapper/cryptlvm
```

Create the logical volumes.
```bash
$ lvcreate -L 32G vg0 -n swap # This should be at least the size of your RAM if you want hybernation to work
$ lvcreate -l 100%FREE vg0 -n root
```

Create the filesystems.
```bash
$ mkfs.fat -F32 -n EFI /dev/sda1
$ mkfs.ext4 -L root /dev/mapper/vg0-root
$ mkswap -L swap /dev/mapper/vg0-swap
```

Mount everything on the live system.
```bash
$ mount /dev/mapper/vg0-root /mnt
$ mount --mkdir /dev/sda1 /mnt/boot/efi
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
```bash
NAME           MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda              8:0    0 931.5G  0 disk
├─sda1           8:1    0   512M  0 part  /mnt/boot/efi
└─sda2           8:3    0   800G  0 part
  └─cryptlvm   254:1    0   800G  0 crypt
    ├─vg0-swap 254:2    0    32G  0 lvm   [SWAP]
    └─vg0-root 254:3    0   784G  0 lvm   /mnt
```

## 3. Install base system

Install the base system and some further components using pacstrap.
```bash
$ pacstrap /mnt base base-devel grub efibootmgr lvm2 linux linux-firmware vim
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
$ locale > /etc/locale.conf
```

Set a hostname, keymap and nice console font.
```bash
$ echo "myhostname" > /etc/hostname
$ echo "127.0.1.1 myhostname.localdomain myhostname" >> /etc/hosts
$ echo "KEYMAP=de_CH-latin1" >> /etc/vconsole.conf
$ echo "FONT=lat9w-16" >> /etc/vconsole.conf
$ echo "FONT_MAP=8859-1_to_uni" >> /etc/vconsole.conf
```

Set a strong root password.
```bash
$ passwd
```

Change mkinitcpio.conf to support encryption. You need to change the following line.
```bash
HOOKS=(base udev autodetect keyboard keymap consolefont modconf block encrypt lvm2 resume filesystems fsck)
```

Regenerate the initrd image. And check for errors.
```bash
$ mkinitcpio -p linux
```

## 4. Install bootloader

Install GRUB to your EFI Partition
```bash
$ grub-install --target=x86_64-efi --efi-directory=/boot/efi
```

Change or add the following lines to your grub config.
To determine the UUID of your crypto partition use `blkid /dev/sda2 -s UUID -o value`.

```bash
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="Arch Linux"
GRUB_ENABLE_CRYPTODISK=y
GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet"
GRUB_CMDLINE_LINUX="rd.lvm.vg=vg0 rd.luks.uuid=UUID_OF_CRYPT_PARTITION resume=/dev/mapper/vg0-swap"
```

I strongly recommend to install microcode updates for security reasons.
Grub will automatically recognize the image so no further configuration is necessary.
```bash
$ pacman -S intel-ucode # for intel processors
or
$ pacman -S amd-ucode # for amd processors
```

Generate grub config.
```bash
$ grub-mkconfig -o /boot/grub/grub.cfg
```

Set a strong root password.
```bash
$ passwd
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
