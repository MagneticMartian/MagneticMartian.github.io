# Void Linux Install (x86, btrfs, MBR)

## Introduction
In this tutorial we are going to walk through the setup and installation of Void Linux on an x86 MBR system. The root partition will be formated as a btrfs partition. Subvolumes will be implemented as well.

We will go through the process starting from fdisk all the way to the final reconfiguration prior to restart.

## Intro to Btrfs
The B-tree file system (btrfs), or more popularly "Butter/Better fs" is a next generation file system that supports RAID0, RAID1, and RAID10. It's primary features that make it so desirable are: It uses a modern implementation of copy-on-write (CoW), It supports the use of snapshots, as mentioned it supports RAID, and it is a self healing system. Some of the other features that are of interest are: 16 EiB max file size (with a practical limit of 8 EiB), space efficient packing of small files, space-efficient indexed directories, and dynamic inode allocation.

## Installing Btrfs on an x86 MBR System
The following instuctions were implemented on a Panisonic Presario V5000 with the Mobile AMD Sempron 3000+ cpu.

## References
1. https://btrfs.wiki.kernel.org/index.php/Main_Page
2. https://btrfs.wiki.kernel.org/index.php/Getting_started
3. https://wiki.archlinux.org/index.php/Btrfs
4. https://btrfs.wiki.kernel.org/index.php/SysadminGuide
5. https://gist.github.com/gbrlsnchs/9c9dc55cd0beb26e141ee3ea59f26e21
6. https://docs.voidlinux.org/installation/guides/chroot.html
