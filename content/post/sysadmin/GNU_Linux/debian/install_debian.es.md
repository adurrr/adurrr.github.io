+++
author = "Adur"
title = "Instalación de Debian con volúmenes LVM cifrados con LUKS"
date = "2021-04-02"
description = "Instalación de Debian con volúmenes LVM cifrados con LUKS"
featured = true
tags = [
    "GNU/Linux",
    "debian",
    "lvm",
    "cifrado"
]
categories = [
    "soberanía tecnológica",
    "administración de sistemas",
]
series = ["Debian"]
aliases = ["instalacion_debian"]
thumbnail = "images/debian.png"
toc = true
weight = 10
+++

En esta entrada se instalará y se configurará el sistema operativo Debian con volúmenes LVM cifrados con LUKS.

## ¿Qué es Debian?

Debian GNU/Linux es un sistema operativo libre, desarrollado por miles de voluntarios de todo el mundo, que colaboran a través de Internet.

La dedicación de Debian al software libre, su base de voluntarios, su naturaleza no comercial y su modelo de desarrollo abierto la distingue de otras distribuciones del sistema operativo GNU[^1]. 

## ¿Qué es LVM (Logical Volume Manager)?

LVM es una implementación de un gestor de volúmenes lógicos para el núcleo Linux. LVM incluye muchas de las características que se esperan de un administrador de volúmenes, incluyendo:
- Redimensionado de grupos lógicos
- Redimensionado de volúmenes lógicos
- Instantáneas de sólo lectura (LVM2 ofrece lectura y escritura)
- RAID0 de volúmenes lógicos.
LVM no implementa RAID1 o RAID5, por lo que se recomienda usar software específico de RAID para estas operaciones, teniendo las LV por encima del RAID[^2].

En esta configuración no se usará RAID.

## ¿Qué es LUKS (Linux Unified Key Setup)?

LUKS es una especificación de cifrado de disco creado por Clemens Fruhwirth, originalmente destinado para Linux. Mientras la mayoría del software de cifrado de discos implementan diferentes e incompatibles formatos no documentados, LUKS especifica un formato estándar en disco, independiente de plataforma, para usar en varias herramientas. Esto no sólo facilita la compatibilidad y la interoperabilidad entre los diferentes programas, sino que también garantiza que todas ellas implementen gestión de contraseñas en un lugar seguro y de manera documentada. La implementación de referencia funciona en Linux y se basa en una versión mejorada de cryptsetup, utilizando dm-crypt como la interfaz de cifrado de disco[^3]. 

## Tabla de particiones

Se utiliza el formato `ext4` para las particiones porque mejora la velocidad de entrada y salida y utiliza menos CPU que los formatos `ext3` y `ext2`. Se recomiendan como mínimo los siguientes valores:

| **Particion** | **Tamaño recomendado** | **Asignado Debian** | **Asignado Propio** | **Contiene**                                              |
| :------------- | :----------------------- | :---------------------------- | :----------------------------- | :--------------------------------------------------------- |
| /              | >= 750MB                 | 22GB                          | 64GB                           | /etc, /bin, /sbin, /lib, /dev, /usr                        |
| /usr           | >= 4-6GB                 | 0                             | 0                              | Programas de usuario, lib y docs                           |
| /var           | >= 2-3GB                 | 32GB                          | 112GB                          | Variables de datos como emails                             |
| /tmp           | >= 100MB                 | 16GB                          | 32GB                           | Páginas web, caché de paquetes, datos temporales           |
| /home          | >= 100MB                 | 200GB                         | 288GB                          | Directorio con Documentos, Descargas, ...                  |
| /boot          | >= 256MB                 | 500MB                         | 512GB                          | Partición Primaria, ext4 o ext2, no se recomienda cifrarla |
| /boot/efi      | >= 100MB                 | 250MB                         | 0                              | No se recomienda cifrarla y bootable flag: on              |
| /swap          | >= 8GB                   | 16GB                          | 16GB                           | Área de intercambio                                        |

## Pasos seguidos 

Se recomienda conectar la máquina a través de ethernet para que se actualice el sistema durante la instalación.

1. Se configura el idioma, región, teclado, etc.
2. (Saltar paso) __Se realiza unas particiones **manuales**, concretamente se realizan 3, una para /boot, otra para /boot/efi y otra para las demás particiones que irá cifrada a través de LUKS.__ 
3. Se cifra con LUKS y se elige una contraseña de más de 20 carácteres. 
4. Se crea un volumen LVM y posteriormente se realizan las particiones de volumen lógico para cada partición.
5. Se asignan las etiquetas y se termina la configuración de las particiones.
6. Se le da un _hostname_ y se crea el usuario root y el usuario sin privilegios de administrador.

---

## Referencias recomendadas

- [Arch Wiki, dm-crypt/Encrypting an entire system](https://wiki.archlinux.org/index.php/Dm-crypt_(Espa%C3%B1ol)/Encrypting_an_entire_system_(Espa%C3%B1ol))
- [Debian Wiki, LVM](https://wiki.debian.org/LVM)
- [Youtube, How to install Debian GNU/Linux with LUKS encrypted LVM](https://www.youtube.com/watch?v=GEl2S5MI-WU)


[^1]: [Wikipedia, Debian](https://es.wikipedia.org/wiki/Debian_GNU/Linux)
[^2]: [Wikipedia, Logical Volume Manager (Linux)](https://es.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
[^3]: [Wikipedia, LUKS (Linux Unified Key Setup)](https://es.wikipedia.org/wiki/LUKS)

