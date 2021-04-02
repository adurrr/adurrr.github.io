+++
author = "Adur"
title = "Permissions in GNU/Linux"
date = "2021-04-02"
description = ""
featured = true
tags = [
    "GNU/Linux",
    "debian"
]
categories = [
    "Infrastructure",
    "Self-Hosting"
]
series = ["GNU/Linux"]
aliases = ["permisos"]
thumbnail = "images/gnu-linux.png"
toc = true
+++


## Create a group

- Create the `shared` group:
```bash
sudo groupadd shared
```

## Add a user
- Add user `devops` to the `shared` group:
```
sudo usermod -a -G shared devops
```
For the changes to take effect, you need to log out and log back in as the devops user. You can verify with the command:
```
groups
```

## Change the group of a directory and its files
- Change the group of the `wiki` directory to `shared`:
```
sudo chgrp -R shared wiki
```

# Add write permissions for a group on a directory and its files

Add write `w` permissions for the user group on the `wiki` directory.
```bash
sudo chmod -R g+w wiki
```
