+++
author = "Adur"
title = "Instalación de LineageOS"
date = "2021-10-21"
description = ""
featured = true
tags = [
    "lineageOS",
    "android"
]
categories = [
	"administración de sistemas",
    "soberanía tecnológica",
]
series = ["F-Droid"]
aliases = [""]
thumbnail = "images/f-droid.png"
toc = true
draft = false
+++

## ¿Qué es LineageOS?

## Instalación de LineageOS en Xiami Mi A1

### Requerimientos

Material utilizado en el escenario:
- **Xiaomi Mi A1**
- **LineageOS16 oficial (nightly=en desarrollo)**
- **[LineageOS17 Unofficial](https://sourceforge.net/projects/test-tissot/files/Lineage/)**
- [TWRP Recovery](https://eu.dl.twrp.me/tissot/)

### Descargas

Descargar e instalar los archivos de desarrollo android(SDK platform) adb y fastboot(en mi caso: minimal_adb_fastboot_v1.4.3).

### Activar EOEM

Se activa el EOEM y la depuración USB en los ajustes del teléfono.

### Desbloqueo del bootloader

Se utilizan los siguientes comandos para desbloaquear bootloader. Se ejecutan en la carpeta de abd y fastboot:
```zsh
adb devices
# Se reinicia el móvil en modo fastboot
adb reboot bootloader
# Se comprueba el estado del dispositivo
fastboot oem device-info
# Se desbloquea el bootloader 
fastboot oem unlock
#Se reinicia el móvil 
```
Se comprueba que se ha desbloqueado correctamente con:
```zsh
adb devices
adb reboot bootloader
fastboot oem device-info
```

### Instalación del modo recovery

Después de haber activado modo depuración USB y desbloqueado la eoem,
se instala el modo recovery con el archivo .img en la misma carpeta que abd y fastboot:

```zsh
fastboot flash recovery twrp-installer-3.3.1-0-tissot.img
```

Se obtiene el siguiente error:

```
	target reported max download size of 534773760 bytes
	sending 'recovery' (29348 KB)...
	OKAY [  0.683s]
	writing 'recovery'...
	FAILED (remote: partition table doesn't exist)
	finished. total time: 0.699s
```

Entonces descubrimos que se soluciona con:

```zsh
fastboot flash boot twrp-3.3.1-0-tissot.img
```

Obtenemos el siguiente resultado:
```
RESULTADO:
	fastboot flash boot twrp-3.3.1-0-tissot.img
	target reported max download size of 534773760 bytes
	sending 'boot' (29348 KB)...
	OKAY [  0.679s]
	writing 'boot'...
	OKAY [  0.181s]
	finished. total time: 0.860s
```

### Instalación de la imagen

Se reinicia en modo recovery con `botón power + vol arriba` y se siguen los siguientes pasos:

1. Wipe-> Format Data
2. Advanced Wipe-> Dalvikk, system y data
3. Install -> lineage-1-...-tissot.zip
4. Wipe Dalvik t reboot sistem.

Una vez seguidos, ya está instalado LineageOS. Por defecto no viene con Google Play instalado. En caso de dar algún error, se pueden instalar las GApps en su versión mínima, por ejemplo 	     `open_gapps-arm-9.0-pico-20200408.zip`.