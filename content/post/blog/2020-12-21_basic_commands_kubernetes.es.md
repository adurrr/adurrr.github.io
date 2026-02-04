+++
author = "Adur"
title = "Comandos básicos de Kubernetes"
date = "2020-12-21"
description = ""
featured = false
tags = [
    "teoría",
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

En esta entrada se realizará un repaso de los comandos básicos del cli `kubectl` que se pueden aplicar a los objetos de Kubernetes. Algunos de los ejemplos de los objetos de Kubernetes son los Pods, ReplicaSets, Deployments, Namespaces, etc.

## Namespaces

Se utilizan `namespaces` en Kubernetes para organizar los objetos del clúster. A fin de cuentas, un `namespace` **representa una carpeta que contiene un conjunto de objetos**. Por defecto, `kubectl` interactúa con el namespace `default`. Para usar otro tipo de de namespace es necesario utilizar el flag `--namespace`, por ejemplo `--namespace=ejemplo`. Si se quiere interacturar con todo el conjunto de namespaces, se utiliza el flag `--all-namespace`<cite> [^1].</cite>
[^1]: [O’Reilly, Kubernetes: Up and Running](https://www.oreilly.com/library/view/kubernetes-up-and/9781492046523/)

### Contexts

En caso de querer cambiar el namespace predreterminado de forma permanente, se puede utilizar un `context`. Al utilizarlo se registra en el archivo de configuración de kubectl, almacenado en `HOME/.kube/config`. Para crear un context con un nuevo nobre predeterminado de namespace, ejecutar<cite> [^1]:</cite>

```bash
kubectl config set-context my-context --namespace=nuevonamespacepordefecto
```
```bash
kubectl config use-context my-context
```

## Objetos de la API de Kubernetes

**Cada objeto de Kubernetes está representado por un recurso RESTful y existe en una ruta HTTP única a la API de Kubernetes**. Los recursos se representan como **archivos JSON o YAML**. A través del comando `kubectl`, se podrá acceder a dichos objetos. Por ejemplo, mediante el comando `kubectl get` se puede acceder a cualquier recurso del namespace por defecto<cite> [^1]:</cite>
```bash
kubectl get <nombre-recurso>
``` 
En caso de querer un recurso más específico:
```bash
kubectl get <nombre-recurso> <nombre-obj>
``` 
En caso de querer obtener más información del objeto en formato JSON o YAML, se pueden agregar los flags `-o json` o `-o yaml` respectivamente. Esta información no es tan legible para los humanos.

Otra opción para obtener más detalles del objeto legible para humanos es usar el comando `kubectl describe`: 

```bash
kubectl describe <nombre-recurso> <nombre-obj>
``` 

## Creación, actualización o destrucción de objetos de Kubernetes

Como se ha mencionado anteriormente, los objetos o recursos de Kubernetes se representan mediante archivos JSON o YAML. Para crear, actualizar o eliminar dichos objetos se utilizan ese tipo de archivos. Por ejemplo, en caso de querer crear o actualizar un objeto almacenado en `ejemplo.yaml` se ejecuta:
```bash
kubectl apply -f ejemplo.yaml
```

En caso de desear realizar ediciones intectivas en lugar del archivo local, se puede utilizar el comando `kubectl edit` para descargar la versión más reciente y lanza un editor. Después de guardar el archivo se subirá el archivo y se actualizará automáticamente.
```bash
kubectl edit <nombre-recurso> <nombre-obj>
```
El comando `kubectl apply` también guardará el historial de versiones de los archivos de configuración. Se pueden acceder a estos registros mediante las opciones `edit-last-applied`, `set-last-applied`, y `view-last-applied`.
```bash
kubectl apply -f myobj.yaml view-last-applied
```

Para eliminar un objeto, simplemente se necesita ejecutar:
```bash
kubectl delete -f ejemplo.yaml
``` 

## Depuración

`kubectl` también  tiene una serie de comandos para depurar sus contenedores. Para ver los registros de un contenedor en ejecución, se ejecuta: 
```bash
kubectl logs <nombre-pod> 
``` 
Si existen varios contenedores en el pod, se puede elegir el contenedor que desea inspeccionar con el flag `-c`. 

De forma predeterminada, `kubectl logs` enumera los registros y salidas actuales. Si, en cambio, se quiere transmitir continuamente los registros a la terminal, se puede agregar el flag `-f` (follow) en la línea de comandos. 

También se puede usar el comando `exec` para ejecutar un comando en un contenedor en ejecución: 
```bash
kubectl exec -it <pod-name> -- bash 
```
Esto proporcionará una consola interactiva dentro del contenedor en ejecución para realizar una depuración más detallada.

