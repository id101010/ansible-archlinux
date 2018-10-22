# Automated ArchLinux

This ansible playbook automates my personal Arch Linux installation. The goal is a fully encrypted desktop system.

## System Overview
* Full Disk Encryption (including /boot if Grub/EFI is used)
* LVM on LUKS
* Plymouth support

## Configurations
* highly customized i3 window manager
* zsh presetup oh-my-zsh theme and custom settings
* rxvt-unicode true color terminal
* tmux with vim bindings

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
then download the playbook and make sure you adjust the values of the global
config in `group_vars/all` to match your system stats. Then run it.

```
$ git clone https://github.com/id101010/ansible-archlinux.git
$ cd ansible-archlinux/ansible
$ vim group_vars/all
$ ansible-playbook --ask-become-pass playbook.yml
```
Lean back and watch the installation.

## Testing

Assuming you've already installed vagrant you can set up a vritual machine with just these steps

``` bash
$ git clone https://github.com/id101010/ansible-archlinux.git
$ cd ansible-archlinux/vagrant
$ vagrant up --provision
```

Now reboot the machine and start a graphical session using virtualbox. 
The default credentials are `user:vagrant pw:vagrant`. 
Alternativly you can log into your machine using the command `vagrant ssh`.

Hint: To reload the configuration into the vagrant box you can eighter reload
(issues a graceful shutdown) the machine using `vagrant reload` or you can update
and apply the configuration changes using `vagrant rsync && vagrant provision`.
This way you don't need to wait for the machine to boot when testing changes.
