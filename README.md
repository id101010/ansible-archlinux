# Automated ArchLinux
This ansible playbook automates my personal Arch Linux installation. The goal is a fully encrypted and secure desktop system. 
All dotfiles are kept in an independent repository. They are managed using [rcm](https://robots.thoughtbot.com/rcm-for-rc-files-in-dotfiles-repos) and will only get installed if the `dotfiles` variable is defined.

## System overview
* Full Disk Encryption (including /boot if Grub/EFI is used)
* LVM on LUKS partitioning scheme
* Plymouth support for a nice boot screen

## Special configuration
* Highly customized i3 window manager
* Zsh with oh-my-zsh theme and custom settings
* Rxvt-unicode true color terminal
* Tmux with vim bindings

## Security features
* Sensitive and internet facing applications are sandboxed using firejail
* Restrictive iptables rules
* Use the of linux-hardened kernel
* Automatic mac address spoofer

## Install base system

You can eighter install your own minimal system or you follow the instructions provided in the two installation guides.

* [INSTALL\_LEGACY](https://github.com/id101010/ansible-archlinux/blob/master/doc/INSTALL_LEGACY.md) to setup an encrypted base system with LVM, syslinux in legacy boot mode.
* [INSTALL\_EFI](https://github.com/id101010/ansible-archlinux/blob/master/doc/INSTALL_EFI.md) to setup a fully encrypted base system with LVM, encrypted /boot partition and EFI support.

The Ansible playbook does not depend on any specific installation method. However the Legacy install is slightly easier and more "user friendly".

## How to run the ansible playbooks

First install ansible
```
$ sudo pacman -S ansible
```
then download and run the provided playbook

```
$ git clone https://github.com/id101010/ansible-archlinux.git
$ cd ansible-archlinux/ansible
$ ansible-playbook -i inventory/hosts playbook.yml [--tags $LIMIT_TO_TAG]
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
