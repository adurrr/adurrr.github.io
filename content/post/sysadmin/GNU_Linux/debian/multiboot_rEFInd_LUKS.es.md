+++
author = "Adur"
title = "Instalación de Debian y ParrotOS con arranque dual en una partición cifrada con LUKS, volúmenes LVM y rEFInd"
date = "2022-01-06"
description = ""
featured = false
tags = [
    "debian",
    "parrot",
    "rEFInd",
    "luks",
]
categories = [
    "administración de sistemas",
    "soberanía tecnológica",
    "seguridad"
]
series = ["GNU/Linux"]
aliases = [""]
thumbnail = "images/debian.png"
toc = true
weight = 20
+++

En esta entrada se instalará y se configurará un arranque dual de múltiples distribuciones de GNU/Linux (Debian y ParrotOS) mediante rEFInd. Si usas GRUB para arrancar múltiples distribuciones y nunca has oído hablar de rEFInd entonces esta publicación es para ti. Hemos seguido las indicación de [este artículo](https://teejeetech.com/2020/09/05/linux-multi-boot-with-refind/).

> **Nota importante**: la instalación que se describe a continuación se ha realizado sobre un disco de almacenamiento limpio. Si tienes datos que quieras guardar, se recomienda encarecidamente hacer una copia de seguridad de todos los datos para realizar la instalaciones sobre un disco vacío.

## Requisitos

- Disco de almacenamiento limpio.
- [USB flasheado con Debian](../usb_boot.es.md) y posteriormente con Parrot
- Revisar los conceptos básicos

## Conceptos básicos

Debian GNU/Linux es un sistema operativo libre, desarrollado por miles de voluntarios de todo el mundo, que colaboran a través de Internet.

La dedicación de Debian al software libre, su base de voluntarios, su naturaleza no comercial y su modelo de desarrollo abierto la distingue de otras distribuciones del sistema operativo GNU[^1]. 

LVM es una implementación de un gestor de volúmenes lógicos para el núcleo Linux. LVM incluye muchas de las características que se esperan de un administrador de volúmenes, incluyendo:
- Redimensionado de grupos lógicos
- Redimensionado de volúmenes lógicos
- Instantáneas de sólo lectura (LVM2 ofrece lectura y escritura)
- RAID0 de volúmenes lógicos.
LVM no implementa RAID1 o RAID5, por lo que se recomienda usar software específico de RAID para estas operaciones, teniendo las LV por encima del RAID[^2].

En esta configuración no se usará RAID.

LUKS es una especificación de cifrado de disco creado por Clemens Fruhwirth, originalmente destinado para Linux. Mientras la mayoría del software de cifrado de discos implementan diferentes e incompatibles formatos no documentados, LUKS especifica un formato estándar en disco, independiente de plataforma, para usar en varias herramientas. Esto no sólo facilita la compatibilidad y la interoperabilidad entre los diferentes programas, sino que también garantiza que todas ellas implementen gestión de contraseñas en un lugar seguro y de manera documentada. La implementación de referencia funciona en Linux y se basa en una versión mejorada de cryptsetup, utilizando dm-crypt como la interfaz de cifrado de disco[^3]. 

Un `boot loader` carga un kernel del sistema operativo en la memoria y lo ejecuta. Un `boot manager` entrega el control a otro programa de arranque. GRUB es tanto un `boot loader` como un `boot manager`. Refind es sólo un `boot manager`.

Otro concepto fundamental es conocer la diferencia entre EFI/UEFI y BIOS.

LVM es una implementación de un gestor de volúmenes lógicos para el núcleo Linux. LVM incluye muchas de las características que se esperan de un administrador de volúmenes, incluyendo:
- Redimensionado de grupos lógicos
- Redimensionado de volúmenes lógicos
- Instantáneas de sólo lectura (LVM2 ofrece lectura y escritura)
- RAID0 de volúmenes lógicos.
LVM no implementa RAID1 o RAID5, por lo que se recomienda usar software específico de RAID para estas operaciones, teniendo las LV por encima del RAID[^2].

En esta configuración no se usará RAID.

LUKS es una especificación de cifrado de disco creado por Clemens Fruhwirth, originalmente destinado para Linux. Mientras la mayoría del software de cifrado de discos implementan diferentes e incompatibles formatos no documentados, LUKS especifica un formato estándar en disco, independiente de plataforma, para usar en varias herramientas. Esto no sólo facilita la compatibilidad y la interoperabilidad entre los diferentes programas, sino que también garantiza que todas ellas implementen gestión de contraseñas en un lugar seguro y de manera documentada. La implementación de referencia funciona en Linux y se basa en una versión mejorada de cryptsetup, utilizando dm-crypt como la interfaz de cifrado de disco[^3]. 

En la tabla de particiones Tabla de particiones se utiliza el formato `ext4` para las particiones porque mejora la velocidad de entrada y salida y utiliza menos CPU que los formatos `ext3` y `ext2`. Se recomiendan como mínimo los siguientes valores:

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


## GParted

Mediante GParted ejecutado desde un USB live, se eliminan todas las particiones y se crea una nueva tabla de particiones GPT(GUID Partition Table). GPT es un formato utilizado por los sistemas con EFI y es una alternativa moderna al formato MBR(Master Boot Record) que se utiliza en los sistemas con BIOS.

UEFI requiere que cada disco de arranque tenga una partición especial llamada EFI System Partion (ESP). El ESP es una simple partición FAT16 o FAT32 con los flags de partición boot y esp. El ESP almacena archivos ejecutables EFI, a pesar que son de un tamaño inferior a 100MB, algunos sistemas operativos requieren que la partición tenga una capacidad de 500MB. Por lo tanto, se procede a crear una partición de 550MB de tipo primaria, fat32, con la etiqueta ESP y efi como el nombre de la partición. Se aplican los cambios y en la opción de `manage flags` se selecionan boot y esp.

El siguiente paso es crear dos particiones ext4 para los sistemas operativos Debian y ParrotOS. Por ejemplo, se puede asignar 250GB a Debian, 150GB a Parrot y el resto puede ser una partición de datos que sea compartida. A cada partición se recomienda asignar su correspondiente etiqueta.

## Instalación de Debian

Accedemos a la instalación en modo experto con gráfica y avanzamos hasta la detección de discos y particiones. Creamos las siguientes particiones:

- 500MB para una partición `EFI` común. 
- 500MB para la partición `boot` de Debian. 
- 500MB para la partición `boot` de ParrotOS. 
- El resto en una partición donde irán los distintos sistemas operativos.

Primero se crea un volumen cifrado en la partición con la etiqueta `all-Operative-Systems` indicando que no se formatee o se borre la partición. Después se crea un volumen LVM de grupo y los siguientes volúmenes lógicos:

### Volúmenes Debian:
- 8GB para SWAP.
- 250GB para root.
- 100GB para home.

### Volúmenes ParrotOS:
- 8GB para SWAP.
- 250GB para root.
- 100GB para home.

### Volúmenes datos comunes:
- El resto para datos comunes.

Se asigna a cada volumen lógico de Debian el punto de montaje correspondiente y se termina la instalación eligendo el entorno de escritorio al gusto del consumidor.

## Instalación de ParrotOS

Como ParrotOS es una distribución derivada de Debian, la instalación se realiza igual que en el apartado anterior. Se accede al modo experto de instalación gráfica y se avanza hasta el punto de detección de discos. Como la partición de los sistemas operativos está cifrada, es necesario descifrarla y detectar el grupo de volúmenes LVM. Para ello, saldremos de la sección de detección de discos y iremos a la sección de abrir una terminal o shell. Lo primero será desencriptar la partición cifrada con:

Nota: la partición /dev/sdaX debe corresponder a la cifrada y el nombre debe ser el asignado en la etiqueta.

```zsh
cryptsetup luksOpen /dev/sdaX all-Operative-Systems
```

Posteriormente detectaremos el grupo de los volúmenes LVM con:
```zsh
vgchange -a y
```

Nota: es probable que los siguientes pasos no funcionen a la primera y sea necesario completarlos con la siguiente sección. Por lo que se podría obviar el final de esta sección e instalar el grub directamente.

Una vez ejecutados los comandos anteriores, seguir con la instalación hasta la instalación del GRUB. Volvemos a abrir una terminal e identificamos el UUID de la partición cifrada con:

```zsh
blkid /dev/sdaX
```

Lo siguiente es modificar el archivo `/etc/crypttab`:

```zsh
nano /etc/crypttab
```

Y se añade el siguiente contenido, donde el UUID es el obtenido del comando `blkid`:

```
all-Operative-Systems UUID=524c1ad6-fabe-4f32-9bb0-c8db1286b262 none luks
```

Terminamos la instalación y reiniciamos el sistema operativo. Si todo funciona correctamente, ya estaría terminado. Lo más habitual es que al intentar arrancar el sistema operativo no pueda abrir la partición cifrada, ni los volúmenes cifrados. Entonces, abrirá una terminal de initramfs y tendremos que configurar los siguientes pasos.



### No abre la partición cifrada y salta una terminal de initramfs

En caso de obtener una terminal con initramfs, será necesario volver a repetir los pasos de descifrar la partición y abrir el grupo de volúmenes cifrados como se menciona anteriormente. 

Abrimos la partición cifrada con:

```
cryptsetup luksOpen /dev/sdaX all-Operative-Systems
```

Posteriormente detectaremos el grupo de los volúmenes LVM con:

```
vgchange -a y
```

Para arrancar el sistema, simplemente utilizamos el siguiente comando:

```
exit
```

Nos llevará al login y entraremos con las credenciales creadas en la instalación. Una vez iniciado el sistema operativo, abrimos una terminal y
detectamos el UUID de la partición cifrada. La X de sdaX corresponde al número de la partición cifrada, si no se conoce simplemente utilizar el comando `blkid`.

```
blkid /dev/sdaX
```

Editamos el archivo `/etc/crypttab` con nano:

```zsh
sudo nano /etc/crypttab
```

Añadimos lo siguiente:
```
all-Operative-Systems UUID=524c1ad6-fabe-4f32-9bb0-c8db1286b262 none luks
```

Una vez terminado se utiliza el siguiente comando para actualizar initramfs:

```
sudo update-initramfs -u
```

Reiniciamos el sistema operativo con:
```
sudo reboot
```

## Instalación rEFInd

La instalación de rEFInd se realiza con el siguiente comando:

```
sudo apt install refind
```

[^1]: [Wikipedia, Debian](https://es.wikipedia.org/wiki/Debian_GNU/Linux)
[^2]: [Wikipedia, Logical Volume Manager (Linux)](https://es.wikipedia.org/wiki/Logical_Volume_Manager_(Linux))
[^3]: [Wikipedia, LUKS (Linux Unified Key Setup)](https://es.wikipedia.org/wiki/LUKS)
