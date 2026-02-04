+++
author = "Adur"
title = "Comandos útiles para administrar sistemas GNU/linux"
date = "2021-10-21"
description = "Comandos útiles para administrar sistemas"
featured = true
tags = [
    "herramientas",
    "redes"
]
categories = [
    "administración de sistemas",
]
series = ["Administración de sistemas"]
aliases = [""]
thumbnail = "images/GNU_Linux.png"
toc = true
weight = 15
+++

En esta entrada se definen varios comandos útiles en la administración de sistemas.

## Consultar la información de red

```
ip a
```

## Monitorizar el tráfico TCP

```
sudo tcpdump -i <device>
```

## Eliminar ruta por defecto 

```
sudo ip route del default via 192.168.1.1
```

