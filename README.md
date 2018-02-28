# Automated ArchLinux

With this project I try to automate my personal Arch Linux installation using
Ansible. The goal is a fully encrypted arch linux desktop system.

## System Overview
* Full Disk Encryption including /boot
* EFI
* LVM
* i3 window manager
* zsh
* rxvt-unicode

## Install Base System

You can eighter install your own minimal system or you follow the instructions provided in [INSTALL.MD](https://github.com/id101010/ansible-archlinux/blob/master/INSTALL.md) to setup a fully encrypted base system with encrypted /boot partition. The Ansible playbook does not depend on this specific installation method.

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

## Testing

Assuming you've already installed vagrant you can set up a vritual machine with
just these steps

``` bash
$ mkdir Vagrant && cd Vagrant
$ vagrant init archlinux/archlinux
$ vagrant up
$ vagrant ssh

$ git clone https://github.com/id101010/ansible-archlinux.git
$ ansible-playbook --ask-become-pass playbook.yml
```

Now reboot the machine and start a graphical session using virtualbox.
