+++
author = "Adur"
title = "Configuración del Plymouth"
date = "2021-11-25T13:21:56+01:00"
description = ""
featured = true
tags = [
    "plymouth",
    "debian",
    "software libre",
]
categories = [
    "soberanía tecnológica",
    "administración de sistemas"
]
series = ["Debian"]
aliases = [""]
thumbnail = "images/debian.png"
toc = true
+++

En esta entrada se definen los pasos a seguir para configurar el playmouth en Debian teniendo todas las particiones cifradas con LUKS, menos la partición `/boot`. Se utiliza como refencia el repositorio [plymouth-themes](https://github.com/adi1090x/plymouth-themes).

## ¿Qué es el Playmouth?

Plymouth es una aplicación que se inicia muy temprano en el proceso de booteo o inicio (incluso antes de que el sistema de archivos esté montado) que proporcional una animación gráfica de booteo o inicio mientas el proceso de inicio ocurre en segundo plano.

Esta diseñado para trabajar en sistemas que tengan drivers DRM modesetting. La idea es que muy temprano en el proceso de booteo o inicio se configure de forma nativa el modesetting, Plymouth usa este modo, este modo debe mantenerse durante todo el proceso de booteo o inicio incluso después de iniciar el servidor gráfico X. El máximo propósito es evitar los parpadeos durante el proceso de inicio<cite> [^1]</cite>.
[^1]: [Plymouth - Debian](https://wiki.debian.org/es/plymouth)

## Instalación de Playmouth

```zsh
sudo apt install plymouth
```

## Modificación del Grub

Modificar el archivo grub que viene por defecto:
```zsh
sudo nano /etc/default/grub
```
Modificamos la línea 9 añadiendo `splash`: 
```
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash"
```

## Temas para el plymouth

Clonamos el [repositorio de temas](https://github.com/adi1090x/plymouth-themes.git):
```zsh
git clone https://github.com/adi1090x/plymouth-themes.git
```

Nos movemos dentro de algún directorio llamado `pack_X`, por ejemplo el `pack_3`:
```zsh
cd plymouth-themes/pack_3
```

Seleccionamos el tema que queremos, por ejemplo `lone:
```zsh 
sudo cp -r lone /usr/share/plymouth/themes/
```