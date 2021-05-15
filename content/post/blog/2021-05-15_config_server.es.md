+++
author = "Adur"
title = "Configurar un servidor"
date = "2021-05-15"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "debian"
]
categories = [
    "administración de sistemas",
    "soberanía tecnológica"
]
series = ["GNU/Linux"]
aliases = ["server"]
thumbnail = "images/building.png"
toc = true
draft=true
+++

## Crear un nuevo usuario

Crear un usuario llamado admin:
```
adduser admin
```
Añadir el usuario admin al grupo sudo:
```
adduser admin sudo
```
Instalar el paquete sudo
```
apt install sudo
```
Añadir privilegios de root al grupo sudo con:
```
/usr/sbin/visudo
```


## Configurar ssh


```
Port 2251
PermitRootLogin no
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
AuthenticationMethods publickey
PubkeyAuthentication yes
AllowUsers pi
X11Forwarding no
Banner /etc/issue
```