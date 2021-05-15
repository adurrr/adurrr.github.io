+++
author = "Adur"
title = "Useful Docker commands"
date = "2021-05-15"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "docker"
]
categories = [
    "Infrastructure",
    "devops",
]
series = ["GNU/Linux"]
aliases = ["server"]
thumbnail = "images/docker.png"
toc = true
+++


## Purge all Docker resources

```zsh
docker system prune --all
docker system prune --volumes
```

## Remove all images

```zsh
docker rmi -f $(docker images -a -q)
```
