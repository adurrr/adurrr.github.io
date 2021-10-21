+++
author = "Adur"
title = "OS and SSH Hardening"
date = "2021-10-21"
description = "OS and SSH Hardening"
featured = true
tags = [
    "raspberry pi",
    "ansible"
]
categories = [
    "Security",
    "devops",
    "Infrastructure"
]
series = ["Raspberry pi"]
aliases = [""]
thumbnail = "images/raspberry-pi.png"
toc = true
+++

This post outlines the steps to harden the OS and SSH on a Raspberry Pi, so you can later deploy a set of secured services.

### 1. SSH Configuration

The next step requires copying your public key to the `~/.ssh/authorized_keys` file on the Raspberry Pi. Use the following command:

```zsh
ssh-copy-id -i <identity.pub> pi@<raspberry ip or node-1>
```
It will prompt for your SSH key password. Then connect to the Raspberry Pi with:

```zsh
ssh pi@node-1
```

### 2. Installing Ansible

- Debian

```zsh
sudo apt install -y ansible
```

- Arch:

```zsh
sudo pacman -Sy ansible
```

### 3. OS and SSH Configuration Using Ansible

#### 3.1. Using a collection

- Installation:

```zsh
ansible-galaxy install dev-sec.os-hardening
ansible-galaxy install dev-sec.ssh-hardening
```

- Create a playbook for each Ansible role named `ansible-os-hardening.yaml` and `ansible-ssh-hardening.yaml`.

- Run these playbooks with the following commands:
```zsh
ansible-playbook ansible-os-hardening.yaml --ask-become-pass
ansible-playbook ansible-ssh-hardening.yaml --ask-become-pass
```

#### 3.2. Using a basic playbook

- Add your SSH key to an ssh-agent using zsh (or bash):
```
ssh-agent zsh
ssh-add ~/.ssh/id_ed25519
```
- Run the Ansible playbook with the sudo password required for the commands:
```
ansible-playbook playbook.yaml --ask-become-pass
```
