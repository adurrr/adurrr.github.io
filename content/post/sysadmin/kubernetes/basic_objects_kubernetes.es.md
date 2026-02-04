+++
author = "Adur"
title = "Objetos básicos de Kubernetes"
date = "2020-10-29"
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

weight= 10
+++

En esta entrada se realizará una descripción de los objetos básicos de Kubernetes. Algunos de los ejemplos de los objetos de Kubernetes son los Pods, ReplicaSets, Deployments, Namespaces, etc. 

## Objetos básicos

Según la [documentación de Kubernetes](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/) los **objetos** de Kubernetes son **entidades persistentes**. Kubernetes usa estas entidades para representar el **estado del clúster**. Cada objeto de Kubernetes está representado por un recurso RESTful y existe en una ruta HTTP única. Específicamente, **los objetos pueden describir**<cite> [^1]:</cite>
[^1]: [Kubernetes Documentation, Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

+ Qué **aplicaciones en contenedores se están ejecutando (y en qué nodos)**.
+ Los **recursos disponibles** para esas aplicaciones.
+ Las **políticas de comportamiento de esas aplicaciones**, como políticas de reinicio, actualizaciones y tolerancia a fallas.

Casi todos los objetos de Kubernetes incluyen dos campos del objeto que lo configuran: el `spec` o especificación del **estado deseado objeto** y el `status` o **estado real o actual del objeto**. En la sección **`spec`** se declara la intención o **estado deseado** del objeto. El **plano de control** es quién se encarga de intentar **hacer coincidir el estado real del objeto con el estado deseado**.

Para crear un objeto es necesario que la API de Kubernetes reciba la información del mismo en formato JSON. Aunque la mayoría de las veces se envía la información mediante `kubectl` en un archivo `.yaml` que será convertido a formato JSON. Un ejemplo de archivo `.yaml` es el siguiente<cite> [^1]:</cite>
```yaml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
```

### Campos requeridos

Se necesitan establecer los siguientes campos obligatorios para crear un objeto<cite> [^1]:</cite>
+ `apiVersion`: qué **versión de la API de Kubernetes** se está utilizando para crear el objeto.
+ `kind`: qué **tipo de objeto** se quiere crear.
+ `metadata`: datos que sirven para **identificar de forma única el objeto**, incluyendo un nombre, un UID y un `namespace` opcional.
+ `spec`: el **estado deseado** para el objeto.

## Etiquetado (labels)

Las etiquetas o `labels` son **pares clave-valor adjuntos a los objetos** de Kubernetes. Se utilizan para **organizar y seleccionar subconjuntos de objetos**, en función de unos requisitos preestablecidos. Muchos objetos pueden tener la misma etiqueta, por lo que no proporcionan unicidad a los objetos<cite> [^2].</cite>
[^2]: [Kubernetes Documentation, Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

Por ejemplo, para agregar la etiqueta `color=verde` a un Pod llamado planta, se puede ejecutar lo siguiente<cite> [^3]:</cite>
[^3]: [O’Reilly, Kubernetes: Up and Running](https://www.oreilly.com/library/view/kubernetes-up-and/9781492046523/)
```bash
kubectl label pods planta color=verde
```
El comando anterior no sobreescribirá una etiqueta existente, por lo tanto, es necesario utilizar le flag `--overwrite`. O en caso de querer eliminar la etiqueta color, se hará con el comando a continuación:
```bash
kubectl label pods bar color -
``` 

### Selectores de etiquetas (label selectors)

Los controladores utilizan selectores de etiquetas para seleccionar un subconjunto de objetos. Kubernetes admite dos tipos de selectores<cite> [^4]:</cite>
[^4]: [Kubernetes Documentation, Label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)

#### Selectores basados en la igualdad
Permiten el filtrado de objetos en función de claves y valores de etiqueta. La coincidencia se consigue utilizando los operadores:
+ Si es igual `=` o `==` (no existe diferencia entre los operadores)
+ Si es distinto !=

#### Selectores basados en conjuntos
Permiten el filtrado objetos en función de un conjunto de valores. Podemos usar operadores `in`, `notin` para valores de etiqueta, y el operador `exists` para claves de etiquetas. 

## Tipos de Objetos

### Pods
Un pod es la unidad de programación más pequeña de Kubernetes. Es una colección lógica de uno o más contenedores que<cite> [^2]:</cite>
[^2]: [edX, Introduction to Kubernetes - The Linux Foundation](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

+ Están programados en el mismo host.
+ Comparten el mismo `network namespace`, por lo tanto comparten una única dirección IP asignada al Pod.
+ Tienen acceso para montar el mismo almacenamiento externo (volúmenes).

Los pods tienen naturaleza efímera y no tienene la capacidad de autorreparación. Por eso se utilizan controladores para gestionar la replicación de Pods, tolerancia a fallos, autorreparación, etc. Algunos de estos controladores son Deployments, ReplicaSets, etc.

### ReplicaSets

### Deployments







