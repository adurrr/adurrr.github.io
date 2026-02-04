+++
author = "Adur"
title = "Creación de un USB Bootable"
date = "2021-01-11"
description = ""
featured = false
tags = [
    "GNU/Linux"
]
categories = [
    "administración de sistemas",
    "soberanía tecnológica"
]
series = ["GNU/Linux"]
aliases = [""]
thumbnail = "images/GNU_Linux.png"
toc = true
weight = 4
+++

## Preparar los ficheros para hacer un USB boot

### Descarga la imagen

```
wget <url-iso>
```

## Quemar la imagen de Arch

Una forma de quemar una imagen es con el comando `dd` como se muestra a continuación:

```sh
# See the partitions
lsblk

# Umount the USB partition
umount /dev/sda1 

# Flash the ISO into USB
sudo dd bs=4M if=archlinux-2022.05.01-x86_64.iso of=/dev/sda conv=fsync oflag=direct status=progress
```

## Random things

* [Cómo crear USB Boot Con Multiples Sistemas Operativos | How to make a USB boot With Multiple OS](https://www.youtube.com/watch?v=K-08hWcMKnA)