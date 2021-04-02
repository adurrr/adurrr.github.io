+++
author = "Adur"
title = "Permisos en GNU/Linux"
date = "2021-04-02"
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
aliases = ["permisos"]
thumbnail = "images/gnu-linux.png"
toc = true
+++


## Crear un grupo

- Se crea el grupo `shared`:
```bash
sudo groupadd shared
```

## Añadir usuario
- Se añade el usuario `devops` al grupo `shared`:
```
sudo usermod -a -G shared devops
```
Para que los cambios surtan efecto se requiere cerrar sesión y volver a entrar en el usuario devops. Se comprobará con el comando:
```
groups
```

## Cambiar el grupo de una carpeta y sus ficheros
- Se cambia el grupo de la carpeta `wiki` por `shared`:
```
sudo chgrp -R shared wiki
```

# Añadir permisos de escritura para el un grupo en una carpeta y sus ficheros

Se añade los permisos de escritura `w` para el grupo de usuarios de la carpeta `wiki`.
```bash
sudo chmod -R g+w wiki
```