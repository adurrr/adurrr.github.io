+++
author = "Adur"
title = "Basic Kubernetes Objects"
date = "2020-10-29"
description = ""
featured = false
tags = [
    "theory",
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

This post provides a description of the basic Kubernetes objects. Some examples of Kubernetes objects include Pods, ReplicaSets, Deployments, Namespaces, etc.

## Basic objects

According to the [Kubernetes documentation](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/), Kubernetes **objects** are **persistent entities**. Kubernetes uses these entities to represent the **state of the cluster**. Each Kubernetes object is represented by a RESTful resource and exists at a unique HTTP path. Specifically, **objects can describe**<cite> [^1]:</cite>
[^1]: [Kubernetes Documentation, Understanding Kubernetes Objects](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

+ Which **containerized applications are running (and on which nodes)**.
+ The **resources available** to those applications.
+ The **behavioral policies for those applications**, such as restart policies, upgrades, and fault tolerance.

Almost all Kubernetes objects include two fields that configure them: the `spec` or **desired state of the object** specification, and the `status` or **actual/current state of the object**. In the **`spec`** section, the intent or **desired state** of the object is declared. The **control plane** is responsible for attempting to **match the actual state of the object with the desired state**.

To create an object, the Kubernetes API must receive the object information in JSON format. Most of the time, the information is sent through `kubectl` in a `.yaml` file that will be converted to JSON format. An example of a `.yaml` file is as follows<cite> [^1]:</cite>
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

### Required fields

The following mandatory fields must be set to create an object<cite> [^1]:</cite>
+ `apiVersion`: which **version of the Kubernetes API** is being used to create the object.
+ `kind`: what **type of object** you want to create.
+ `metadata`: data that serves to **uniquely identify the object**, including a name, a UID, and an optional `namespace`.
+ `spec`: the **desired state** for the object.

## Labels

Labels are **key-value pairs attached to Kubernetes objects**. They are used to **organize and select subsets of objects** based on predefined requirements. Many objects can share the same label, so labels do not provide uniqueness to objects<cite> [^2].</cite>
[^2]: [Kubernetes Documentation, Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

For example, to add the label `color=green` to a Pod called plant, you can run the following<cite> [^3]:</cite>
[^3]: [O'Reilly, Kubernetes: Up and Running](https://www.oreilly.com/library/view/kubernetes-up-and/9781492046523/)
```bash
kubectl label pods planta color=verde
```
The above command will not overwrite an existing label, so you need to use the `--overwrite` flag. Or if you want to remove the color label, use the following command:
```bash
kubectl label pods bar color -
```

### Label selectors

Controllers use label selectors to select a subset of objects. Kubernetes supports two types of selectors<cite> [^4]:</cite>
[^4]: [Kubernetes Documentation, Label selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)

#### Equality-based selectors
Allow filtering objects based on label keys and values. Matching is achieved using the operators:
+ Equals `=` or `==` (there is no difference between the operators)
+ Not equals `!=`

#### Set-based selectors
Allow filtering objects based on a set of values. You can use `in`, `notin` operators for label values, and the `exists` operator for label keys.

## Object types

### Pods
A pod is the smallest scheduling unit in Kubernetes. It is a logical collection of one or more containers that<cite> [^2]:</cite>
[^2]: [edX, Introduction to Kubernetes - The Linux Foundation](https://kubernetes.io/docs/concepts/overview/working-with-objects/kubernetes-objects/)

+ Are scheduled on the same host.
+ Share the same `network namespace`, and therefore share a single IP address assigned to the Pod.
+ Have access to mount the same external storage (volumes).

Pods are ephemeral in nature and do not have self-healing capabilities. That is why controllers are used to manage Pod replication, fault tolerance, self-healing, etc. Some of these controllers include Deployments, ReplicaSets, etc.

### ReplicaSets

### Deployments
