---
title: "Python basics"
date: 2021-02-17T09:13:29+01:00
draft: false
weight: 15
tags:
  - "Python"

categories:
  - "development"
---

## Requirements

### Pipenv
```
# Install
pip install pipenv

# Install a packages for the project
pipenv install <package>

# Activate Virtual Env
pipenv shell

# Run a script in the virtual env
pipenv run python <script.py>
```

Generate `requirements.txt`:
```bash
pipenv lock -r >> requirements.txt
```