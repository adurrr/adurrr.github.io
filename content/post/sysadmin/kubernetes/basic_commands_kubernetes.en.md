+++
author = "Adur"
title = "Basic Kubernetes Commands"
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

This post provides an overview of the basic `kubectl` CLI commands that can be applied to Kubernetes objects. Some examples of Kubernetes objects include Pods, ReplicaSets, Deployments, Namespaces, etc.

## Namespaces

`Namespaces` are used in Kubernetes to organize cluster objects. Essentially, a `namespace` **represents a folder containing a set of objects**. By default, `kubectl` interacts with the `default` namespace. To use a different namespace, the `--namespace` flag is required, for example `--namespace=example`. To interact with all namespaces, use the `--all-namespaces` flag<cite> [^1].</cite>
[^1]: [O'Reilly, Kubernetes: Up and Running](https://www.oreilly.com/library/view/kubernetes-up-and/9781492046523/)

### Contexts

If you want to change the default namespace permanently, you can use a `context`. When used, it is recorded in the kubectl configuration file, stored at `HOME/.kube/config`. To create a context with a new default namespace name, run<cite> [^1]:</cite>

```bash
kubectl config set-context my-context --namespace=nuevonamespacepordefecto
```
```bash
kubectl config use-context my-context
```

## Kubernetes API objects

**Every Kubernetes object is represented by a RESTful resource and exists at a unique HTTP path in the Kubernetes API**. Resources are represented as **JSON or YAML files**. Through the `kubectl` command, you can access these objects. For example, using `kubectl get` you can access any resource in the default namespace<cite> [^1]:</cite>
```bash
kubectl get <resource-name>
```
To get a more specific resource:
```bash
kubectl get <resource-name> <object-name>
```
To get more information about the object in JSON or YAML format, you can add the `-o json` or `-o yaml` flags respectively. This output is not very human-readable.

Another option to get human-readable details about an object is to use the `kubectl describe` command:

```bash
kubectl describe <resource-name> <object-name>
```

## Creating, updating, or deleting Kubernetes objects

As mentioned earlier, Kubernetes objects or resources are represented by JSON or YAML files. To create, update, or delete these objects, such files are used. For example, to create or update an object stored in `ejemplo.yaml`, run:
```bash
kubectl apply -f ejemplo.yaml
```

If you prefer to make interactive edits instead of modifying the local file, you can use the `kubectl edit` command to download the latest version and launch an editor. After saving the file, it will be uploaded and automatically updated.
```bash
kubectl edit <resource-name> <object-name>
```
The `kubectl apply` command also saves the version history of configuration files. You can access these records using the `edit-last-applied`, `set-last-applied`, and `view-last-applied` options.
```bash
kubectl apply -f myobj.yaml view-last-applied
```

To delete an object, simply run:
```bash
kubectl delete -f ejemplo.yaml
```

## Debugging

`kubectl` also has a set of commands for debugging your containers. To view the logs of a running container, run:
```bash
kubectl logs <pod-name>
```
If there are multiple containers in the pod, you can choose the container to inspect with the `-c` flag.

By default, `kubectl logs` lists the current logs and exits. If you want to continuously stream the logs to the terminal instead, you can add the `-f` (follow) flag to the command line.

You can also use the `exec` command to run a command in a running container:
```bash
kubectl exec -it <pod-name> -- bash
```
This will provide an interactive console inside the running container for more detailed debugging.
