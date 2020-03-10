# How I install Arch Linux

EFI/GPT, single partition, full disk encryption, "LUKS on a partition" (https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition)

This is pretty much following the [Installation Guide](https://wiki.archlinux.org/index.php/Installation_guide) with steps from [LUKS on a partition](https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#LUKS_on_a_partition) inserted at the right places.

It is a good idea to compare the steps from this guide with the steps on the pages linked above - the wiki might have been updated. When in doubt, the official wiki is right.

Written on 2020-03-07

# First steps

Are you booted in EFI mode?

`ls /sys/firmware/efi/efivars`

If the directory exists and contains stuff, yes. If the directory doesn't exist, no.

**This guide assumes that you're booted in EFI mode**

Update system clock

`timedatectl set-ntp true`

# Partition disks

> I use a single root partition because it's the least work to set up.

> I might use a HDD for old photos and videos on my home desktop, but I add that to /etc/fstab later on.

> I don't use swap because I usually have enough RAM and I'm setting up a personal machine - I'd rather have my program OOM-killed than have my system swapping.

Find your drive via `fdisk -l` or `lsblk`.

In this guide, this will be `nvme0n1`

## Pre-wipe the disk

If you had sensitive data on the disk or want to prefill it with random data that is indistinguishable from the encrypted filesystem contents.

See https://wiki.archlinux.org/index.php/Dm-crypt/Drive_preparation#dm-crypt_specific_methods

`cryptsetup open --type plain -d /dev/urandom /dev/nvme0n1 to_be_wiped`

`dd if=/dev/zero of=/dev/mapper/to_be_wiped status=progress bs=1M`

`cryptsetup close to_be_wiped`

Now, the disk should contain no partitions.

## Create partitions

`cfdisk /dev/nvme0n1` (use your drive path)

* Delete every existing partition.
    * Should not be necessary if you pre-wiped the disk.
* Create one 260MiB partition from the start
    * Set its partition type to `EFI system partition`
    * In this guide, this will be `nvme0n1p1`
* Create one partition that spans the rest of the drive
    * Set its partition type to `Linux x86-64 root` 
    * In this guide, this will be `nvme0n1p2`

## Encrypt the root partition

> I chose plain LUKS based on the overview here: https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Overview
> It's easy to set up and I don't need more than one partition.
> Also, it was recommended to me by a colleague whose opinion I respect.

`cryptsetup --use-random luksFormat /dev/nvme0n1p2` 

*Choose an encryption password here and don't forget it. You can change passwords afterwards!*
See https://wiki.archlinux.org/index.php/Dm-crypt/Device_encryption#Cryptsetup_passphrases_and_keys

`cryptsetup open /dev/nvme0n1p2 cryptroot`
The encrypted container name can be chosen freely, it's `cryptroot` here.

`mkfs.ext4 /dev/mapper/cryptroot`

`mount /dev/mapper/cryptroot /mnt`

## Prepare the boot partition

See https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Preparing_the_boot_partition_2

`mkfs.fat -F32 /dev/nvme0n1p1`

`mkdir /mnt/boot`

`mount /dev/nvme0n1p1 /mnt/boot`

# Installation

Pretty much following https://wiki.archlinux.org/index.php/Installation_guide#Installation etc

`vim /etc/pacman.d/mirrorlist` and move a few entries from a nearby country to the top of the list

`pacstrap /mnt base linux linux-firmware`

`genfstab -U /mnt >> /mnt/etc/fstab`

`arch-chroot /mnt`

`ln -sf /usr/share/zoneinfo/REGION/CITY /etc/localtime`

`hwclock --systohc`

`pacman -Syu neovim fish amd-ucode`

`abbr vim nvim`

*If you're not using an AMD cpu, don't install `amd-ucode` but something for your CPU*

`vim /etc/locale.gen`

Uncomment locales you're gonna use

`locale-gen`

`echo "LANG=en_US.UTF-8" > /etc/locale.conf`

`echo "KEYMAP=us" > /etc/vconsole.conf`

`echo HOSTNAME > /etc/hostname`
Choose a sensible hostname

Add the following to `/etc/hosts`:

`127.0.0.1	localhost
::1			localhost
127.0.1.1	HOSTNAME.localdomain	HOSTNAME`

## Going back to LVM settings: Configuring mkinitcpio

See https://wiki.archlinux.org/index.php/Dm-crypt/Encrypting_an_entire_system#Configuring_mkinitcpio_2

`vim /etc/mkinitcpio.conf`

Change the `HOOKS=(...)` line to something like
`HOOKS=(base systemd autodetect keyboard sd-vconsole modconf block sd-encrypt filesystems fsck)`

`mkinitcpio -P`

## root password

`passwd`

Enter your root password twice

## Boot loader/LUKS/LVM

systemd-boot because why not and it works.

`bootctl --path=/boot install`

`mkdir -p /etc/pacman.d/hooks/`

`vim /etc/pacman.d/hooks/100-systemd-boot.hook`
Add the following:
```
[Trigger]
Type = Package
Operation = Upgrade
Target = systemd

[Action]
Description = Updating systemd-boot
When = PostTransaction
Exec = /usr/bin/bootctl update
```

Get the UUID of the LUKS partition (in this case `/dev/nvme0n1p2`) into your boot loader config via `lsblk -f | grep 'nvme0n1p2' | awk '{ print $4 }' >> /boot/loader/entries/arch.conf`

Then add the following around the UUID:
`vim /boot/loader/entries/arch.conf`
Add the following: 
```
title   Arch Linux
linux   /vmlinuz-linux
initrd  /amd-ucode.img
initrd  /initramfs-linux.img
options rd.luks.name=THE_UUID=cryptroot rd.luks.options=discard root=/dev/mapper/cryptroot rw
```

*If you're not using an AMD CPU, replace the `amd-ucode.img` line*
See https://wiki.archlinux.org/index.php/Microcode

> The `rd.luks.options=discard` is required to enable the TRIM operation on SSDs, which is apparently good for SSD health and/or performance: https://wiki.archlinux.org/index.php/Solid_state_drive#TRIM

Enable a weekly trim (which is apparently superior to continuous trim: https://wiki.archlinux.org/index.php/Solid_state_drive#Continuous_TRIM)
`systemctl enable fstrim.timer`

**Only use the discard option and enable the fstrim timer when your SSD supports TRIM!**
See https://wiki.archlinux.org/index.php/Solid_state_drive#TRIM

# Last steps

While the internet is available thanks to the arch installer, get your favorite packages now...

`pacman -Syu man base-devel vi git firefox gnome gnome-tweaks nextcloud-client keepassxc reflector libappindicator-gtk3`

> Package choices:
> man: why is this not in base-devel?
> base-devel: I write software
> vi: to make visudo work
> git: I write software
> firefox: The best browser right now if you want to try supporting Google less. Actually often better than Chrome
> gnome: best desktop for me as of today.
> gnome-tweaks: required for gnome to be customizable
> nextcloud-client: synchronize my data
> keepassxc: my password store
> reflector: Keeps my pacman mirrorlist up to date (https://wiki.archlinux.org/index.php/Reflector)
> libappindicator-gtk3: Required so app indicators of electron apps like slack are displayed. (The gnome extension `KStatusNotifierItem/-Appindicator Support` is a prerequisite for this - but my gnome config is documented in my nextcloud dir)

`systemctl enable gdm NetworkManager`

Add a pacman hook to run the `reflector` tool after the `pacman-mirrorlist` package is updated:

`vim /etc/pacman.d/hooks/mirrorupgrade.hook`
Add the following:
```
[Trigger]
Operation = Upgrade
Type = Package
Target = pacman-mirrorlist

[Action]
Description = Updating pacman-mirrorlist with reflector and removing pacnew...
When = PostTransaction
Depends = reflector
Exec = /bin/sh -c "reflector --country 'INSERT_COUNTRY_HERE' --latest 200 --age 24 --sort rate --save /etc/pacman.d/mirrorlist; rm -f /etc/pacman.d/mirrorlist.pacnew"
```

Create a user

`useradd -m -G wheel INSERT_NAME_HERE`
`passwd INSERT_NAME_HERE`

To be able to `sudo`:
`visudo`
Then navigate to the line saying something about `wheel` and uncomment the line that lets wheel members sudo properly

# Booting into the new system

Exit the chroot shell with `C-d`

`umount -R /mnt`

Fingers crossed you didn't screw up the bootloader entry!

`reboot`

