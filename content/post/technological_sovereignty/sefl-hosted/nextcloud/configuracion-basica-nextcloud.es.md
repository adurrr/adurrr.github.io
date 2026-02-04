+++
author = "Adur"
title = "Configuración de tu propio servidor Nextcloud"
date = "2020-10-03"
description = "Configuración básica de tu propio servidor Nextcloud para comenzar a cacharrear."
featured = true
tags = [
    "herramientas",
    "nextcloud",
    "software libre",
]
categories = [
    "soberanía tecnológica",
]
series = ["Herramientas"]
aliases = [""]
thumbnail = "images/nextcloud.png"
+++

Se llevará a cabo en un servidor LAMP (Linux, Apache, MySQL/MariaDB, PHP). La distribución GNU/Linux puede ser cualquiera Ubuntu, Trisquel, Manjaro, etc. Se ha realizado la configuración con Nextcloud 19.0.0

## Configuración del cortafuegos

Utilizando el cortafuegos sencillo `ufw`, permitiremos el tráfico SSH (puerto 22) y web (puerto 80):

```
sudo apt install ufw
sudo ufw allow 22
sudo ufw allow 80
```

## Instalación de de Apache

Instalación del servidor Apache:
```
sudo apt install apache2
```

## Instalación de php
Instalación de php para poder servir el contenido dinámico:
- Si se está utilizando una distribución basada en debian, es necesario añadir un repositorio para acceder a la versión 7.3. Por defecto, se encuentra disponible la versión 7.
```
sudo add-apt-repository ppa:ondrej/php
sudo apt update
```
-  Instalación de la versión 7.3 y de los módulos requeridos:
```
sudo apt install php7.3
sudo apt install php7.3-ctype php7.3-curl php7.3-dom php7.3-gd php7.3-iconv php7.3-mbstring php7.3-posix php7.3-xmlreader php7.3-zip php7.3-bz2 php7.3-exif php7.3-intl
```
- Instalación del módulo para poder conectarse a la base de datos:
```
sudo apt install php7.3-mysql
```

## Instalación de la base de datos

```
sudo apt install mariadb-server
```

## Ejecución de un archivo de seguridad de MySql

Este script se utiliza para utilizar el servidor MariaDB en producción:


```
sudo mysql_secure_installation
```
Como es la primera vez que se ejecuta, introducir un intro y configurar una nueva contraseña de root. Además, confirmar con Y las siguientes opciones: 
- Eliminar usuario anónimo
- Deshabilitar conectarse como root remotamente
- Eliminar base de datos de prueba
- Recargar tabla de priviligios

## Configuración de base de datos

Entrar en el monitor de MariaDB:
```
sudo mysql
```
Creación de una nueva base de datos llamada `nextcloud_db` (aunque se puede nombrar de otra manera). Creación de un usuario llamado `nc_admin@localhost` con una contraseña segura (cambiar password por una de verdad). Para evitar errores se utiliza la regla GRANT ALL ON, se actualizan los privilegios, y se sale de la base de datos:

```
CREATE DATABASE nextcloud_db;
CREATE USER 'nc_admin'@'localhost' IDENTIFIED BY 'password'
GRANT ALL ON nextcloud_db.* TO 'nc_admin'@'localhost';
flush privileges;
exit;
```

## Descarga de Nextcloud

Se descarga el archivo y se descomprime en la carpeta `/var/www/`:

```
wget https://download.nextcloud.com/server/releases/nextcloud-19.0.0.zip
unzip /home/ubuntu/nextcloud-19.0.0.zip -d /var/www/
```


## Configuración de apache

Se crea un archivo de configuración:
```
sudo nano /etc/apache2/sites-available/nextcloud.conf
```
Se copia y se pega en su interior la configuración de apache (sacada de la documentación oficial):
```
Alias /nextcloud "/var/www/nextcloud/"

<Directory /var/www/nextcloud/>
  Require all granted
  AllowOverride All
  Options FollowSymLinks MultiViews

  <IfModule mod_dav.c>
    Dav off
  </IfModule>

</Directory>
```
Se activa la nueva configuración:
```
sudo a2ensite nextcloud.conf
systemctl reload apache2
```

Accediendo a `http://IP_MÁQUINA/nextcloud` se podrá visibilizar un error interno del servidor, esto se debe a la falta de permisos del usuario web en la carpeta nextcloud.

Se añaden los siguientes permisos:

- El usuario por defecto `www-data` es cualquier usuario que se conecte a través de la web.
```
sudo chown -R www-data:www-data /var/www/nextcloud/
sudo chmod -R 755 /var/www/nextcloud/
```

Volvemos a acceder a `http://IP_MÁQUINA/nextcloud` y rellenamos lo siguiente:
- Unas credenciales para el usuario administrador
- Añadir el nombre del usuario de la base de datos: nc_admin
- Su respectiva contraseña definida anteriormente
- Nombre de la base de datos: nextcloud_db

Terminamos la instalación.

## Configuración adicional recomendable

1. Cambiar el límite de memoria (memory_limit) php a 512M en el archivo siguiente:

```
sudo nano /etc/php/7.3/apache2/php.ini
systemctl reload apache2
```
2. Descargar módulos php recomendados:

```
sudo apt install php7.3-imagick php7.3-bcmath php7.3-gmp
```
Recargar el servicio apache2 para hacer efectivos los cambios:
```
systemctl reload apache2
```

## Enlaces útiles

1. [Taller básico de NextCloud con Apache - Rafael Mateus (LibreLabUCM)](https://invidious.snopyta.org/watch?v=tDkpdQvYkgU)

2. [Comandos utilizados](https://gitlab.librelabucm.org/rmateus/tallernextcloud/blob/master/README.md)

3. [Documentación oficial de Nextcloud](https://docs.nextcloud.com/server/19/admin_manual/installation/source_installation.html)