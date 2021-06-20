+++
author = "Adur"
title = "Comandos útiles de docker"
date = "2021-06-20"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "docker"
]
categories = [
    "devops",
    "cheatsheet"
]
series = ["GNU/Linux"]
aliases = ["server"]
thumbnail = "images/docker.png"
toc = true
+++


## Purgar todos los recursos de Docker

```zsh
docker system prune --all
docker system prune --volumes
```

## Eliminar todas las imágenes 

```zsh
docker rmi -f $(docker images -a -q)
```