---
layout:     post
title:      "Arch persistent & encrypted USB install"
subtitle:   "A guide on how to install Arch Linux on a USB stick as persistent and encrypted"
date:       2018-02-14 14:00:00
categories: [linux]
---

# Background
Arch Linux is by far my favorite desktop distro and I use it on my main computer as well as my laptop with no problems.

Why would you want Arch Linux on a USB you say? For me at least, the convenience. To have a ready-to-use Arch install in
your pocket to fit your preferences and needs is amazing. If you have private SSH keys you need to store, this is perfect
for accessing your server everywhere you are with no hassle as well as secured with the encryption.

The reason I decided to write this guide is because I never saw any good guides or articles on a encrypted persistent USB 
install. So hopefully this helps others. 

# Information
I will be using LUKS to encrypt the USB with a 512byte key and SHA512 as the hash, syslinux for the boot loader and no swap. Feel like most computers these days do not need swap at all.

# Sources
If you ever get stuck with this guide or would like to read more I can strongly recommend you to check out the 
[Arch wiki article](https://wiki.archlinux.org/index.php/Installing_Arch_Linux_on_a_USB_key) regarding this subject as well
as [this guide](http://valleycat.org/linux/arch-usb.html) on a persistent USB install but without the encryption.

# Requirements
What you need with this install is:
* A USB stick - would recommend at least 16GiB
* Another Arch or any other distro already installed/Another USB stick(2GiB is enough)

Depending if you already have Arch or another distro already installed or not this section will be different. 

### Arch already installed on host
Great! You just need to run:
```bash
$ sudo pacman -S arch-install-scripts
```
to install the essentials and you are ready to go, just skip to the **Identify the node** section.

### Another USB stick
Then you will need to create a live archiso and boot from it. There are multiple ways to do this depending on what OS you
 are currently using. There is a great article on the Arch Wiki [here](https://wiki.archlinux.org/index.php/USB_Flash_Installation_Media) covering basically every methods. I would recommend:
```bash
$ dd bs=4M if=/path/to/archlinux.iso of=/dev/sdX status=progress oflag=sync
```
Change */dev/sdX* to whatever you have, if you are not sure. Check the **Identify the node** section.

### Another distro installed on host
Never done this but read [this](https://wiki.archlinux.org/index.php/Install_from_existing_Linux#From_a_host_running_another_Linux_distribution) section and try to follow along as good as possible.

### From Windows/OS X or another OS
Download VirtualBox and VirtualBox Extensions, add a VM with the archiso and start in live mode. Then attach the USB device you want the persistent encrypted install on.

# Secure erase
Now that we have that covered, we can start preparing our USB. Just follow the steps on the [Arch Wiki](https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation). This is if you have sensitive data on your USB. From the article:

*"it is recommended to perform a secure erase of the disk by overwriting the entire drive with random data. To prevent cryptographic attacks or unwanted file recovery"*

Just keep in mind that this may take a long time.


# Root
Before we begin, we want to become root instead of using sudo in every command. So if you are not already root do:
```bash
$ sudo su
```

# Identify the node
First things first, list your devices with:
```bash
$ lsblk
```
or with:
```bash
$ fdisk -l
```
and identify which /dev node was assigned. For me, it was */dev/sdc* but for you it may be different. So replace */sdc*
with whatever node you got assigned. **Do not just copy the command without knowing your *node*.**

# Internet
Before you start you need to connect to the internet. Depending if you have a wire or not, this is different. Check out 
[this](https://wiki.archlinux.org/index.php/Installation_guide#Connect_to_the_Internet)

# Partitioning
I will be using *fdisk* to format the USB. So follow along(just remember to change /dev/sdc to your node):
```bash
$ fdisk /dev/sdc
```
Now create a new GPT disklabel and two partitions, we will be using the first one for boot, 256MiB, and the second one for 
the actual encrypted disk.
```bash
Command (m for help): g
Created a new GPT disklabel

Command (m for help): n
Partition number (1-128, default 1):
First sector (2048-121307102, default 2048):
Last sector, +sectors or +size{K,M,G,T,P} (2048-121307102, default 121307102): +256MiB

Created a new partition 1 of type 'Linux filesystem' and of size 256 MiB.

Command (m for help): n
Partition number (2-128, default 2):
First sector (526336-121307102, default 526336):
Last sector, +sectors or +size{K,M,G,T,P} (526336-121307102, default 121307102):

Created a new partition 2 of type 'Linux filesystem' and of size 57.6 GiB.

Command (m for help): w
```
Command used was: **g -> n -> 2x[ENTER] -> +256MiB -> n -> 3x[ENTER] -> w**

Now if we run *lsblk /dev/sdc*. We should see two partitions on our disk:
```bash
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sdc      8:32   1 57.9G  0 disk
├─sdc1   8:33   1  256M  0 part
└─sdc2   8:34   1 57.6G  0 part
```
# Encryption
Next is to create our */dev/sdc2* (change /sdX2 to your node) encryption using *cryptsetup* issuing this command:
```bash
$ cryptsetup luksFormat --hash sha512 --key-size 512 --iter-time 3000 --use-random /dev/sdc2
```
If you want to read more about LUKS and the options I used, you can do it [here](https://wiki.archlinux.org/index.php/Dm-crypt/Device_Encryption#Encryption_options_for_LUKS_mode)

Now write *YES* in uppercase and the password you want to use twice.
We need to unlock the encrypted partition in order to work with it using:
```bash
$ cryptsetup luksOpen /dev/sdc2 cryptroot-usb
```
Then enter your passphrase, I gave it the name *cryptroot-usb* to have something unique. You can name it whatever you want. Just remember to change the name every time it gets mention in this guide.

# Filesystem & Mounting
Next thing we have to do is to create our filesystem:
```bash
$ mkfs.ext4 -O "^has_journal" -O ^64bit /dev/sdc1
$ mkfs.ext4 -O "^has_journal" /dev/mapper/cryptroot-usb
```
*/dev/mapper/cryptroot-usb* is our unlocked partition.

Now we have to mount our freshly created filesystems using:
```bash
$ mount /dev/mapper/cryptroot-usb /mnt/
$ mkdir /mnt/boot
$ mount /dev/sdc1 /mnt/boot/
```
If you already have */mnt/* occupied, you can just create another directory and use that as target. Just change the name 
to your directory every time */mnt/* gets mentioned.

# Mirrors
You should already have good mirrors if you are using Arch on your host and you can skip this part. But if you are not, the easiest way is to *wget* the mirrors:
```bash
$ wget "archlinux.org/mirrorlist/?country=XX" -O /etc/pacman.d/mirrorlist.b
```
Change *XX* to your country code, I live in Sweden for example so for me it would be */?country=SE*

Now remove the '#' at the start of every server using sed and after that, use *rankmirrors* to output the 3 fastest:
```bash
$ sed -i 's/^#Server/Server/' /etc/pacman.d/mirrorlist.b
$ rankmirrors -n 3 /etc/pacman.d/mirrorlist.b > /etc/pacman.d/mirrorlist
```
*The sed command basically replaces '#Server' with 'Server' in the mirrorlist.b file.*

# Pacstrap
Now that we have our fast mirrors ready, it's time to install the base on our USB. This is done by a command called 
pactrap.
```bash
$ pacstrap /mnt base base-devel vim
```

I like using vim to edit files, so I install it directly with pacstrap. It's optional though.

# Fstab & Chroot
Now that the base is installed on our USB stick it's time to generate the fstab and then chroot into our newly created 
install. *-U* for using UUIDs instead of names. This is important.
```bash
$ genfstab -U -p /mnt >> /mnt/etc/fstab
$ arch-chroot /mnt
```
# Standard configuration
Now that we have changed our root to the USB it's time to configure our system. We will start with setting our locale. I use
 *en_US.UTF-8* but you can change this to your preferred language:
```bash
$ echo "en_US.UTF-8 UTF-8" > /etc/locale.gen
$ locale-gen
$ echo "LANG=en_US.UTF-8" > /etc/locale.conf
$ export LANG=en_US.UTF-8
```
Now that our locales are correctly setup, we want to create a symbolic link with our correct time-zone:
```bash
$ ln -s /usr/share/zoneinfo/Europe/Stockholm /etc/localtime
```
*Change /Europe/Stockholm to where you are located*

Set our hardware clock to match:
```bash
$ hwclock --systohc
```
Then set a hostname:
```bash
$ echo YOURNAME > /etc/hostname
$ vim /etc/hosts
```
Just add these lines in */etc/hosts*:
```bash
127.0.0.1       localhost.localdomain   localhost YOURNAME
::1             localhost.localdomain   localhost YOURNAME
```
Change the root password:
```bash
$ passwd
```
Add a user and change the password:
```bash
$ useradd -m -s /bin/bash USERNAME
$ passwd USERNAME
```
Then add sudo privileges to our new user with:
```bash
$ visudo
```
or if you want to use *nano*:
```bash
$ EDITOR=nano visudo
```
and look for *"root ALL=(ALL) ALL"* in that directory. Once you find it, copy it and just change *root* to your username. The output should be as following:
```bash
##
## User privilege specification
##
root ALL=(ALL) ALL
USERNAME ALL=(ALL) ALL
```

# Mkinitcpio
We need to change the hooks in */etc/mkinitcpio.conf* to fit our system:
```bash
$ vim /etc/mkinitcpio.conf
```
Look for the HOOKS line and change the order like so:
```bash
HOOKS=(base udev block encrypt filesystems keyboard fsck)
```
and run:
```bash
$ mkinitcpio -p linux
```

# USB configuration
Following [this](http://valleycat.org/linux/arch-usb.html) article I linked before we want to remove any hardware 
network device naming:
```bash
$ ln -s /dev/null /etc/udev/rules.d/80-net-setup-link.rules
```
Enable the journal to write to RAM to reduce writes on the USB:
```bash
$ sed -i 's/^#Storage=auto/Storage=volatile/' /etc/systemd/journald.conf
```
Ensure that the journal does not overfill the RAM:
```bash
$ sed -i 's/^#SystemMaxUse=/SystemMaxUse=16M/' /etc/systemd/journald.conf
```
Change the *relatime* to *noatime* in */etc/fstab* on root (/dev/mapper/cryptroot-usb). This is for limiting the writing of
 the metadata.
```bash
$ vim /etc/fstab
```

# Syslinux
Now we need to install and configure our bootloader. Install it along with *gptfdisk*:
```bash
$ pacman -S gptfdisk syslinux
```
Then run:
```bash
$ syslinux-install_update -iam
```
*-i (install the files), -a (mark the partition active with the boot flag), -m (install the MBR boot code)*

I like just having my bootloader to boot for me with no screen or choice at 
all. I just want the simplicity of booting directly into my OS and the config 
for that is very simple. So if you want to customize your syslinux different, 
then go ahead. But this is what I'm going to use. First open up *syslinux.cfg*
```bash
$ vim /boot/syslinux/syslinux.cfg
```

Then remove everything and paste in this if you just want it to boot without 
any prompts:
```bash
DEFAULT arch
LABEL arch
    LINUX ../vmlinuz-linux
    APPEND cryptdevice=UUID=YOUR-DEV-SDX2-UUID:cryptroot-usb root=/dev/mapper/cryptroot-usb rw
    INITRD ../initramfs-linux.img
```
What you need to change is the highlighted cryptdevice=**UUID=YOUR-DEV-SDX2-UUID**:cryptroot-usb 
and to find the UUID on (in my case */dev/sdc2*) we can run:
```bash
$ blkid /dev/sdc2
```
You will see *UUID="XXXXXXXX-XXXX-XXXX-XXXXXXXXXXXX"* just copy the output you 
got and if that was my UUID for example, my *syslinux.cfg* would look like this:
```bash
DEFAULT arch
LABEL arch
    LINUX ../vmlinuz-linux
    APPEND cryptdevice=UUID=XXXXXXXX-XXXX-XXXX-XXXXXXXXXXXX:cryptroot-usb root=/dev/mapper/cryptroot-usb rw
    INITRD ../initramfs-linux.img
```

# Drivers & essentials
Now all we have to is to install our GUI, drivers and some other essential software. Things we are going to install are:
```bash
$ pacman -S xf86-video-vesa xf86-video-ati xf86-video-intel xf86-video-amdgpu xf86-video-nouveau
```
Video drivers, we need multiple to support common GPUs. We are probably using our USB on different hardware.

```bash
$ pacman -S xf86-input-synaptics
```
Standard notebook touchpad/touchscreen for laptop use.

```bash
$ pacman -S acpi
```
Support for checking battery charge and state.

```bash
$ pacman -S iw wpa_supplicant dialog ifplugd
```
Packages for managing networking. Both wired and wireless. If you want to connect to wifi, just use *sudo wifi-menu -o* once
 you rebooted

Turn on Network Time Protocol to synchronize system time:
```bash
$ timedatectl set-ntp true
```

---

Congratulations! You just got a working Arch install on your USB device, just reboot and you are all done. From here on you can install your compositor and 
 window manager or desktop environment. Personally, I use xorg and i3-gaps. If you want the same setup as me, you can 
still follow along.

# GUI & AUR manager
To get our graphical user interface to work we need xorg/wayland and a window manager/desktop environment. 
I will be installing xorg along with i3-gaps:
```bash
$ pacman -S xorg xorg-xinit i3-gaps
```
Now that we have that installed we want to autostart it at boot, if you prefer not to you can skip 
this part. But to be able to autostart at boot we need to edit/add two files: *bash_profile* and *xinitrc*

In **~/.bash_profile** add this:
```bash
XDG_CONFIG_HOME="$HOME/.config"
export XDG_CONFIG_HOME
if [ -z "$DISPLAY" ] && [ -n "$XDG_VTNR" ] && [ "$XDG_VTNR" -eq 1 ]; then
    exec startx
fi
```
If you want to use zsh instead. Just add that in *.zprofile*

And in **~/.xinitrc** add this:
```bash
#!/bin/sh

[[ -f ~/.Xresources ]] && xrdb -merge ~/.Xresources
exec i3
```

I like having a AUR package manager and my preferred one is *trizen* so that's what I'll install:
```bash
$ git clone https://aur.archlinux.org/trizen-git.git
$ cd trizen-git
$ makepkg -si
```

# All done!
Now you have a fully graphical, persistent, encrypted working Arch environment on a USB stick! 
From here on you can to whatever you like with your install. 

If you are into pentesting and security like me. I can recommend you to take a look at [BlackArch](https://www.blackarch.org/index.html).
It's a great alternative to Kali and you can even install it on top of your installation you just did.

Thanks for reading!
