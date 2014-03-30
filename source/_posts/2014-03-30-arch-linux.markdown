---
layout: post
title: "Arch Linux, EFISTUB, Gummiboot, and the Apple bootloader"
date: 2014-03-30 14:08
comments: true
categories:
---

Hi everyone, I recently went through and setup my favorite OS, [Arch Linux](https://www.archlinux.org/),
on a Macbook Air I own but I decided to keep the Apple bootloader. There are
a number of reasons why you might want to keep the Apple bootloader vs. going
down the rEFIt route. I personally chose to pursue this route because I wanted to
keep the seamless appearance of an Apple product. Configuring rEFIt to boot
OS X as the default would be easy enough to do and I highly recommend that approach
if you're not concerned about looks. Finally I thought a writeup of the steps would possibly
help someone else who wants to do this but needs a reference :)


The first step is getting your OS X partition(s) setup correctly and I felt
the [triple boot documentation](https://wiki.archlinux.org/index.php/MacBook#Mac_OS_X.2C_Windows_XP.2C_and_Arch_Linux_triple_boot)
was adequate enough to get the idea of what needed to be done. Here is the
procedure (slightly modified for our purposes):  

1. In Mac OS X, run Disk Utility (located in /Applications/Utilities).  
2. Select the drive to be partitioned in the left-hand column (not the partitions!). Click on the partition tab on the right.
3. Select the volume to be resized in the volume scheme.  
4. Decide how much space you wish to have for your Mac OS X partition, and how much
   for Arch Linux. Remember that a typical installation of Mac OS X requires around 15-20 GiB.  
5. Go ahead and resize the OS X partition leaving the empty space for Arch Linux later in the installation process.
6. Finally, click apply. This will resize your partition and make room for Arch!  

Next step is getting the arch install medium onto a usb drive. I followed
the instructions listed on the [USB Installation Media](https://wiki.archlinux.org/index.php/USB_Installation_Media)
page. After you have the USB drive plugged into the laptop and reboot make sure to hold the `⌥ option key` and select the usb drive
as the boot option.

Great, now you should be presented with a shell prompt and automatically logged
in as root.  

__NOTE:__ I'm making the assumption that your drive will be `/dev/sda`. It could very well
be something completely different just be sure to double check with `fdisk -l` before you
continue.

1. Setup your wired/wireless internet connection ([Establish an internet connection](https://wiki.archlinux.org/index.php/Beginners'_guide#Establish_an_internet_connection))

2. Verify you're booted into UEFI mode by running `efivar -l`. If efivar lists the uefi variables without any error, then you're good to continue

3. Run `fdisk -l` and verify `/dev/sda1` is of type `EFI system`

4. `cd /boot` and `mkdir efi`

5. `mount /dev/sda1 /boot/efi`

6. Run `cgdisk /dev/sda` and arrow down until you've selected the partition type `free space`
    * Choose _New_ and when the tool prompts you for the First sector enter `+128M` (OS X likes to see a 128MB gap after partitions)
    * When the second prompt asks you for the `Size in sectors` press enter if you want to use the rest of the available disk space.
    If you don't want to use the rest of the available disk space then enter the size here
    * Use Hex code 8300 (or press enter as 8300 is the default) for the `Hex code or GUID` prompt
    * Enter whatever partition name you would like to have--I chose Arch Linux as my partition name
    * Write the changes you've made to disk by selecting `Write` and typing `yes` when prompted

7. After partitioning I rebooted then once I was back at a shell I formatted the Arch partition using the
   [Ext4](https://en.wikipedia.org/wiki/Ext4) file system `mkfs.ext4 /dev/sda4`

8. Make sure pacman will use the mirror you want by editing `/etc/pacman.d/mirrorlist`. I personally prefer [RIT](http://www.rit.edu/)
because it's my alma mater so the first line in my mirrorlist file is `Server = http://mirror.rit.edu/archlinux/$repo/os/$arch`

9. Install the base system! `pacstrap -i /mnt base` and press enter for the prompts
   to install all packages

10. Generate your fstab `genfstab -U -p /mnt >> /mnt/etc/fstab`

11. chroot into your base system `arch-chroot /mnt /bin/bash`

12. At this point I'm going to defer to the archwiki step of
    [Chroot and configure the base system](https://wiki.archlinux.org/index.php/Beginners'_guide#Chroot_and_configure_the_base_system).
    Simply follow the wiki until you get to
    [Install and configure a bootloader](https://wiki.archlinux.org/index.php/Beginners'_guide#Install_and_configure_a_bootloader)
    then continue back here

13. So now the fun begins. Assuming you're still in your chrooted setup run `pacman -S gummiboot`

14. Run `mkdir /boot/efi` and `mount /dev/sda1 /boot/efi`

15. `cd /boot/efi` run `gummiboot --path=/boot/efi install` and create `/boot/efi/EFI/arch`

16. copy `/boot/vmlinuz-linux`, `/boot/initramfs-linux.img`, and `/boot/initramfs-linux-fallback.img` to
    `boot/efi/EFI/arch/vmlinuz-arch.efi`, `boot/efi/EFI/arch/initramfs-arch.img`, and `boot/efi/EFI/arch/initramfs-arch-fallback.img` respectively.

17. `cd /boot/efi/loader` and edit the `loader.conf` file to be
    <pre>
    default arch
    timeout 10
    </pre>

18. `cd entries` and create a file named `arch.conf` with the contents of
    <pre>
    title Arch Linux
    efi EFI/arch/vmlinuz-arch.efi
    options initrd=EFI/arch/initramfs-arch.img root=/dev/sda4 rw
    </pre>
    * __NOTE:__ If you don't set the initrd and root paths on the same line you'll
    get an error along the lines of `kernel panic-not syncing: VFS: unable to mount root fs on
    unknown block(0,0)`.

19. `reboot` the computer and when prompted by gummiboot choose to boot OS X

20. Once you've booted into OS X open a terminal and `sudo bless -folder /System/Library/CoreServices -setBoot`
    * __NOTE:__ this sets the OS X default bootloader back to being the default bootloader

21. Reboot and hold down `⌥ option key` and select `EFI BOOT` then select Arch Linux

22. enter your login information and `mount /dev/sda1 /boot/efi && cd /boot/efi/loader`

23. edit `loader.conf` and remove the timeout that was set earlier. Reboot

24. Congratulations! You now have a setup that will on default boot to OS X but if you
    want to boot into Arch all you need to do is hold down `⌥ option key` and select the EFI BOOT option!  

Here's the collection of sources I used while working this out:

* [https://wiki.archlinux.org/index.php/EFISTUB](https://wiki.archlinux.org/index.php/EFISTUB)
* [https://help.ubuntu.com/community/UEFIBooting](https://help.ubuntu.com/community/UEFIBooting)
* [https://wiki.archlinux.org/index.php/Beginners'_guide](https://wiki.archlinux.org/index.php/Beginners'_guide)
* [https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface](https://wiki.archlinux.org/index.php/Unified_Extensible_Firmware_Interface)
* [https://wiki.archlinux.org/index.php/Gummiboot](https://wiki.archlinux.org/index.php/Gummiboot)
* [https://wiki.archlinux.org/index.php/MacBook](https://wiki.archlinux.org/index.php/MacBook)
* [http://www.freedesktop.org/wiki/Software/gummiboot/](http://www.freedesktop.org/wiki/Software/gummiboot/)
* [http://wilkiecat.wordpress.com/2012/02/03/bless-system-in-mac-os-x-if-your-partition-wont-boot/](http://wilkiecat.wordpress.com/2012/02/03/bless-system-in-mac-os-x-if-your-partition-wont-boot/)
