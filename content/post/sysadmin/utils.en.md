+++
author = "Adur"
title = "Useful commands for GNU/Linux system administration"
date = "2021-10-21"
description = "Useful commands for system administration"
featured = true
tags = [
    "tools",
    "networking"
]
categories = [
    "Infrastructure",
]
series = ["System Administration"]
aliases = [""]
thumbnail = "images/GNU_Linux.png"
toc = true
+++

This post covers several useful commands for system administration.

## Query network information

```
ip a
```

## Monitor TCP traffic

```
sudo tcpdump -i <device>
```

## Delete the default route

```
sudo ip route del default via 192.168.1.1
```
