+++
author = "Adur"
title = "Primeros pasos con una Raspberry Pi y Ubuntu server"
date = "2020-09-28"
description = "Instalación del SO, conexión WiFi y ssh"
featured = true
tags = [
    "raspberry pi",
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

En esta entrada se definen los primeros pasos a seguir con una Raspberry Pi sin necesidad de una pantalla o un cable HDMI. Es una configuración rápida y básica para poder empezar a cacharrear con el terminal.

## 1.  Instalación del SO 
Usando el programa Raspberry Pi Imager se puede quemar la imagen de Ubuntu, RaspberryPiOS, LibreElec o que prefieras, en cualquier tarjeta micro SD. <cite> [^1]</cite>
[^1]: [Instalación de Ubuntu en la Raspberry - Atareao](https://www.atareao.es/podcast/ubuntu-en-la-raspberry/)

## 2. Conexión WiFi

Si prefieres no utilizar el cable HDMI para la configuración de la red, únicamente con modificar el archivo `network-config` con la siguiente configuración:

``` 
wifis:
  wlan0:
  dhcp4: true
  optional: true
  access-points:
    "SSID":
      password: "contraseña"
```
El SSID es el nombre de la red WiFi y no hace falta comillas, mientras que para la contraseña si que se necesitan comillas.

## 3. Primera conexión

Lo más sencillo es conectarse a través de SSH. Para ello es necesario saber cuál es la dirección IP que se asigna automáticamente mediante DHCP a la Raspberry. Se puede conocer a través de la interfaz web del router o realizando un escaneo de red con nmap. 

La conexión se realiza mediante el siguiente comando:

```
ssh ubuntu@IP_máquina
```

El usuario y contraseña por defecto son `ubuntu`

## 4. Seguridad SSH

### 4.1. Configuración de SSH mediante llaves

La diferencia de conectarse por SSH a un dispositivo mediante una contraseña o un par de llaves RSA es que la contraseña la puede conocer cualquiera y las llaves son personales e intransferibles<cite> [^2]</cite>. Por eso para generar un par de claves público-privada será a través del siguiente comando:

[^2]: [La_bekka, Accedamos remotamente a la Raspberry Pi usando SSH - Parte II: ¡Hola maquinita!](https://labekka.red/servidoras-feministas/2019/10/09/fanzine-parte-2.html#ssh) 
```bash
ssh-keygen -t rsa
```
Nos pedirá una contraseña para usar la clave SSH y se crearán dos claves en el diretorio `.ssh` en el home del usuario. La clave privada será `id_rsa`, la cuál no habrá que compartirla nunca, y la pública `id_rsa.pub`. 

En el siguiente paso es necesario copiar la clave pública al archivo `~/.ssh/authorized_keys` de la RaspberryPi. Para ello se utilizará el siguiente comando:

```bash 
ssh-copy-id -i ubuntu@<ip de la raspberry>
```
Ahora nos pedirá la contraseña de nuestra clave SSH y se conectará a la RaspberryPi.

Lo último que haremos en este apartado será restringir el acceso remoto con contraseñas. De esa manera sólo podrán conectarse remotamente aquellas personas que hayan subido previamente su llave SSH pública a la Raspberry Pi. Lo que tendremos que hacer es editar el archivo de configuración ` `sshd_config`. Ejecutamos el siguiente comando en nuestra Raspberry Pi:
```bash
sudo nano /etc/ssh/sshd_config
```
Y en la línea `PasswordAuthentication` poner `no`:
```
PasswordAuthentication no
``` 
Guardamos los cambios y reiniciamos el servicio con:
```bash
sudo service ssh restart
```

### 4.2. Cambio del puerto de las conexiones SSH

Una buena práctica de seguridad es cambiar el puerto por defecto para las conexiones SSH (el puerto 22), ya que si intentan atacarnos probablemente prueben a conectarse por ese puerto. De esta manera, cambiando el puerto se reducirá considerablemente el numero de intentos de conexión por parte de posibles atacantes. Pueden elegir un número desde el 1024 hasta el 65535 si quieren. Pero primero debemos revisar que no haya otro servicio utilizando ese mismo puerto. Escribimos en la terminal:
```bash
netstat -punta | grep LISTEN
```
Una vez comprobado que el puerto que elegimos está libre deben modificar el archivo de configuración de SSH:
```bash
sudo nano /etc/ssh/sshd_config
```
Cambien la línea `#Port 22` por, por ejemplo, `Port 2251` u otro puerto que prefieran y que no esté ocupado. Es importante que borren la almohadilla/gato/numeral porque si no el sistema lo interpreta como un comentario.

Guarden, salgan y reinicien el servicio para que se haga efectivo el cambio en la configuración:
```bash
sudo service ssh restart
```
Cuidado, si hay un error en el cambio de puerto puede que se queden fuera de la servidora y pierdan la conexión por SSH que tenían. Es mejor si están físicamente cerca de la servidora en este paso.

A partir de ahora para conectarse a la Raspberry Pi via SSH deberán especificar el puerto, porque ya no será el puerto por defecto. Así que el comando que deberán escribir de ahora en más será:
```bash
ssh -p 2251 ubuntu@<dirección ip>
```
Si se se les olvida el puerto que pusieron o no saben si está abierto, pueden hacer un escaneo de puertos desde otra computadora con nmap así:
```bash
sudo nmap -p 1-65535 -T4 -A -v <ip-servidora>
```
Nota: apartado copiado y reproducido de [La_bekka, Seguridad de SSH - Parte IV: Securicemos nuestra servidora web](https://labekka.red/servidoras-feministas/2019/10/09/fanzine-parte-4.html#ssh)<cite> [^2]</cite>.
[^2]: [Seguridad de SSH - Parte IV Securicemos nuestra servidora web | la_bekka](https://labekka.red/servidoras-feministas/2019/10/09/fanzine-parte-4.html#ssh).