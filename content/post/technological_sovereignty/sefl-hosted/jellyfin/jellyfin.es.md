+++
author = "Adur"
title = "Despliegue de un servidor multimedia Jellyfin"
date = "2021-12-26T12:40:37+01:00"
description = ""
featured = true
tags = [
    "raspberry pi",
    "software libre"    
]
categories = [
    "soberanía tecnológica",
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/raspberry-pi.png"
toc = true
+++

En esta entrada se definen los pasos a seguir para configurar un servidor multimedia en una Raspberry Pi y hacerla accesible desde internet. La instalación se realizará en contenedoes mediante `docker-comnpose`.

## ¿Qué es Jellyfin?

Jellyfin es la solución construida por voluntarios permite el control de contenidos multimedia. Transmite a cualquier dispositivo desde un servidor propio, sin ataduras. Tus medios, tu servidor, a tu manera. Es un sistema multimedia de software libre que permite controlar la gestión y el streaming de archivos multimedia. Es una alternativa a los sistemas propietarios Emby y Plex, para proporcionar medios desde un servidor dedicado a los dispositivos de los usuarios finales a través de múltiples aplicaciones.<cite> [^1]</cite>.
[^1]: [Documentación, Jellyfin](https://jellyfin.org/docs/))

## Requisitos

Instalar:

- Docker

- Docker-compose

## Instalación en docker-compose básica

En la documentación de Jellyfin, se recomienda utilizar la imagen de LinuxServer.io<cite> [^2]</cite> para la instalación en una raspberry pi. El archivo inicial `docker-compose.yml` es el siguiente: 
[^1]: [linuxserver/jellyfin, LinuxServer.io](https://hub.docker.com/r/linuxserver/jellyfin)

```yaml
---
version: "2.1"
services:
  jellyfin:
    image: lscr.io/linuxserver/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - JELLYFIN_PublishedServerUrl=192.168.0.5 #optional
    volumes:
      - /path/to/library:/config
      - /path/to/tvseries:/data/tvshows
      - /path/to/movies:/data/movies
      - /opt/vc/lib:/opt/vc/lib #optional
      - /media/pi/Cineteca:/data/movies/Cineteca
    ports:
      - 8096:8096
      - 8920:8920 #optional
      - 7359:7359/udp #optional
      - 1900:1900/udp #optional
    devices:
#      - /dev/dri:/dev/dri #optional
#      - /dev/vcsm:/dev/vcsm #optional
#      - /dev/vchiq:/dev/vchiq #optional
      - /dev/video10:/dev/video10 #optional
      - /dev/video11:/dev/video11 #optional
      - /dev/video12:/dev/video12 #optional
    restart: unless-stopped

```

Nota: Hemos añadido el volumen `/media/pi/Cineteca` porque es un disco duro externo que se monta en la raspberry pi con el comando:

```zsh
udisksctl mount -b /dev/sdb1
```

## Arrancar el docker-compose

Utilizamos el siguiente comando para levantar el escenario:

```zsh
docker-compose up
```

Una vez levantado, es accesible en la dirección `http://<ip-raspberry>:8096`.


## Instalación en docker-compose con traefik y certificados Let's Encrypt

Primero, añadimos la variable de entorno con el nombre del dominio:

```zsh
export $DOMINIO=dominioejemplo.org
```

En segundo lugar creamos un fichero `docker-compose.yml` y añadimos el siguiente contenido:

```yaml
version: "3.3"

services:

  traefik:
    image: "traefik:v2.5"
    container_name: "traefik"
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.tlschallenge=true"
      - "--certificatesresolvers.myresolver.acme.caserver=https://acme-v02.api.letsencrypt.org/directory"
      - "--certificatesresolvers.myresolver.acme.email=hello@email-example.org"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
    ports:
      - "443:443"
      - "8080:8080"
    volumes:
      - "./letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
#    extra_hosts:
#      - host.docker.internal:172.17.0.1 # Needed to avoid Bad Gateway.

  whoami:
    image: "traefik/whoami"
    container_name: "simple-service"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.whoami.rule=Host(`whoami.${DOMAIN}`)"
      - "traefik.http.routers.whoami.entrypoints=websecure"
      - "traefik.http.routers.whoami.tls.certresolver=myresolver"

  jellyfin:
    image: lscr.io/linuxserver/jellyfin
    container_name: jellyfin
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Europe/London
      - JELLYFIN_PublishedServerUrl=jellyfin.${DOMAIN} #optional
    volumes:
      - /path/to/library:/config
      - /path/to/tvseries:/data/tvshows
      - /path/to/movies:/data/movies
      - /opt/vc/lib:/opt/vc/lib #optional
    ports:
      - 8096:8096
      - 8920:8920 #optional
#      - 7359:7359/udp #optional
#      - 1900:1900/udp #optional
    devices:
#      - /dev/dri:/dev/dri #optional
#      - /dev/vcsm:/dev/vcsm #optional
#      - /dev/vchiq:/dev/vchiq #optional
      - /dev/video10:/dev/video10 #optional
      - /dev/video11:/dev/video11 #optional
      - /dev/video12:/dev/video12 #optional
    restart: unless-stopped
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.jellyfin.rule=Host(`jellyfin.${DOMAIN}`)"
      - "traefik.http.routers.jellyfin.entrypoints=websecure"
      - "traefik.http.routers.jellyfin.tls.certresolver=myresolver"

```

Una vez levantado, es accesible en la dirección `http://<ip-raspberry>:8096`.