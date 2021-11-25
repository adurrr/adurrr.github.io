+++
author = "Adur"
title = "Plymouth Configuration"
date = "2021-11-25T13:21:56+01:00"
description = ""
featured = true
tags = [
    "plymouth",
    "debian",
    "free software",
]
categories = [
    "Self-Hosting",
    "Infrastructure"
]
series = ["Debian"]
aliases = [""]
thumbnail = "images/debian.png"
toc = true
+++

This post describes the steps to configure Plymouth on Debian when all partitions are encrypted with LUKS, except for the `/boot` partition. The [plymouth-themes](https://github.com/adi1090x/plymouth-themes) repository is used as a reference.

## What is Plymouth?

Plymouth is an application that starts very early in the boot process (even before the filesystem is mounted) and provides a graphical boot animation while the startup process runs in the background.

It is designed to work on systems that have DRM modesetting drivers. The idea is that very early in the boot process, native modesetting is configured, and Plymouth uses this mode. This mode must be maintained throughout the entire boot process, even after starting the X graphical server. The main purpose is to avoid flickering during the startup process<cite> [^1]</cite>.
[^1]: [Plymouth - Debian](https://wiki.debian.org/es/plymouth)

## Plymouth Installation

```zsh
sudo apt install plymouth
```

## GRUB Modification

Edit the default GRUB configuration file:
```zsh
sudo nano /etc/default/grub
```
Modify line 9 by adding `splash`:
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

## Plymouth Themes

Clone the [themes repository](https://github.com/adi1090x/plymouth-themes.git):
```zsh
git clone https://github.com/adi1090x/plymouth-themes.git
```

Navigate into one of the directories named `pack_X`, for example `pack_3`:
```zsh
cd plymouth-themes/pack_3
```

Select the theme you want, for example `lone`:
```zsh
sudo cp -r lone /usr/share/plymouth/themes/
```