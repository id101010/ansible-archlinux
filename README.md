# Automated ArchLinux

Ansible automated installation for a fully encrypted arch linux desktop system

## System Overview
* Full Disk Encryption including /boot
* EFI
* LVM
* i3 window manager
* zsh
* rxvt-unicode

## Base system

Follow the instructions provided in [INSTALL.MD](https://github.com/id101010/ansible-archlinux/blob/master/INSTALL.md) to setup a fully encrypted base system with encrypted /boot partition.

## Install Software
First install ansible
```
$ sudo pacman -S ansible
```
then run the provided playbook

```
$ ansible-playbook --ask-become-pass playbook.yml
```
Lean back and watch the installation
