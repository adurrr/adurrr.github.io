+++
author = "Adur"
title = "Creating a Bootable USB Flashdrive"
date = "2021-01-11"
description = ""
featured = false
tags = [
    "GNU/Linux"
]
categories = [
    "sysadmin",
    "technological sovereignty"
]
series = ["GNU/Linux"]
aliases = [""]
thumbnail = "images/GNU_Linux.png"
toc = true
weight = 4
+++

## Preparing Files for USB Memory Stick Booting

### Download the image

```zsh
wget <url-iso>
```

### Flash the ISO into USB

First of all is to identified the partition of the USB connected. We use `lsblk` command and then, we unmount the partition. After that, we format in vFAT and, finnally, flash the ISO into USB.

One way is to burn an image is with the `dd` command as shown below:

```sh
# See the partitions
lsblk

# Umount the USB partition
sudo umount /dev/sdc1 

# Flash the ISO into USB
sudo dd bs=4M if=image.iso of=/dev/sda conv=fsync oflag=direct status=progress
```