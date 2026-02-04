+++
author = "Adur"
title = "Primeros pasos con Minikube"
date = "2020-10-29"
description = ""
featured = false
tags = [
    "minikube",
    "kubernetes",
    "devops"
]
categories = [
    "devops",
]
series = ["Kubernetes"]
aliases = [""]
thumbnail = "images/kubernetes.png"
toc = true
weight= 20
+++

## Instalación en local de Kubernetes

Existen varias herramientas que se pueden utilizar para desplegar Kubernetes en uno o muchos clusters. Algunos de ellos serían:

+ [Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/)
+ [Kind](https://kubernetes.io/docs/setup/learning-environment/kind/)
+ [Docker Desktop](https://www.docker.com/products/docker-desktop)
+ [MicroK8s](https://microk8s.io/)
+ [K3S](https://k3s.io/)

**Minikube es el método más fácil** y preferido para crear una configuración de Kubernetes de forma local. Se utiliza para administrar un **clúster de un solo nodo** aunque ya existe una funcionalidad experimental que es compatible con los clústeres de múltiples nodos.

## Minikube

El proyecto `minikube` es una **implementación local de clusters de Kubernetes para Linux, mac0S y Windows**. Su objetivo es ser la mejor herramienta para desarrollar localmente aplicaciones de Kubernetes<cite> [^1].</cite>
[^1]: [GitHub, minikube](https://github.com/kubernetes/minikube)

Los primeros pasos con minikube se pueden seguir en la [documentación oficial](https://minikube.sigs.k8s.io/docs/start/) y son los siguientes<cite> [^2]:</cite>
[^2]: [Minikube, Cómo empezar](https://minikube.sigs.k8s.io/docs/start/)

## Requisitos

* 2 CPUs o más 
* 2GB de memoria RAM
* 20GB de espacio en disco
* Conexión a internet
* Manejador de contenedores o máquinas virtuales, como: Docker, Hyperkit, Hyper-V, KVM, Parallels, Podman, VirtualBox, o VMWare.

## Instalación de minikube

Para Linux hay tres opciones:
* Paquete binario:
```bash
 curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
 sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
* Paquete Debian:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```
* Paquete RPM:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -ivh minikube-latest.x86_64.rpm
```

Para **empezar** con minikube ejecutar:
```bash
minikube start
```

Para **parar** de forma segura minikube ejecutar:
```bash
minikube stop
```

## Instalación de kubernetes

Kubernetes puede instalarse localmente en máquinas virtuales o directamente en el sistema operativo. Existen herramientas como `Ansible` o `kubeadm` para **automatizar la instalación**.

Con la herramienta CLI `kubectl` se pueden **administrar, desplegar y configurar los recursos y las aplicaciones del cluster de Minikube**, por lo tanto se instala con los siguientes comandos:

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2 curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

Para los dellates de los comandos de **kubectl** se puede buscar en el [libro de kubectl](https://kubectl.docs.kubernetes.io/), en la [documentación oficial de Kubernetes](https://kubernetes.io/search/?q=kubectl) o en su [repositorio github](https://github.com/kubernetes/kubectl). 

Un paso muy habitual después de la instalación, es configurar y habilitar la autocompletado de comandos de **kubectl**:

```bash
sudo apt install -y bash-completion
source /usr/share/bash-completion/bash-completion
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

Otros paquetes relevantes a instalar son los siguientes:
* `kubeadm`: sirve para administrar o automatizar la instalación
* `kubelet`: es un agente que se ejecuta en cada nodo y se comunica con los componentes del plano de control
* `kubernetes-cni`: permite configurar los elementos de red

```bash
sudo apt-get install kubelet kubeadm kubernetes-cni
```