+++
author = "Adur"
title = "Installing Debian with LUKS-encrypted LVM Volumes"
date = "2021-04-02"
description = "Installing Debian with LUKS-encrypted LVM volumes"
featured = true
tags = [
    "GNU/Linux",
    "debian",
    "lvm",
    "encryption"
]
categories = [
    "Self-Hosting",
    "Infrastructure",
]
series = ["Debian"]
aliases = ["instalacion_debian"]
thumbnail = "images/debian.png"
toc = true
+++

This post covers installing and configuring the Debian operating system with LUKS-encrypted LVM volumes.

## What is Debian?

Debian GNU/Linux is a free operating system, developed by thousands of volunteers from around the world who collaborate via the Internet.

Debian's dedication to free software, its volunteer base, its non-commercial nature, and its open development model distinguish it from other GNU operating system distributions[^1].

## What is LVM (Logical Volume Manager)?

LVM is an implementation of a logical volume manager for the Linux kernel. LVM includes many of the features expected from a volume manager, including:
- Resizing of logical groups
- Resizing of logical volumes
- Read-only snapshots (LVM2 offers read and write)
- RAID0 of logical volumes.
LVM does not implement RAID1 or RAID5, so it is recommended to use dedicated RAID software for these operations, placing the LVs on top of the RAID[^2].

RAID will not be used in this configuration.

## What is LUKS (Linux Unified Key Setup)?

LUKS is a disk encryption specification created by Clemens Fruhwirth, originally intended for Linux. While most disk encryption software implements different and incompatible undocumented formats, LUKS specifies a standard on-disk format, platform-independent, for use with various tools. This not only facilitates compatibility and interoperability between different programs, but also ensures that they all implement password management in a secure and documented manner. The reference implementation runs on Linux and is based on an enhanced version of cryptsetup, using dm-crypt as the disk encryption interface[^3].

## Partition Table

The `ext4` format is used for partitions because it improves I/O speed and uses less CPU than the `ext3` and `ext2` formats. The following minimum values are recommended:

| **Partition** | **Recommended Size** | **Debian Allocation** | **Custom Allocation** | **Contains**                                              |
| :------------- | :----------------------- | :---------------------------- | :----------------------------- | :--------------------------------------------------------- |
| /              | >= 750MB                 | 22GB                          | 64GB                           | /etc, /bin, /sbin, /lib, /dev, /usr                        |
| /usr           | >= 4-6GB                 | 0                             | 0                              | User programs, libs and docs                               |
| /var           | >= 2-3GB                 | 32GB                          | 112GB                          | Variable data such as emails                               |
| /tmp           | >= 100MB                 | 16GB                          | 32GB                           | Web pages, package cache, temporary data                   |
| /home          | >= 100MB                 | 200GB                         | 288GB                          | Directory with Documents, Downloads, ...                   |
| /boot          | >= 256MB                 | 500MB                         | 512GB                          | Primary Partition, ext4 or ext2, encryption not recommended|
| /boot/efi      | >= 100MB                 | 250MB                         | 0                              | Encryption not recommended and bootable flag: on           |
| /swap          | >= 8GB                   | 16GB                          | 16GB                           | Swap area                                                  |

## Steps Followed

It is recommended to connect the machine via ethernet so the system updates during installation.

1. Configure the language, region, keyboard, etc.
2. (Skip this step) __Create **manual** partitions, specifically 3: one for /boot, another for /boot/efi, and another for the remaining partitions which will be encrypted with LUKS.__
3. Encrypt with LUKS and choose a password of more than 20 characters.
4. Create an LVM volume and then create the logical volume partitions for each partition.
5. Assign the labels and finish configuring the partitions.
6. Set a _hostname_ and create the root user and a non-privileged user.

---

## Recommended References

- [Arch Wiki, dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/index.php/Dm-crypt_(Espa%C3%B1ol)/Encrypting_an_entire_system_(Espa%C3%B1ol))
- [Debian Wiki, LVM](https://wiki.debian.org/LVM)
- [Youtube, How to install Debian GNU/Linux with LUKS encrypted LVM](https://www.youtube.com/watch?v=GEl2S5MI-WU)


[^1]: [Wikipedia, Debian](https://es.wikipedia.org/wiki/Debian_GNU/Linux)
[^2]: [Wikipedia, Logical Volume Manager (Linux)](https://es.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
[^3]: [Wikipedia, LUKS (Linux Unified Key Setup)](https://es.wikipedia.org/wiki/LUKS)
