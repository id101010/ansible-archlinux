# Automated ArchLinux

The ansible playbooks in this project automates my personal Arch Linux installation. The goal is a fully encrypted arch linux desktop system.

## System Overview
* Full Disk Encryption (including /boot if Grub/EFI are used)
* EFI / Legacy
* LVM
* i3 window manager
* zsh
* rxvt-unicode

## Install Base System

You can eighter install your own minimal system or you follow the instructions provided in the two installation guides.

* [INSTALL\_EFI](https://github.com/id101010/ansible-archlinux/blob/master/INSTALL_EFI.md) to setup a fully encrypted base system using LVM, encrypted /boot partition and EFI Support.
* [INSTALL\_LEGACY](https://github.com/id101010/ansible-archlinux/blob/master/INSTALL_LEGACY.md) to setup an encrypted base system using LVM, syslinux in legacy boot mode.

The Ansible playbook does not depend on any specific installation method. However the Legacy install is slightly easier and more beginner friendly.

## Needed Software to run Ansible

First install ansible
```
$ sudo pacman -S ansible
```
then run the provided playbook

```
$ ansible-playbook --ask-become-pass playbook.yml
```
Lean back and watch the installation.

## Testing

Assuming you've already installed vagrant you can set up a vritual machine with just these steps

``` bash
$ mkdir Vagrant && cd Vagrant
$ vagrant init archlinux/archlinux
$ vagrant up
$ vagrant ssh

$ git clone https://github.com/id101010/ansible-archlinux.git
$ ansible-playbook --ask-become-pass playbook.yml
```

Now reboot the machine and start a graphical session using virtualbox.
