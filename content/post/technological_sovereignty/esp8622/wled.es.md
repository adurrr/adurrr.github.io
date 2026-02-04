+++
author = ""
title = "Instalación y configuración de WLED en ESP8266"
date = "2021-10-21"
description = ""
featured = true
tags = [
    "ESP8266",
    "WLED",
    "software libre"
]
categories = [
    "soberanía tecnológica",
]
series = ["ESP8266"]
aliases = [""]
thumbnail = "images/esp8266.png"
toc = true
weight = 15
pre = "<b> </b>"
+++

En esta entrada se definen los pasos a seguir para instalar y configurar WLED en una placa ESP8266.

## Flashear el ESP8266

NodeMCU es un firmware de código abierto basado en Lua para el SOC WiFi ESP8266 de Espressif y utiliza un sistema de archivos SPIFFS basado en la flash del módulo. NodeMCU está implementado en C y se basa en el SDK NON-OS de Espressif.

El firmware se desarrolló inicialmente como un proyecto complementario a los populares módulos para desarrollo NodeMCU basados en ESP8266, pero el proyecto está ahora apoyado por la comunidad, y el firmware puede ejecutarse en cualquier módulo ESP<cite> [^1]</cite>.
[^1]: [Documentación NodeMCU](https://nodemcu.readthedocs.io/en/latest)

Para flasear el ESP8266 intalams la herramienta [esptool](https://nodemcu.readthedocs.io/en/latest/getting-started/): 

```zsh
pip install esptool
```

Descargamos la última versión de WLED de su GitHub y flaseamos la placa con el siguiente comando:

```
esptool.py write_flash 0x0 ./WLED_0.13.1_ESP8266.bin
```

