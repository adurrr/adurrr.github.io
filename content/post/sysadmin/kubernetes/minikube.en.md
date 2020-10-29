+++
author = "Adur"
title = "Getting Started with Minikube"
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
+++

## Local Kubernetes installation

There are several tools that can be used to deploy Kubernetes on one or many clusters. Some of them include:

+ [Minikube](https://kubernetes.io/docs/setup/learning-environment/minikube/)
+ [Kind](https://kubernetes.io/docs/setup/learning-environment/kind/)
+ [Docker Desktop](https://www.docker.com/products/docker-desktop)
+ [MicroK8s](https://microk8s.io/)
+ [K3S](https://k3s.io/)

**Minikube is the easiest** and preferred method for setting up Kubernetes locally. It is used to manage a **single-node cluster**, although there is already an experimental feature that supports multi-node clusters.

## Minikube

The `minikube` project is a **local Kubernetes cluster implementation for Linux, macOS, and Windows**. Its goal is to be the best tool for local Kubernetes application development<cite> [^1].</cite>
[^1]: [GitHub, minikube](https://github.com/kubernetes/minikube)

The first steps with minikube can be found in the [official documentation](https://minikube.sigs.k8s.io/docs/start/) and are as follows<cite> [^2]:</cite>
[^2]: [Minikube, Getting Started](https://minikube.sigs.k8s.io/docs/start/)

## Requirements

* 2 CPUs or more
* 2GB of RAM
* 20GB of disk space
* Internet connection
* Container or virtual machine manager, such as: Docker, Hyperkit, Hyper-V, KVM, Parallels, Podman, VirtualBox, or VMware.

## Installing minikube

For Linux there are three options:
* Binary package:
```bash
 curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
 sudo install minikube-linux-amd64 /usr/local/bin/minikube
```
* Debian package:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb
```
* RPM package:
```bash
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-latest.x86_64.rpm
sudo rpm -ivh minikube-latest.x86_64.rpm
```

To **start** minikube, run:
```bash
minikube start
```

To **stop** minikube safely, run:
```bash
minikube stop
```

## Installing Kubernetes

Kubernetes can be installed locally on virtual machines or directly on the operating system. Tools such as `Ansible` or `kubeadm` can be used to **automate the installation**.

The CLI tool `kubectl` can be used to **manage, deploy, and configure the resources and applications of the Minikube cluster**, and can be installed with the following commands:

```bash
sudo apt-get update && sudo apt-get install -y apt-transport-https gnupg2 curl
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
echo "deb https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee -a /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```

For details on **kubectl** commands, you can refer to the [kubectl book](https://kubectl.docs.kubernetes.io/), the [official Kubernetes documentation](https://kubernetes.io/search/?q=kubectl), or its [GitHub repository](https://github.com/kubernetes/kubectl).

A common step after installation is to configure and enable **kubectl** command autocompletion:

```bash
sudo apt install -y bash-completion
source /usr/share/bash-completion/bash-completion
source <(kubectl completion bash)
echo 'source <(kubectl completion bash)' >>~/.bashrc
```

Other relevant packages to install include:
* `kubeadm`: used for managing or automating the installation
* `kubelet`: an agent that runs on each node and communicates with the control plane components
* `kubernetes-cni`: allows configuring network elements

```bash
sudo apt-get install kubelet kubeadm kubernetes-cni
```
