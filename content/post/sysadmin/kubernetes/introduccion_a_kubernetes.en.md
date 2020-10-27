+++
author = "Adur"
title = "Introduction to Kubernetes"
date = "2020-10-27"
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
+++

This post covers the basic introductory concepts of Kubernetes. The main sources of information are the **[Introduction to Kubernetes](https://www.edx.org/es/course/introduction-to-kubernetes)** course by **The Linux Foundation** on edX, authored by Chris Pokorni and Neependra Khare, and the official [Kubernetes documentation](https://kubernetes.io/docs/home/).

## What is Kubernetes and why is it used?

**Kubernetes or "K8s"** is a portable, extensible, open-source platform, licensed under [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0), for managing workloads and services. It is used for **automating the deployment, scaling, and management of containerized applications** and was originally designed by Google and donated to the [Cloud Native Computing Foundation](https://www.cncf.io/) (part of the Linux Foundation). It supports different container runtime environments, including Docker <cite> [^1].</cite>
[^1]: [Wikipedia, Kubernetes](https://es.wikipedia.org/wiki/Kubernetes)

Currently, there is a trend toward running processes in what is known as the **"cloud"**. However, this was preceded by a model known as **monolithic**, which relied on outdated software architecture principles, with large components written in legacy programming languages and the entire system deployed on expensive hardware that was costly to manage.

Therefore, the current trend is to separate and simplify each software component to turn them into distributed components, described by their specific characteristics. This creates **microservices** that can be coupled together and are easy to replace or relocate. The microservices architecture is aligned with the principles of **Event-Driven Architecture (EDA) and Service-Oriented Architecture (SOA)**.

Each microservice is developed in a modern programming language, selected as the most suitable for the type of service and its function. This offers great **flexibility in combining microservices with specific hardware** when necessary, enabling deployments on **low-cost commodity hardware**. Although the **distributed nature** of microservices **adds complexity** to the architecture, one of the greatest benefits of microservices is **scalability**. With the overall application becoming **modular, each microservice can be scaled individually**, either **manually or automatically** through demand-based autoscaling. Additionally, there is practically **no downtime or service disruption** for clients because updates are rolled out seamlessly, one service at a time, instead of having to recompile, rebuild, and restart an entire monolithic application <cite> [^2].</cite>
[^2]: [edX, Introduction to kubernetes - The Linux Foundation](https://www.edx.org/es/course/introduction-to-kubernetes).

In summary, these microservices are deployed in containers and **Kubernetes is a container orchestrator**. Therefore, to understand what Kubernetes is, it is necessary to review the basic concepts of containers and container orchestrators.

## What are containers?

**Containers** are **OS-level virtual spaces** that bundle application code with associated libraries and configuration files, along with the dependencies needed for the application to run. They provide **scalability and high performance** to applications on **any infrastructure** of your choice. They are best suited for delivering microservices by providing **portable and isolated virtual environments** so applications can run **without interference from other running applications**.

### Benefits of using containers

The benefits of using containers include<cite> [^3]:</cite>
[^3]: [Kubernetes Documentation, What is Kubernetes?](https://kubernetes.io/es/docs/concepts/overview/what-is-kubernetes/)

+ **Agile application creation and deployment**: Greater ease and efficiency in creating container images instead of virtual machines.
+ **Continuous development, integration, and deployment**: Allows container images to be built and deployed frequently and reliably, facilitating rollbacks since the image is immutable.
+ **Dev and Ops separation of concerns**: You can create container images at build time rather than at deployment time, decoupling the application from the infrastructure.
+ **Observability**: Surfaces not only OS-level information and metrics, but also application health and other signals.
+ **Consistency across development, testing, and production environments**: The application runs the same on a laptop as it does in the cloud.
+ **Cloud and OS distribution portability**: Runs on Ubuntu, RHEL, CoreOS, your physical datacenter, Google Kubernetes Engine, and everything else.
+ **Application-centric management**: Raises the level of abstraction from the OS and virtualized hardware to the application running on a system with logical resources.
+ **Loosely coupled, distributed, elastic, liberated microservices**: Applications are broken into smaller, independent pieces that can be deployed and managed dynamically, rather than as a monolithic application running on a single high-capacity machine.
+ **Resource isolation**: Makes application performance more predictable.
+ **Resource utilization**: Enables greater efficiency and density.

## What are container orchestrators?

In development environments, running containers on a single host for application development and testing can be a viable option. However, when **migrating to quality assurance (QA) and production (Prod) environments**, it is no longer viable because applications and **services must meet specific requirements**<cite> [^2]</cite>:

* Fault tolerance.
* On-demand scalability.
* Optimal resource usage.
* Auto-discovery to automatically discover and communicate between components.
* Accessibility from the outside world.
* Seamless security updates or patches with zero downtime.

**Container orchestrators** are tools that **group systems to form clusters where container deployment and management are automated at scale, meeting the requirements** listed above.

There are several container orchestrator solutions, and some of the available ones are:

* Amazon Elastic Container Service (ECS).
* Azure Container Instances.
* Azure Service Fabric.
* Kubernetes.
* Marathon.
* Nomad.
* Docker Swarm.

While it is feasible to manage a few containers manually, **orchestrators greatly simplify administration for operators**, especially when dealing with hundreds or thousands of containers. Most container orchestrators **can perform** the following actions <cite> [^2]</cite>:

* **Group hosts** while creating a cluster.
* Schedule containers to **run on cluster hosts based on resource availability**.
* Allow containers in a cluster to **communicate with each other regardless of the host they are deployed on** in the cluster.
* **Bind containers and storage resources**.
* Group sets of similar containers and bind them to **load-balancing** constructs to simplify access to containerized applications, creating a **level of abstraction between containers and the user**.
* **Manage and optimize resource usage**.
* Allow the **implementation of policies to secure access** to applications running inside containers.

With all these configurable yet flexible features, container orchestrators are a great choice when it comes to managing containerized applications at scale.

## Kubernetes features

Kubernetes offers a very broad set of features for container orchestration. Its main features are<cite> [^2]</cite>:

* **Automatic bin packing**: Kubernetes automatically schedules containers based on resource needs and constraints, to maximize utilization without sacrificing availability.

* **Self-healing**: Kubernetes automatically replaces and reschedules containers from failed nodes. It kills and restarts containers that do not respond to health checks, based on existing rules or policies. It also prevents traffic from being routed to unresponsive containers.

* **Horizontal scaling**: With Kubernetes, applications scale manually or automatically based on CPU usage or custom metrics.

* **Service discovery and load balancing**: Containers receive their own IP addresses from Kubernetes, while Kubernetes assigns a single Domain Name System (DNS) name to a set of containers to help load balance requests across all containers in the set.

* **Automated rollouts and rollbacks**: Kubernetes seamlessly rolls out and rolls back application updates and configuration changes, constantly monitoring application health to prevent any downtime.

* **Secret and configuration management**: Kubernetes manages sensitive data and application configuration details separately from the container image, to avoid rebuilding the respective image. Secrets consist of sensitive or confidential information passed to the application without revealing the sensitive content to the code configuration.

* **Storage orchestration**: Kubernetes automatically mounts Software-Defined Storage (SDS) solutions to containers from local storage, external cloud providers, distributed storage, or network storage systems.

* **Batch execution**: Kubernetes supports batch execution, long-running jobs, and replaces failed containers.

The **architecture of Kubernetes is modular and pluggable**. It not only orchestrates microservice-type applications as decoupled modules, but its own architecture also follows decoupled microservice patterns. Kubernetes functionality can be extended by writing custom resources, operators, custom APIs, scheduling rules, or plugins.
