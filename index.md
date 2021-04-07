# Void Linux Install (x86, btrfs, MBR)

## Introduction
In this tutorial we are going to walk through the setup and installation of Void Linux on an x86 MBR system. The root partition will be formated as a btrfs partition. Subvolumes will be implemented as well.

We will go through the process starting from fdisk all the way to the final reconfiguration prior to restart.

## Intro to Btrfs
The B-tree file system (btrfs), or more popularly "Butter/Better fs" is a next generation file system that supports RAID0, RAID1, and RAID10. It's primary features that make it so desirable are: It uses a modern implementation of copy-on-write (CoW), It supports the use of snapshots, as mentioned it supports RAID, and it is a self healing system. Some of the other features that are of interest are: 16 EiB max file size (with a practical limit of 8 EiB), space efficient packing of small files, space-efficient indexed directories, and dynamic inode allocation.

## Installing Btrfs on an x86 MBR System
The following instuctions were implemented on a Panisonic Presario V5000 with the Mobile AMD Sempron 3000+ cpu.

For my system the disk that was partitioned was /dev/sda. This can vary on other systems. To find the available disks run:
```
# lsblk
```
It will have an output similar to
```
NAME  MAJ:MIN RM  SIZE  RO  TYPE  MOUNTPOINT
sda     8:0    0  60G    0  disk
sr0    11:0    1  1024M  0  rom
```
Now, we use fdisk to partition the desired disk:
```
# fdisk /dev/sda

Welcome to fdisk (util-linux X.XX.X)
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command


Command (m for help): n
Partition type
   p  primary (0 primary, 0 extended, 4 free)
   e  extended (container for logical partitions)
Select (default p): [Enter]
Partition number (1-4, default 1): [Enter]
First sector (2048-xxxxxxxxxx, default 2048): [Enter]
Last sector, +sectors or +size{K,M,G,T.P} (2048-xxxxxxxxxx, default xxxxxxxxxx): +128M

Created a new partition 1 of type 'Linux filesystem' and of size 128MiB.

Command (m for help): n
Partition type
   p  primary (0 primary, 0 extended, 4 free)
   e  extended (container for logical partitions)
Select (default p): [Enter]
Partition number (1-4, default 2): [Enter]
First sector (XXXXXX-xxxxxxxxxx, default XXXXXX): [Enter]
Last sector, +sectors or +size{K,M,G,T.P} (XXXXXX-xxxxxxxxxx, default xxxxxxxxxx): +2G

Created a new partition 2 of type 'Linux' and of size 2GiB.

Command (m for help): t
Partition number (1-4): 2
Hex code (type L to list codes): 82

Changed system type of partition 2 from 'Linux filesystem' to 'Linux swap/ Solaris OS'

Command (m for help): n
Partition type
   p  primary (0 primary, 0 extended, 4 free)
   e  extended (container for logical partitions)
Select (default p): [Enter]
Partition number (1-4, default 3): [Enter]
First sector (XXXXXX-xxxxxxxxxx, default XXXXXX): [Enter]
Last sector, +sectors or +size{K,M,G,T.P} (XXXXXX-xxxxxxxxxx, default xxxxxxxxxx): [Enter]

Created a new partition 3 of type 'Linux' and of size 53.4GiB.

Command (m for help): w
```
Next we need to format the different partitions:
```
# mkfs.ext4 /dev/sda1
# mkswap /dev/sda2
# swapon /dev/sda2
# mkfs.btrfs /dev/sda3
```
The next step is to mount the btrfs partition and create the desired subvolumes.
```
# mount -t btrfs /dev/sda3 /mnt
# btrfs subvolume create /mnt/@
# btrfs subvolume create /mnt/@home
# btrfs subvolume create /mnt/@snapshots
# umount /mnt
```
Now we need to create the directories that will be mounted inside of /mnt and mount them:
```
# mkdir /mnt/{boot,home,.snapshots}
# mount -t btrfs -o subvol=@ /mnt
# mount -t btrfs -o subvol=@home /mnt/home
# mount -t btrfs -o subvol=@snapshots /mnt/.snapshots
# mount -o rw,noatime /dev/sda2 /mnt/boot
```
It is now a good time to install the base system plus a few extra packages. First, define a few variables. Then, run xbps-install.
```
# REPO=https://alpha.us.repo.voidlinux.org/current
# ARCH=i686
# XBPS_ARCH=$ARCH xbps-install -S -r /mnt -R "$REPO" base-system btrfs-progs vim grub
```
Once this has finished we have the basic FHS directory structure to mount the psuedo-filesystems into
```
# for dir in dev proc sys run; do mount --rbind /$dir /mnt/$dir; mount --make-rslave /mnt/$dir; done
```
Before we can chroot we need to copy the DNS config over.
```
# cp /etc/resolv.conf /mnt/etc
```
We are now ready to chroot and set an options variable to make life a little easier.
```
# PS1='(chroot) # ' chroot /mnt /bin/bash
(chroot) # BTRFS_OPTS="rw,noatime,compress=lzo,space_cache"
```
Next, edit the files /etc/hostname and /etc/default/libc-locales to reflect the desired name and language configuration of the system.
```
(chroot) # vim /etc/hostname
(chroot) # vim /etc/default/libc-locales
```
Set the root password
```
(chroot) # passwd
```
Now comes the most tedious part. Editing the fstab file. We are going to use variables and a here document to do this editting.
```
(chroot) # BOOT_UUID=$(blkid -s UUID -o value /dev/sda1)
(chroot) # SWAP_UUID=$(blkid -s UUID -o value /dev/sda2)
(chroot) # ROOT_UUID=$(blkid -s UUID -o value /dev/sda3)
(chroot) # cat << EOF > /etc/fstab
> UUID=$BOOT_UUID /boot ext4 defaults,noatime 0 2
> UUID=$SWAP_UUID swap swap rw,noatime,discard 0 0
> UUID=$ROOT_UUID / btrfs $BTRFS_OPTS,subvol=@ 0 1
> UUID=$ROOT_UUID /home btrfs $BTRFS_OPTS,subvol=@home 0 2
> UUID=$ROOT_UUID /.snapshots btrfs $BTRFS_OPTS,subvol=@snapshots 0 2
> tmpfs /tmp tmpfs defaults,nosuid,nodev 0 0
> EOF
```
To help streamline the initramfs we will do the following
```
(chroot) # echo hostonly=yes >> /etc/dracut.conf
```
Now it is time to install grub on the system. Remember that we installed the package earlier in the base install step.
```
(chroot) # grub-install /dev/sda
```
As long as this returns a message related to no errors being found, we are ready to reconfigure the system.
```
(chroot) # xbps-reconfigure -fa
```
If this finishes without error it is time to exit, poweroff, and remove the installation media. Once, these steps are done reboot the system and hopefully in a short time we are met with a login screen. Login as root and do the various post install stuff.
## References
1. https://btrfs.wiki.kernel.org/index.php/Main_Page
2. https://btrfs.wiki.kernel.org/index.php/Getting_started
3. https://wiki.archlinux.org/index.php/Btrfs
4. https://btrfs.wiki.kernel.org/index.php/SysadminGuide
5. https://gist.github.com/gbrlsnchs/9c9dc55cd0beb26e141ee3ea59f26e21
6. https://docs.voidlinux.org/installation/guides/chroot.html
