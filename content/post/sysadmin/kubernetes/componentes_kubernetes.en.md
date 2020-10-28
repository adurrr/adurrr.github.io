+++
author = "Adur"
title = "Kubernetes Architecture Components"
date = "2020-10-28"
description = "Basic theory to get started with Kubernetes."
featured = true
tags = [
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
shareImage = "images/kubernetes.png"
+++

This post covers the components of the Kubernetes architecture. The main sources of information are the **[Introduction to Kubernetes](https://www.edx.org/es/course/introduction-to-kubernetes)** course by **The Linux Foundation** on edX, authored by Chris Pokorni and Neependra Khare, and the official [Kubernetes documentation](https://kubernetes.io/docs/home/).

## Components of the Kubernetes architecture

A Kubernetes cluster consists of a set of worker machines, called nodes, that run containerized applications. Every cluster has at least one worker node. At a very high level of abstraction, Kubernetes has the following main components:

* One or more **master nodes**, on the control plane side.
* One or more **worker nodes**.

The following figure shows the **architecture of the components of a Kubernetes cluster**<cite> [^1]:</cite>
[^1]: [Documentation Kubernetes, Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

<!-- ![Figure 1: Kubernetes architecture components.](../img/arch-kubernetes.svg) -->

The **master node** provides a runtime environment for the control plane responsible for **managing the state of a Kubernetes cluster and is the brain behind all operations** within the cluster. The **control plane components** are agents with **very distinct roles in cluster management**. To **communicate** with the Kubernetes cluster, users **send requests to the control plane through** a **command-line interface** (CLI) tool, a **web user interface** dashboard, or an **application programming interface** (API)<cite> [^2]</cite>.
[^2]: [edX, Introduction to kubernetes - The Linux Foundation](https://www.edx.org/es/course/introduction-to-kubernetes)

It is essential to keep the control plane running at all costs. **Losing the control plane can cause downtime, resulting in service disruption** to clients, with a potential loss of business. To **ensure fault tolerance of the control plane**, **master node replicas** can be added to the cluster, configured in **high availability mode**. While only one of the master nodes is dedicated to actively managing the cluster, the control plane components remain synchronized across the master node replicas. This type of configuration adds resilience to the cluster's control plane, in case the active master node fails<cite> [^2]</cite>.

To preserve the state of the Kubernetes cluster, all **cluster configuration data** is saved in `etcd`. **etcd** is a **distributed key-value store that only holds data related to the cluster state, not client workload data**. **etcd** can be **configured** on the **master node (stacked topology) or on its dedicated host (external topology)** to help reduce the chances of data store loss by decoupling it from the other control plane agents<cite> [^2]</cite>.

With the stacked etcd topology, high availability master node replicas also ensure the resilience of the etcd data store. However, that is not the case with the external etcd topology, where etcd hosts must be replicated separately for high availability, a configuration that introduces the need for additional hardware.

A **master node** runs the following **control plane** components<cite> [^1]</cite>:

+ `kube-apiserver` or API server
+ `kube-scheduler` or scheduler
+ `kube-controller-manager` or controller manager
+ `etcd` or data store

While a **worker node** has the following components:

+ `Container Runtime`
+ `kubelet` or node agent
+ `kube-proxy` or proxy
+ `Addons` for DNS, dashboard, cluster-level monitoring, and logging

## Master node

### kube-apiserver

All **administrative tasks are coordinated** by `kube-apiserver`, a central control plane component that runs on the master node. The **API server receives RESTful requests from users, operators, and external agents, then validates and processes them**. During processing, the API server reads the current state of the Kubernetes cluster from the etcd data store, and after the execution of a call, the resulting state of the Kubernetes cluster is saved in the distributed key-value data store for persistence. The API server is the only control plane component that communicates with the etcd data store, both for reading and saving Kubernetes cluster state information, acting as an intermediary interface for any other control plane agent querying the cluster state.

The **API server is highly configurable and customizable**. It can **scale horizontally**, and it also **supports adding custom secondary API servers**, a configuration that turns the primary API server into a proxy for all custom secondary API servers and routes all incoming RESTful calls to them based on custom-defined rules<cite> [^2]</cite>.


### kube-scheduler

The role of the `kube-scheduler` is to **assign new workload objects**, such as pods, to nodes. During the scheduling process, decisions are made based on the current state of the Kubernetes cluster and the requirements of the new object. The **scheduler obtains from the etcd data store**, through the API server, **the resource usage data for each worker node** in the cluster. The scheduler **also receives** from the API server **the requirements of the new object that are part of its configuration data**. Requirements may include constraints set by users and operators, such as scheduling work on a node labeled with **`disk == ssd`** as a key-value pair. The scheduler also **takes into account Quality of Service (QoS) requirements, data locality, affinity, anti-affinity, dependent data location, taints, cluster topology, etc**. Once all the cluster data is available, the scheduling algorithm filters the nodes with predicates to isolate potential candidate nodes, which are then scored with priorities to select the node that satisfies all the requirements for the new workload. The result of the decision process is communicated to the API server, which then delegates the workload deployment to other control plane agents.

The scheduler is highly configurable and customizable through scheduling policies, plugins, and profiles. Additional custom schedulers are also supported. A scheduler **is extremely important and complex in a multi-node** Kubernetes cluster<cite> [^2]</cite>.

### kube-controller-manager

A **control plane** component that **runs the controllers to regulate the state of the Kubernetes cluster**. Controllers are **watch loops** that **run continuously and compare the desired state** of the cluster (provided by the configuration data of objects) **with its current state** (obtained from the etcd data store through the API server). In case of a discrepancy, corrective actions are taken in the cluster until its current state matches the desired state. It runs controllers responsible for acting **when nodes become unavailable**, for **ensuring the expected number of pods**, for **creating endpoints**, **service accounts, and API access tokens**<cite> [^2]</cite>. Logically, **each controller is an independent process**, but to reduce complexity, **they are all compiled into a single binary and run in a single process**. These controllers include<cite> [^1]</cite>:

+ **Node controller**: responsible for detecting and responding when a node goes down
+ **Replication controller**: responsible for maintaining the correct number of pods for each replication controller in the system
+ **Endpoints controller**: builds the Endpoints object, i.e., joins Services and Pods
+ **Service account and token controllers**: create default accounts and API access tokens for new Namespaces.

### etcd

A **persistent, consistent, and distributed key-value data store used to store all Kubernetes cluster information**<cite> [^1]</cite>. **New data is appended** to the data store, never replaced. **Obsolete data is periodically compacted** to minimize the size of the data store.

Of all the control plane components, **only the API server can communicate with the etcd data store**.

The etcd CLI management tool, **etcdctl**, **provides options for backups, snapshots, and restores**. These are especially useful for a single-instance etcd Kubernetes cluster, common in development and learning environments. However, in Staging and Production environments, it is extremely important to replicate data stores in high availability mode.

Some Kubernetes cluster bootstrapping tools, such as `kubeadm`, provision **stacked etcd master nodes**, where the data store runs alongside the other control plane components on the same master node and shares resources with them<cite> [^2]</cite>.

<!-- ![Figure 2: Stacked etcd topology.](../img/kubeadm-ha-topology-stacked-etcd.svg) -->

For **data store isolation** from the control plane components, the bootstrapping process can be configured for an **external etcd topology**. The data store is deployed on a separate dedicated host from the control plane, **thus reducing the chances of an etcd failure**<cite> [^2]</cite>.

<!-- ![Figure 3: External etcd topology.](../img/kubeadm-ha-topology-external-etcd.svg) -->

Both **stacked and external etcd topologies support** `high availability` configurations. etcd is based on the **[Raft](https://en.wikipedia.org/wiki/Raft_(algorithm)) consensus protocol**, which **allows** a **set of machines to survive the failure of some of them**, including master node failures. At any given time, one of the nodes in the group will be the leader and the rest will be followers<cite> [^2]</cite>.

<!-- ![Figure 4: Leader and followers.](../img/Master_and_Followers.png) -->

**etcd is written** in the **Go** programming language. In Kubernetes, besides storing the cluster state, **etcd is also used to store configuration details such as subnets, ConfigMaps, Secrets, etc**.

## Worker node

A **worker node provides a runtime environment for client applications**. Although they are containerized microservices, these **applications are encapsulated in pods**, controlled by the cluster's control plane agents running on the master node. Pods are scheduled on worker nodes, where they find the necessary compute, memory, and storage resources to run, and networking to communicate with each other and the outside world. **A pod is the smallest scheduling unit in Kubernetes. It is a logical collection of one or more containers scheduled together**, and the collection can be started, stopped, or rescheduled as a single unit of work.

Additionally, in a multi-worker Kubernetes cluster, **network traffic between client users and the containerized applications** deployed in Pods is **handled directly by the worker nodes** and is not routed through the master node<cite> [^2]</cite>.

### Container Runtime

Although Kubernetes is described as a "container orchestration engine", **it does not have the ability to handle containers directly**. To manage the lifecycle of a container, Kubernetes **requires a container runtime** on the node where a Pod and its containers will be scheduled. Kubernetes supports many container runtimes<cite> [^2]</cite>:

+ **[Docker](https://www.docker.com/)**: although it is a container platform that uses `containerd` as its container runtime, it is the most popular option used with Kubernetes
+ **[CRI-O](https://cri-o.io/)**: a lightweight container runtime for Kubernetes that also supports Docker image registries
+ **[containerd](https://containerd.io/)**: a simple, portable container runtime that provides robustness
+ **[frakti](https://github.com/kubernetes/frakti#frakti)**: a hypervisor-based container runtime for Kubernetes

### kubelet

The `kubelet` is an **agent that runs on every node and communicates with the control plane components on the master node**. It receives pod definitions, primarily from the API server, and interacts with the container runtime on the node to **run containers associated with the pod**. It also **monitors the health and resources of the containers running in pods**. The kubelet agent takes a set of Pod specifications, called **PodSpecs**, that have been created by Kubernetes and ensures that the containers described in them are running and healthy.

The kubelet **connects** to container runtimes **through** a plugin based on the **Container Runtime Interface (CRI)**. The CRI consists of protocol buffers, gRPC APIs, libraries, and additional specifications and tools that are currently under development. To connect to interchangeable container runtimes, kubelet uses a shim application that provides a clear abstraction layer between kubelet and the container runtime.

<!-- ![Figure 5: Container Runtime Interface (CRI).](../img/CRI.png) -->

From [blog.kubernetes.io](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)

As shown above, the **kubelet acting as a gRPC client connects to the CRI shim**, which in turn **acts as a gRPC server** to perform container and image operations. The CRI implements two services: `ImageService` and `RuntimeService`. `ImageService` is responsible for all image-related operations, while `RuntimeService` is responsible for all pod and container-related operations<cite> [^2]</cite>.

### kube-proxy

`kube-proxy` is the **network agent that runs on every node, responsible for dynamic updates and maintenance of all network rules** on the node. It extracts Pod network details and forwards connection requests to Pods.

The kube-proxy **is responsible for TCP, UDP, and SCTP stream forwarding or round-robin forwarding across a set of pod backends**, and it **implements forwarding rules defined by users** through Service API objects<cite> [^2]</cite>.

### Addons

**Addons are cluster features and functionalities not yet available in Kubernetes**, so they are implemented through third-party pods and services<cite> [^2]</cite>.

+ **DNS**: the cluster DNS is a DNS server required to assign DNS records to Kubernetes objects and resources

+ **Dashboard**: a general-purpose web-based user interface for cluster management

+ **Monitoring**: collects cluster-level container metrics and stores them in a central data store

+ **Logging**: collects cluster-level container logs and stores them in a central log store for analysis.

## Networking challenges

Decoupled microservices-based applications rely heavily on networking to mimic the tight coupling that was once available in the monolithic era. Networking, in general, is not the easiest to understand and implement. Kubernetes is no exception: as an orchestrator of containerized microservices, it must address several distinct networking challenges<cite> [^2]</cite>:

+ **Container-to-container communication within pods**
+ **Pod-to-pod communication on the same node and across all cluster nodes**
+ **Pod-to-Service communication within the same namespace and across cluster namespaces**
+ **External-to-Service communication so that clients can access applications** in a cluster.

### Container to container within pods

By leveraging the virtualization features of the underlying host OS kernel, a container runtime **creates an isolated network space for each container** it starts. On Linux, this isolated network space is called a ___network namespace___. A ___network namespace___ **can be shared between containers or with the host operating system**.

When a pod is started, the `Container Runtime` initializes a special pause container with the sole purpose of creating a network namespace for the pod. All additional containers, created through user requests, running within the Pod will share the Pause container's network namespace so they can all communicate with each other via localhost.

### Pod to pod across nodes

In a Kubernetes cluster, pods are scheduled on nodes in a nearly unpredictable manner. Regardless of their host node, **pods are expected to be able to communicate with all other pods in the cluster**, all without the implementation of Network Address Translation (NAT). This is a fundamental requirement of any Kubernetes networking implementation.

The Kubernetes networking model aims to reduce complexity and **treats Pods as VMs on a network**, where each **VM is equipped with a network interface**, so each Pod receives a unique IP address. This model is called "**IP-per-Pod**" and ensures pod-to-pod communication, just as virtual machines can communicate with each other on the same network.

However, let us not forget about containers. They share the Pod's network namespace and must coordinate port assignments within the Pod just as applications would on a VM, while being able to communicate with each other on **localhost** within the Pod. However, **containers are integrated with the overall Kubernetes networking model through the use of Container Network Interface (CNI)**-compatible CNI plugins. **CNI is a set of specifications and libraries that allow plugins to configure networking for containers**. While there are some core plugins, most CNI plugins are third-party Software-Defined Networking (SDN) solutions that implement the Kubernetes networking model. In addition to addressing the fundamental networking model requirement, some networking solutions offer support for network policies. Flannel, Weave, and Calico are just a few of the SDN solutions available for Kubernetes clusters.

### Pod to the outside world

A successfully deployed containerized application running in pods within a Kubernetes cluster may require accessibility from the outside world. Kubernetes enables external accessibility through `Services`, **complex encapsulations of routing rule definitions stored in** `iptables` **on cluster nodes and implemented by kube-proxy agents**. By exposing services to the external world with the help of **kube-proxy**, applications become accessible from outside the cluster through a virtual IP address.
