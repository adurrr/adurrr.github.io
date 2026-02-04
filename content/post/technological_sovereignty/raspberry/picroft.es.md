+++
author = "Adur"
title = "Instalación y configuración de Mycroft en una Raspberry mediante Picroft"
date = "2022-09-03"
description = "Instalación y configuración de Mycroft en una Raspberry mediante Picroft"
featured = true
tags = [
    "raspberry pi",
    "mycroft",
    "picroft"
]
categories = [
    "soberanía tecnológica"
]
series = ["Raspberry Pi"]
aliases = ["migrate-from-jekyl"]
thumbnail = "images/raspberry-pi.png"
toc = true
weight = 8
pre = "<b> </b>"
+++

En esta entrada se definen los pasos a seguir para instalar y configurar Picroft en una Raspberry Pi.

# Instalación de la imagen Picroft en una tarjeta SD o un usb

Mediante el siguiente comando se quema la imagen en una tarjeta SD o un USB.

```zsh
sudo dd bs=4M if=Picroft_v20.08_2020-09-07.img of=/dev/sda1 conv=fsync oflag=direct status=progress
```

Una vez terminado, se introduce en una raspberry y se enciende conectada a un monitor, teclado, ratón y micrófono y auriculares (USB o jack, lo que se prefiera).

# Configuración de Pycroft

Se configura al inicio y es probable que no se escuche el audio por el auricular o altavoz debido a que por defecto está configurado ALSA en vez de pulseaudio. Para configurarlo correctamente, seguimos los siguientes pasos<cite> [^1]</cite>: 
[^1]: [Picroft Setup Steps - Quick Setup - USB Mic/Headphones Output](https://community.mycroft.ai/t/picroft-setup-steps-quick-setup-usb-mic-headphones-output/10868)


Configuramos picroft con el asistente de comandos `mycroft-setup-wizard`:

1. Las opciones elegidas son:
Salida: Altavoces vía 3.5 o USB
Mic: Otro - No soportado

2. Lo más probable es que las pruebas fallen en esta etapa, esto es de esperar
Una vez que llegue a la mycroft-cli y registre su dispositivo, es el momento de aplicar algunas actualizaciones.

- a. Actualice la lista de paquetes
```
sudo apt update
```

- b. Actualice los paquetes
```
sudo apt upgrade -y
```

- c. Eliminar los paquetes no utilizados
```
sudo apt autoremove -y
```

- d. Reinicie
```
sudo reboot
```

3. Cuando las cosas vuelvan a arrancar, es el momento de hacer funcionar el audio. Se puede hacer esto a través de SSH.

- a. Revise los dispositivos de audio por defecto
```
pactl info
```

- b. Establecer el dispositivo de salida por defecto a la toma de auriculares
```
pactl set-default-sink alsa_output.platform-bcm2835_audio.analog-stereo
```

- c. Verificar que los dispositivos por defecto son los correctos
```
pactl info
```

- d. Indique a MyCroft que utilice el sistema de pulsos por defecto
```
mycroft-config set listener.device_name "pulse"
```
- e. Reinicie
```
sudo reboot
```

4. Actualice el archivo `/etc/mycroft/mycroft.conf`
- a. Actualice las siguientes líneas:
"play_wav_cmdline": "aplay -Dhw:0,0 %1",
"play_mp3_cmdline": "mpg123 -a hw:0,0 %1",

- b. Para que se vea así:
"play_wav_cmdline": "aplay %1",
"play_mp3_cmdline": "mpg123 %1",

5. Actualiza la habilidad de DuckDuckGo
```
msm install https://github.com/JarbasSkills/skill-ddg
```

6. Cuando el sistema vuelva a estar en línea, ejecuta una prueba de mycroft-mic
```
mycroft-mic-test
```

7. Instalar habilidades adicionales
```
mycroft-msm install fallback-aiml
mycroft-msm install skill-finished-booting
mycroft-msm install mycroft-spotify
```

8. Asegúrese de que MyCroft emita un pitido cuando escuche la palabra "wake" - Debería devolver "true"
```
mycroft-config set confirm_listening true
```

Se puede ver la configuración por defecto de Mycroft en [GitHub - MycroftAI/mycroft-core ](https://github.com/MycroftAI/mycroft-core/blob/dev/mycroft/configuration/mycroft.conf).

9. Instalar raspotify (todavía tengo que conseguir que funcione)
```
curl -sL https://dtcooper.github.io/raspotify/install.sh | sh
```

10. Paquetes opcionales
```
sudo apt install rustc libpulse-dev python3-venv libatlas-base-dev -y
```


