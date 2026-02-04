+++
author = "Adur"
title = "Configuración de servidor DNS en una Raspbery Pi"
date = "2021-10-21"
description = ""
featured = true
tags = [
    "raspberry pi",
    "DNS",
    "redes"
]
categories = [
    "soberanía tecnológica",
    "administración de sistemas"
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/dns.png"
toc = true
weight = 25
+++

En esta entrada se definen los pasos a seguir para configurar un servidor DNS en una Raspberry Pi.

## Instalación de DNSMasq

```zsh
sudo apt install -y dnsmasq
```

## Configuración de dnsmasq

```zsh
sudo nano /etc/dnsmasq.conf
```

Buscamos (CTRL+W) y descomentamos las siguientes líneas eliminando el signo de almohadilla (#):

- **domain-needed** - Configuramos el servidor DNS para que no reenvíe los nombres sin un punto (.) o un nombre de dominio a los servidores upstream. Los nombres sin punto o dominio se quedan en la red local.
- **bogus-priv** - Impide que el servidor DNS reenvíe las consultas de búsqueda inversa del rango de IP local a los servidores DNS ascendentes. Esto previene la filtración de la red local a los servidores upstream.
- **no-resolv** - Deja de leer los servidores de nombres upstream del archivo `/etc/resolv.conf`, confiando en cambio en los de la configuración de DNSMasq.

Buscamos (CTRL+W) y eliminamos la siguiente línea:
```
#sever=/localnet/192.168.0.1
```
La sustitimos por las DNS de Cloudflare:

```
server=1.1.1.1

server=1.0.0.1
```

Este paso hace uso de los servidores DNS de Google para los servidores de nombres ascendentes.

Buscamos (CTRL+W) la siguiente línea:
```
#cache-size=150
```

Descomentamos y cambiamos el tamaño de la caché a 1000:
```
cache-size=1000
```

Aumentar el tamaño de la caché ahorra un mayor número de peticiones DNS a la caché de DNSMasq. El rendimiento de la red mejora porque el tiempo de búsqueda de DNS se reduce.

Guardamos el archivo con CTRL+X, luego presionamos Y y pulsamos Enter para guardar los cambios.

Reiniciamos DNSMasq para aplicar los cambios:
```
sudo systemctl restart dnsmasq
```

Comprobamos el estado del servidor DNS con:
```
sudo systemctl status dnsmasq
```

## Añadir una url

```zsh
sudo nano /etc/dnsmasq.d/example.conf
```
Añadimos la ip local asociada a un nombre de dominio, por ejemplo `home.localhost`.
```
address=home.localhost/192.168.1.130
```
## Probar el servidor DNS en una

```zsh
sudo apt install -y dnsutils
``` 