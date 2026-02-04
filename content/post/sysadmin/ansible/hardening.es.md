+++
author = "Adur"
title = "Hardening de OS y SSH"
date = "2021-10-21"
description = "Hardening de OS y SSH"
featured = true
tags = [
    "raspberry pi",
    "ansible"
]
categories = [
    "seguridad",
    "devops",
    "administración de sistemas"
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/raspberry-pi.png"
toc = true
weight = 15
+++

En esta entrada se definen los pasos a seguir para hacer hardening de OS y SSH una Raspberry Pi y posteriormente poder desplegar una serie de servicios securizados.

### 1. Configuración SSH

En el siguiente paso es necesario copiar la clave pública al archivo `~/.ssh/authorized_keys` de la RaspberryPi. Para ello se utilizará el siguiente comando:

```zsh 
ssh-copy-id -i <identity.pub> pi@<ip de la raspberry o node-1>
```
Ahora nos pedirá la contraseña de nuestra clave SSH y nos conectaremos a la RaspberryPi mediante:

```zsh
ssh pi@node-1
``` 

### 2. Instalación de Ansible

- Debian

```zsh
sudo apt install -y ansible
``` 

- Arch:

```zsh
sudo pacman -Sy ansible
```

### 3. Configuración OS y SSH usando Ansible

#### 3.1. Usando una colección

- Instalación:

```zsh
ansible-galaxy install dev-sec.os-hardening
ansible-galaxy install dev-sec.ssh-hardening
```

- Crea un playbook para cada rol de ansible llamado `ansible-os-hardening.yaml` y `ansible-ssh-hardening.yaml`.

- Ejecuta estos playbooks con los siguientes comandos:
```zsh
ansible-playbook ansible-os-hardening.yaml --ask-become-pass
ansible-playbook ansible-ssh-hardening.yaml --ask-become-pass
```

#### 3.2. Usando un playbook básico

- Añade tu clave ssh en un ssh-agent usando zsh (o bash):
```
ssh-agent zsh
ssh-add ~/.ssh/id_ed25519
```
- Ejecutar el playbook de ansible con la contraseña sudo requerida para los comandos:
```
ansible-playbook playbook.yaml --ask-become-pass
```
