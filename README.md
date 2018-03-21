# Automated ArchLinux

This ansible playbook automates my personal Arch Linux installation. The goal is a fully encrypted desktop system.

## System Overview
* Full Disk Encryption (including /boot if Grub/EFI is used)
* LVM on LUKS
* Plymouth
* i3 window manager
* zsh with oh-my-zsh theme
* rxvt-unicode true color terminal

## Install Base System

You can eighter install your own minimal system or you follow the instructions provided in the two installation guides.

* [INSTALL\_LEGACY](https://github.com/id101010/ansible-archlinux/blob/master/doc/INSTALL_LEGACY.md) to setup an encrypted base system using LVM, syslinux in legacy boot mode.
* [INSTALL\_EFI](https://github.com/id101010/ansible-archlinux/blob/master/doc/INSTALL_EFI.md) to setup a fully encrypted base system using LVM, encrypted /boot partition and EFI Support.

The Ansible playbook does not depend on any specific installation method. However the Legacy install is slightly easier and more beginner friendly.

## Needed Software to run Ansible

First install ansible
```
$ sudo pacman -S ansible
```
then download and run the provided playbook

```
$ git clone https://github.com/id101010/ansible-archlinux.git
$ cd ansible-archlinux/ansible
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
$ cd ansible-archlinux/ansible
$ ansible-playbook --ask-become-pass playbook.yml
```

Now reboot the machine and start a graphical session using virtualbox.
