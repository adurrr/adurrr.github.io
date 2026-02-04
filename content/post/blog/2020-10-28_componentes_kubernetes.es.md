+++
author = "Adur"
title = "Componentes de la arquitectura de Kubernetes"
date = "2020-10-28"
description = "Teoría básica para empezar con kubernetes."
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

En esta entrada se definirán los componentes de la arquitectura de Kubernetes. Las fuentes principales de la información a continuación son el curso de **[Introduction to Kubernetes](https://www.edx.org/es/course/introduction-to-kubernetes)** de **The Linux Foundation** en edX, cuyos autores son Chris Pokorni y Neependra Khare, y la propia [documentación de Kubernetes](https://kubernetes.io/docs/home/). 

## Componentes de la arquitectura en Kubernetes

Un clúster de Kubernetes consta de un conjunto de máquinas trabajadoras, llamadas nodos, que ejecutan aplicaciones en contenedores. Cada clúster tiene al menos un nodo trabajador. En un nivel muy alto de abstracción, Kubernetes tiene los siguientes componentes principales:

* Uno o más **nodos maestros (master nodes)**, en la parte del plano de control (control plane).
* Uno o más **nodos trabajadores (worker nodes)**.

Se puede ver en la siguiente figura que es la **arquitectura de los componentes de un cluster de Kubernetes**<cite> [^1]:</cite>
[^1]: [Documentation Kubernetes, Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)

<!-- ![Figura 1: Componentes de la arquitectura de Kubernetes.](../img/arch-kubernetes.svg) -->


El **nodo maestro** proporciona un entorno de ejecución para el plano de control responsable de **administrar el estado de un clúster de Kubernetes y es el cerebro detrás de todas las operaciones** dentro del clúster. Los **componentes del plano de control** son agentes con **roles muy distintos en la gestión del clúster**. Para **comunicarse** con el clúster de Kubernetes, los usuarios **envían solicitudes al plano de control a través** de una herramienta de interfaz de **línea de comandos** (CLI), un panel de **interfaz de usuario web** o una **interfaz de programación de aplicaciones** (API)<cite> [^2]</cite>.
[^2]: [edX, Introduction to kubernetes - The Linux Foundation](https://www.edx.org/es/course/introduction-to-kubernetes)

Es importante mantener el plano de control funcionando a toda costa. **La pérdida del plano de control puede generar tiempo de inactividad, provocando la interrupción del servicio** a los clientes, con una posible pérdida de negocio. Para **garantizar la tolerancia a fallos del plano de control**, se pueden agregar **réplicas del nodo maestro** al clúster, configuradas **en modo de alta disponibilidad**. Si bien solo uno de los nodos maestros está dedicado a administrar activamente el clúster, los componentes del plano de control permanecen sincronizados en las réplicas del nodo maestro. Este tipo de configuración agrega resistencia al plano de control del clúster, en caso de que falle el nodo maestro que está activo<cite> [^2]</cite>.

Para conservar el estado del clúster de Kubernetes, todos los **datos de configuración del clúster** se guardan en `etcd`. **etcd** es un **almacén de valores clave distribuido que solo contiene datos relacionados con el estado del clúster, sin datos de carga de trabajo del cliente**. **etcd** se puede **configurar** en el **nodo maestro (topología apilada) o en su host dedicado (topología externa)** para ayudar a reducir las posibilidades de pérdida del almacén de datos al desacoplarlo de los otros agentes del plano de control<cite> [^2]</cite>.

Con la topología de etcd apilada, las réplicas del nodo maestro de alta disponibilidad también garantizan la resistencia del almacén de datos de etcd. Sin embargo, ese no es el caso de la topología externa etcd, donde los hosts etcd deben replicarse por separado para alta disponibilidad, una configuración que introduce la necesidad de hardware adicional.

Un **nodo maestro** ejecuta los siguientes componentes del **plano de control**<cite> [^1]</cite>:

+ `kube-apiserver` o servidor API
+ `kube-scheduler` o planificador
+ `kube-controller-manager` o manejador de controladores
+ `etcd` o almacén de datos

Mientras que un **nodo trabajador** tiene los siguientes componentes:

+ `Container Runtime` o entorno de ejecución del contenedor
+ `kubelet` o agente de nodo
+ `kube-proxy` o proxy
+ `Addons` o complementos para DNS, interfaz de usuario, monitoreo y registro a nivel de clúster

## Nodo maestro

### kube-apiserver

Todas las **tareas administrativas están coordinadas** por `kube-apiserver`, un componente central del plano de control que se ejecuta en el nodo maestro. El **servidor API recibe las peticiones RESTful de usuarios, operadores y agentes externos, luego las valida y las procesa**. Durante el procesamiento, el servidor API lee el estado actual del clúster de Kubernetes del almacén de datos etcd y, después de la ejecución de una llamada, el estado resultante del clúster de Kubernetes se guarda en el almacén de datos distribuido de clave-valor para debida su persistencia. El servidor API es el único componente del plano de control que se comunica con el almacén de datos etcd, tanto para leer como para guardar la información del estado del clúster de Kubernetes, actuando como una interfaz intermedia para cualquier otro agente del plano de control que pregunte sobre el estado del clúster.

El **servidor API es altamente configurable y personalizable**. Puede **escalar horizontalmente**, pero **también admite la adición de servidores API secundarios personalizados**, una configuración que transforma el servidor API principal en un proxy para todos los servidores API personalizados secundarios y enruta todas las llamadas RESTful entrantes a ellos en función de reglas definidas personalizadas<cite> [^2]</cite>.


### kube-scheduler

La función del `kube-scheduler` o planificador es **asignar nuevos objetos de carga de trabajo**, como pods, a los nodos. Durante el proceso de planificación, las decisiones se toman según el estado actual del clúster de Kubernetes y los requisitos del nuevo objeto. El **planificador obtiene del almacén de datos etcd**, a través del servidor API, **los datos de uso de recursos para cada nodo trabajador** en el clúster. El planificador **también recibe** del servidor API **los requisitos del nuevo objeto que forman parte de sus datos de configuración**. Los requisitos pueden incluir restricciones que los usuarios y operadores establezcan, como programar el trabajo en un nodo etiquetado con **`disk == ssd`** como par clave/valor. El planificador también **tiene en cuenta los requisitos de calidad de servicio (QoS), la localidad de los datos, la afinidad, la antiafinidad, localización de datos dependientes, la tolerancia, la topología del clúster, etc**. Una vez que todos los datos del clúster están disponibles, el algoritmo de planificación filtra los nodos con predicados para aislar los posibles nodos candidatos que luego son puntuados con prioridades para seleccionar el nodo que satisface todos los requisitos para la nueva carga de trabajo. El resultado del proceso de decisión se comunica al servidor API, que luego delega la implementación de la carga de trabajo con otros agentes del plano de control.

El planificador es altamente configurable y personalizable mediante políticas de planificación, complementos y perfiles. También se admiten planificadores adicionales personalizados. Un planificador **es extremadamente importante y complejo en un clúster** de Kubernetes de **varios nodos**<cite> [^2]</cite>.

### kube-controller-manager

Componente del **plano de control** que **ejecuta los controladores para regular el estado del clúster de Kubernetes**. Los controladores son **bucles de vigilancia** que **se ejecutan continuamente y comparan el estado deseado** del clúster (proporcionado por los datos de configuración de los objetos) **con su estado actual** (obtenido del almacén de datos etcd a través del servidor API). En caso de una discrepancia, se toman medidas correctivas en el clúster hasta que su estado actual coincida con el estado deseado. Ejecuta controladores responsables de actuar **cuando los nodos dejan de estar disponibles**, para **garantizar que el número de pods** sean los esperados, para **crear endpoints**, **cuentas de servicio y tokens de acceso a la API**<cite> [^2]</cite>. Lógicamente **cada controlador es un proceso independiente**, pero para reducir la complejidad, **todos se compilan en un único binario y se ejecuta en un mismo proceso**. Estos controladores incluyen<cite> [^1]</cite>:

+ **Controlador de nodos**: es el responsable de detectar y responder cuándo un nodo deja de funcionar
+ **Controlador de replicación**: es el responsable de mantener el número correcto de pods para cada controlador de replicación del sistema
+ **Controlador de endpoints**: construye el objeto Endpoints, es decir, hace una unión entre los Services y los Pods
+ **Controladores de tokens y cuentas de servicio**: crean cuentas y tokens de acceso a la API por defecto para los nuevos Namespaces.

### etcd

**Almacén de datos persistente, consistente y distribuido de clave-valor utilizado para almacenar toda a la información del clúster de Kubernetes**<cite> [^1]</cite>. Los **nuevos datos se añaden** al almacén de datos, nunca se reemplazan. Los **datos obsoletos se comprimen** periódicamente para minimizar el tamaño del almacén de datos.

De todos los componentes del plano de control, **solo el servidor API puede comunicarse con el almacén de datos etcd**.

La herramienta de administración CLI de etcd, **etcdctl**, **proporciona la opción de realizar copias de seguridad, instantáneas y restauraciones**. Son especialmente útiles para un clúster de Kubernetes de única instancia de etcd, común en entornos de desarrollo y aprendizaje. Sin embargo, en los entornos Stage y Production, es extremadamente importante replicar los almacenes de datos en modo alta disponibilidad.

Algunas herramientas de arranque del clúster de Kubernetes, como `kubeadm`, aprovisionan **nodos maestros etcd apilados**, donde el almacén de datos se ejecuta junto con los otros componentes del plano de control en el mismo nodo maestro y los comparte con ellos<cite> [^2]</cite>.

<!-- ![Figura 2: Topología etcd apilada.](../img/kubeadm-ha-topology-stacked-etcd.svg) -->

Para el **aislamiento del almacén de datos** de los componentes del plano de control, el proceso de arranque se puede configurar para una **topología etcd externa**. El almacén de datos se despliega en un host dedicado separado del plano de control, **reduciendo así las posibilidades de un fallo del etcd**<cite> [^2]</cite>.

<!-- ![Figura 3: Topología etcd externa.](../img/kubeadm-ha-topology-external-etcd.svg) -->


Las **topologías de etcd apilado y externa admiten** configuraciones de `alta disponibilidad`. etcd se basa en **protocolo de consenso [Raft](https://es.wikipedia.org/wiki/Raft)** que **permite** que un **conjunto de máquinas pueda sobrevivir al fallo de algunas de ellas**, incluido a fallos de nodo maestro. En un momento dado, uno de los nodos del grupo será el maestro y el resto serán los seguidores<cite> [^2]</cite>.

<!-- ![Figura 4: Maestros y seguidores.](../img/Master_and_Followers.png) -->

**etcd está escrito** en el lenguaje de programación **Go**. En Kubernetes, además de almacenar el estado del clúster, **etcd también se usa para almacenar detalles de configuración como subredes, ConfigMaps, Secrets, etc**.

## Nodo trabajador

Un **nodo trabajador proporciona un entorno de ejecución para las aplicaciones cliente**. Aunque son microservicios en contenedores, estas **aplicaciones están encapsuladas en pods**, controlados por los agentes del plano de control del clúster que se ejecutan en el nodo maestro. Los pods se programan en los nodos trabajadores, donde encuentran los recursos informáticos, de memoria y de almacenamiento necesarios para ejecutarse, y las redes para comunicarse entre sí y con el mundo exterior. **Un pod es la unidad de programación más pequeña de Kubernetes. Es una colección lógica de uno o más contenedores programados juntos**, y la colección se puede iniciar, detener o reprogramar como una sola unidad de trabajo.

Además, en un clúster de Kubernetes de varios trabajadores, el **tráfico de red entre los usuarios del cliente y las aplicaciones en contenedores** implementadas en Pods lo **manejan directamente los nodos trabajadores** y no se enruta a través del nodo maestro<cite> [^2]</cite>.

### Container Runtime

Aunque Kubernetes se describe como un "motor de orquestación de contenedores", **no tiene la capacidad de manejar contenedores directamente**. Para administrar el ciclo de vida de un contenedor, Kubernetes **requiere un entorno de ejecución del contendor** en el nodo donde se programará un Pod y sus contenedores. Kubernetes admite muchos entornos de ejecución de contenedores<cite> [^2]</cite>:

+ **[Docker](https://www.docker.com/)**: aunque es una plataforma de contenedores que usa `containerd` como entorno de ejecución de contenedor, es más popular usado con Kubernetes
+ **[CRI-O](https://cri-o.io/)**: un entorno de ejecución de contenedor ligero para Kubernetes, también admite registros de imágenes de Docker
+ **[containerd](https://containerd.io/)**: un entorno de ejecución de contenedor simple y portátil que proporciona robustez
+ **[frakti](https://github.com/kubernetes/frakti#frakti)**: un tiempo de ejecución de contenedor basado en hipervisor para Kubernetes

### kubelet

El `kubelet` es un **agente que se ejecuta en cada nodo y se comunica con los componentes del plano de control desde el nodo maestro**. Recibe definiciones de pod, principalmente del servidor API, e interactúa con el entorno de ejecución del contenedor en el nodo para **ejecutar contenedores asociados con el pod**. También **supervisa el estado y los recursos de los contenedores que ejecutan pods**. El agente kubelet toma un conjunto de especificaciones de Pod, llamados **PodSpecs**, que han sido creados por Kubernetes y garantiza que los contenedores descritos en ellos estén funcionando y en buen estado.

El kubelet **se conecta** a los entornos de ejecución del contenedor **a través** de un complemento basado en el **Container Runtime Interface (CRI)**. El CRI consta de protocolos de búferes, API de gRPC, bibliotecas y especificaciones y herramientas adicionales que se encuentran actualmente en desarrollo. Para conectarse a tiempos de ejecución de contenedores intercambiables, kubelet utiliza una aplicación de compensación que proporciona una capa de abstracción clara entre kubelet y el tiempo de ejecución del contenedor.

<!-- ![Figura 5: Container Runtime Interface (CRI).](../img/CRI.png) -->

Obtenido de [blog.kubernetes.io](https://kubernetes.io/blog/2016/12/container-runtime-interface-cri-in-kubernetes/)

Como se muestra arriba, el **kubelet que actúa como cliente grpc se conecta al shim CRI**, que a su vez, **actúa como servidor grpc** para realizar operaciones en el contenedor y la imagen. El CRI implementa dos servicios: `ImageService` y `RuntimeService`. `ImageService` es responsable de todas las operaciones relacionadas con la imagen, mientras que `RuntimeService` es responsable de todas las operaciones relacionadas con el pod y el contenedor<cite> [^2]</cite>.

### kube-proxy

El `kube-proxy` es el **agente de red que se ejecuta en cada nodo responsable de las actualizaciones dinámicas y el mantenimiento de todas las reglas de red** en el nodo. Extrae los detalles de la red de Pods y reenvía las solicitudes de conexión a los Pods.

El proxy kube **es responsable del reenvío de transmisiones TCP, UDP y SCTP o el reenvío por turnos a través de un conjunto de backends de pod**, e **implementa reglas de reenvío definidas por los usuarios**a través de objetos de API de servicio<cite> [^2]</cite>.

### Addons

Los **complementos son características y funcionalidades del clúster que aún no están disponibles en Kubernetes**, por lo que se implementan a través de pods y servicios de terceros<cite> [^2]</cite>.

+ **DNS**: el DNS del clúster es un servidor DNS necesario para asignar registros DNS a objetos y recursos de Kubernetes

+ **Dashboard**: una interfaz de usuario basada en web de propósito general para la administración de clústeres

+ **Monitorización**: recopila métricas de contenedores a nivel de clúster y las guarda en un almacén de datos central

+ **Registro**: recopila registros de contenedores a nivel de clúster y los guarda en un almacén de registros central para su análisis.

## Desafíos de red

Las aplicaciones basadas en microservicios desacoplados dependen en gran medida de las redes para imitar el acoplamiento estrecho que alguna vez estuvo disponible en la era monolítica. Las redes, en general, no son las más fáciles de entender e implementar. Kubernetes no es una excepción: como orquestador de microservicios en contenedores, debe abordar algunos desafíos de redes distintos<cite> [^2]</cite>:

+ Comunicación de **contenedor a contenedor dentro de los pods**
+ Comunicación de **pod a pod en el mismo nodo y en todos los nodos del clúster**
+ Comunicación **Pod-to-Service dentro del mismo espacio de nombres y entre espacios de nombres de clústeres**
+ Comunicación **externa al servicio para que los clientes accedan a aplicaciones** en un clúster.

### De contenedor a contenedor dentro de los pods

Al hacer uso de las funciones de virtualización del kernel del sistema operativo host subyacente, un entorno de ejecución de contenedor **crea un espacio de red aislado para cada contenedor** que inicia. En Linux, este espacio de red aislado que se denomina ___network namespace___. Un ___network namespace___ **se puede compartir entre contenedores o con el sistema operativo del host**.

Cuando se inicia un pod, el `Container Runtime` inicializa un contenedor de pausa especial con el único propósito de crear un espacio de nombres de red para el pod. Todos los contenedores adicionales, creados a través de solicitudes de usuarios, que se ejecutan dentro del Pod compartirán el espacio de nombres de red del contenedor Pause para que todos puedan comunicarse entre sí a través de localhost.

### De pod a pod a través de nodos

En un clúster de Kubernetes, los pods se programan en los nodos de una manera casi impredecible. Independientemente de su nodo host, **se espera que los pods puedan comunicarse con todos los demás pods del clúster**, todo esto sin la implementación de la traducciones de direcciones de red (NAT). Este es un requisito fundamental de cualquier implementación de redes en Kubernetes.

El modelo de red de Kubernetes tiene como objetivo reducir la complejidad y **trata a los Pods como VM en una red**, donde cada **VM está equipada con una interfaz de red**, por lo que cada Pod recibe una dirección IP única. Este modelo se llama "**IP por pod**" y garantiza la comunicación de pod a pod, al igual que las máquinas virtuales pueden comunicarse entre sí en la misma red.

Sin embargo, no nos olvidemos de los contenedores. Comparten el espacio de nombres de la red del Pod y deben coordinar la asignación de puertos dentro del Pod tal como lo harían las aplicaciones en una VM, todo mientras pueden comunicarse entre sí en **localhost**, dentro del Pod. Sin embargo, **los contenedores se integran con el modelo de red general de Kubernetes mediante el uso de Container Network Interface (CNI)** compatible con los complementos CNI. **CNI es un conjunto de una especificación y bibliotecas que permiten a los complementos configurar la red para contenedores**. Si bien hay algunos complementos principales, la mayoría de los complementos CNI son soluciones de redes definidas por software (SDN) de terceros que implementan el modelo de redes de Kubernetes. Además de abordar el requisito fundamental del modelo de red, algunas soluciones de red ofrecen soporte para políticas de red. Flannel, Weave, Calico son solo algunas de las soluciones SDN disponibles para los clústeres de Kubernetes.

### De Pod al mundo exterior

Una aplicación implementada con éxito en contenedores que se ejecuta en pods dentro de un clúster de Kubernetes puede requerir accesibilidad desde el mundo exterior. Kubernetes permite la accesibilidad externa a través de `Servicios`, **encapsulaciones complejas de definiciones de reglas de enrutamiento de red almacenadas en** `iptables` **en nodos de clúster e implementadas por agentes de kube-proxy**. Al exponer los servicios al mundo externo con la ayuda de **kube-proxy**, las aplicaciones se vuelven accesibles desde fuera del clúster a través de una dirección IP virtual.
