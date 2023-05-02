---
tags: ["Arch"]
title: "Chronicles Of My First Arch Installation"
date: 2023-04-19T12:11:48+02:00
---

I've been meaning to start a blog for a couple of months now, but I kept putting it off it, not knowing what to write about at first. Last week I installed Arch Linux on my laptop, so I guess this is the right place to start.

If you don't know what Arch Linux is, I'm glad to be the one to introduce it to you. You can read more about it [here](https://archlinux.org/about/). You may also want to get more familiar with Linux in general before reading the rest of this blog post.

## Some background information

To give you some context, about a year and a half ago, I decided to install a Linux distribution on my laptop in a dual-boot with Windows 10. After some research, I chose an Arch-based distribution called [Arcolinux](https://arcolinux.com/). Since then, I've used Windows only a dozen times, exclusively to access the platform required to take exams at university.

During this time, I've customized my setup and played around with different configuration files, but I've often struggled to figure out how a certain functionality is achieved. The system as a whole worked fine, but I wanted to understand what made it work. That's why I decided to make the jump to pure Arch. I figured that by building the system myself, I could get a better understanding of what each piece does.

## Disclamer

I hope it's now clear that I'm not an expert on the subject and that I'm learning things along the way. So take everything I say with a grain of salt and feel free to correct me if you see any mistakes.

The purpose of this post is to share with you the difficulties I encountered during my installation and how I overcame them. If you're trying to install Arch on a laptop similar to mine (Asus Zephyrus M GU502GU), you'll probably run into the same problems. So it may be helpful to have seen them before.

Keep in mind, however, that this is by no means intended to be a general guide to installing Arch. I suggest you do your own research, ideally by reading the [official installation guide](https://wiki.archlinux.org/title/installation_guide), and make sure you have at least a vague idea of what each command does before you run it.

## Preparation

To prepare for the installation, I downloaded the ISO and burned it to a USB stick, then copied all the files I wanted to keep to an external SSD.

Since I was going to be tinkering with the partitions on my hard drive, I took the opportunity to shrink the Windows partition as much as possible. Following this [guide](https://superuser.com/a/1060508), I defragmented the C drive and then compressed it, freeing up a grand total of 79GB.

## Let the installation start

To avoid having to connect to the Wi-Fi from the terminal, I connected my laptop to an ethernet cable. I plugged in the USB stick with the ISO on it, powered on the laptop and accessed the boot menu. From there, I selected to boot from the USB stick and made sure safe boot was disabled.

After some welcome messages and fast scrolling text, I was presented with a black screen with a prompt at the top. This is were the actual installation starts.

Right away, I changed the keyboard layout to Italian, my native language.

```bash
loadkeys it
```

Then, I synced the system's clock using the [Network Time Protocol](https://en.wikipedia.org/wiki/Network_Time_Protocol) (NTP).

```bash
timedatectl set-ntp true
```

Now we get to partitioning the hard drive, which is arguably the most dangerous part of the installation. A mistake at this stage can result in the loss of all your data, so make sure you have a backup.

I ran the `lsblk` command to list all [block devices](https://en.wikipedia.org/wiki/Device_file#Block_devices). The name of my main disk indicates that it is a [NVME](https://en.wikipedia.org/wiki/NVM_Express) SSD. To get more information about its partitions I ran the `fdisk -l` command. The output looked something like this:

```
NAME        MAJ:MIN RM   SIZE RO TYPE
nvme0n1     259:0    0 476.9G  0 disk
├─nvme0n1p1 259:1    0   260M  0 EFI System
├─nvme0n1p2 259:2    0    16M  0 Microsoft reserved
├─nvme0n1p3 259:3    0   191G  0 Microsoft basic data
├─nvme0n1p4 259:4    0 203.5G  0 Linux Filesystem
└─nvme0n1p6 259:5    0   850M  0 Windows recovery environment
```

I started `cfdisk` to partition the drive.

```bash
cfdisk /dev/nvme0n1/
```

The plan was to delete my old Linux file system (`nvme0n1p4`), create a small partition (256MB) to store the [boot loader](https://wiki.archlinux.org/title/Arch_boot_process#Boot_loader) and give the rest of the available space to the new Linux file system.

However, I realized that there was already an EFI partition, which is where the boot loader should be stored. This is what Windows 10 uses to boot. I checked the Arch wiki to understand if I needed to create a new partition or use the same one. The latter turned out to be the correct option, so I proceeded to install grub on the existing EFI partition, not without some difficulty.

First, I formatted the main partition as [ext4](https://en.wikipedia.org/wiki/Ext4).

```
mkfs.ext4 /dev/nvme01p4
```

Then, I mounted the main and boot partitions to the default location of `/mnt` and `/mnt/boot`, respectively.

```bash
mount /dev/nvme01p4 /mnt
mkdir /mnt/boot
mount /dev/nvme01p1 /mnt/boot
```

I ran the [pacstrap](https://man.archlinux.org/man/pacstrap.8) script to install the [base](https://archlinux.org/packages/core/any/base/) and [base-devel](https://archlinux.org/packages/core/any/base/) packages, which contain the tools for a basic Arch Linux installation (whatever that means), the Linux [kernel](https://wiki.archlinux.org/title/Kernel), some firmware for common hardware, and finally the text editor of my choice.

```bash
pacstrap /mnt base base-devel linux linux-firmware vim
```

I created the `/etc/fstab` file, which [stores static information](https://linuxconfig.org/how-fstab-works-introduction-to-the-etc-fstab-file-on-linux) about the file system and its mount options. By default, the block device label (e.g. `nvme0n1p4`) is used as the identifier, but it's preferable to use the UUID (Universal Unique IDentifier) instead, as this is guaranteed to remain the same under all circumstances. This is achieved by using the `-U` flag.
```bash
genfstab -U /mnt >> /mnt/etc/fstab
```

Finally, I moved into `/mnt` to access the new file system I'd just mounted.

```bash
arch-chroot /mnt /bin/bash
```

This is confirmed by a change in the command prompt letting you know that you are now in the root directory (`/`) of the new file system. Running the `ls` command will list all the standard Linux directories.

There I installed and enabled the [program](https://wiki.archlinux.org/title/NetworkManager) to manage network connections and the boot loader.

```bash
pacman -S networkmanager grub
systemctl enable NetworkManager
grub-install /dev/sda
```

This is where the struggle began.

I was getting some errors from `grub-install` saying that it couldn't determine which firmware to install and that it couldn't find the EFI directory. After some research on forums and wikis and a few more failed attempts, I found out that I needed to install the [efibootmgr](https://archlinux.org/packages/core/x86_64/efibootmgr/) package, which is required for booting UEFI systems. I tried again, specifying the name of the EFI directory, and it worked.

```bash
pacman -S efibootmgr
grub-install /dev/nvme01n1 --efi-directory=boot
grub-mkconfig -o /boot/grub/grub.cfg
```

The output of the last command contained a warning telling me that os-prober would not be executed. Of course, I completely ignored it.

I later found out what it meant when I noticed that the option to boot into Windows had disappeared from the boot menu. For a moment, I feared I had deleted my Windows partition. Fortunately, that wasn't the case, and I was able to get the option back on the menu by following [these instructions](https://wiki.archlinux.org/title/GRUB#Detecting_other_operating_systems).

With grub finally configured, I ran the `passwd` command to set a strong and complicated password for root. As you'll soon see, I later disabled the root login altogether, as it is insecure to access your system that way.

I set my [locale](https://en.wikipedia.org/wiki/Locale_(computer_software)) to US by uncommenting the lines with `en_US.UTF-8 UTF-8` and `es_US ISO-8859-1` from the `/etc/locale.gen` file.

```bash
vim /etc/locale.gen
locale-gen
```

After that, I made my change of console keyboard layout permanent

```bash
echo "KEYMAP=it" > /etc/vconsole.conf
```

set the system's language as US English

```bash
echo "LANG=en_US.UTF-8" > /ect/local.conf
```

and set my timezone.

```bash
ln -sf /usr/share/zoneinfo/Europe/Rome /etc/localtime
```

Finally, I exited the `chroot` environment and unmounted the file system.

```bash
exit
umount -R /mnt
```

I removed the USB stick containing the ISO and rebooted. The installation had succeeded. I logged in and, as one should, immediately installed and ran neofetch.

![](https://i.imgur.com/AVADA4u.jpg)

After enjoying this sight with pride, I created a new user and added it to the `wheel` group. Then I edited the `/etc/sudoers` file to give root permissions to the members of said group.

```bash
useradd -mG wheel gb
passwd gb
vim /etc/sudoers
# uncomment the line %wheel ALL=(ALL) ALL
```

I logged out and logged back in as `gb`. Since locking myself out of my newly installed system would have been rather embarrassing, I made sure I actually had root privileges before disabling root login forever.

```bash
passwd -l root
```

## Graphical environment

This completed the so-called minimal installation, but all I had was a black terminal. Not exactly the most practical setup. To fix this, I installed the display server [Xorg](https://wiki.archlinux.org/title/xorg) and my favorite window manager [Qtile](http://www.qtile.org/) along with the program I use to manage keybindings and a nice font.

```bash
pacman -S xorg-server xorg-xinit xf86-video-intel Qtile sxhkd ttf-inconsolata
```

Following the instructions for [xinit](https://wiki.archlinux.org/title/xinit), I created the `.xinitrc` file in my home directory, which is run when the X server is started. In it, I added the command to start my window manager.

```bash
qtile start
```

Afterwards, I [changed](https://wiki.archlinux.org/title/command-line_shell#Changing_your_default_shell) my default shell to zsh and added the following lines to the `.zprofile` file.

```bash
if [ -z "${DISPLAY}" ] && [ "${XDG_VTNR}" -eq 1 ]; then
  exec startx
fi
```

This way, every time I log in, the X server is automatically started and Qtile is launched, giving me access to a graphical environment.

From there, I installed a bunch of useful programs, cloned my Qtile config from my GitHub profile, and copied all the files I had backed up to the to the external SSD. Then I [connected to the wifi](https://wiki.archlinux.org/title/NetworkManager#Usage) and fixed some issues with the audio not working.

## Conclusions

In the time since the installation, I've been fixing small issues and improving my setup, and I expect to continue doing so for quite some time. In the process, I will start adding my dotfiles to a dedicated [repository](https://github.com/GBergatto/dotfiles) on my Github account. Feel free to check it out to see how my setup is taking shape.

More posts about the programs I use most will follow.

I also plan to learn more about the inner workings of Linux, so I'll be writing about that as well. Stay tuned.

