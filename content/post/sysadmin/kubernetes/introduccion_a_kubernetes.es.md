+++
author = "Adur"
title = "Introducción a Kubernetes"
date = "2020-10-27"
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
+++

En esta entrada se definirán los conceptos básicos e introductorios de Kubernetes. Las fuentes principales de la información a continuación son el curso de **[Introduction to Kubernetes](https://www.edx.org/es/course/introduction-to-kubernetes)** de **The Linux Foundation** en edX, cuyos autores son Chris Pokorni y Neependra Khare, y la propia [documentación de Kubernetes](https://kubernetes.io/docs/home/).  

## ¿Qué es Kubernetes y por qué se utiliza?

**Kubernetes o “K8s”** es una plataforma portable y extensible de código libre, bajo la licencia [Apache 2.0](https://www.apache.org/licenses/LICENSE-2.0), para administrar cargas de trabajo y servicios. Sirve para la **automatización del despliegue, ajuste de escala y manejo de aplicaciones en contenedores**​ que fue originalmente diseñado por Google y donado a la [Cloud Native Computing Foundation](https://www.cncf.io/) (parte de la Linux Foundation). Soporta diferentes entornos para la ejecución de contenedores, incluido Docker <cite> [^1].</cite>
[^1]: [Wikipedia, Kubernetes](https://es.wikipedia.org/wiki/Kubernetes)

Actualmente, se tiende a una ejecución de procesos en lo que se conoce como **"nube"**. Aunque, por otra parte, se precede de un modelo conocido como **monolítico** donde se tienen principios de arquitectura software obsoletos, con grandes componentes programados en lenguajes de programación antiguos y todo el sistema desplegado requiriendo un hardware caro y costoso de manejar. 

Por lo tanto, la tendencia actual es separar y simplificar cada componente software para convertirlos en componentes distribuidos, descritos por sus respectivas características específicas. De manera que se crean **microservicios** que se acomplan unos con otros y fáciles de reemplazar o reubicar. La arquitectura de microservicios está alineada con los principios de la **arquitectura dirigida por eventos (EDA) y de la arquitectura orientada a servicios (SOA)**. 

Cada microservicio se desarrolla en un lenguaje de programación moderno, seleccionado para ser el más adecuado para el tipo de servicio y su función. Esto ofrece una gran **flexibilidad al combinar microservicios con hardware específico** cuando sea necesario, lo que permite implementaciones en **hardware básico de bajo costo**. Aunque la **naturaleza distribuida** de los microservicios **agrega complejidad** a la arquitectura, uno de los mayores beneficios de los microservicios es la **escalabilidad**. Con la aplicación general volviéndose **modular, cada microservicio se puede escalar individualmente**, ya sea de forma **manual o automatizada** a través del autoescalado basado en la demanda. Además, prácticamente **no hay tiempo de inactividad ni interrupción del servicio** para los clientes porque las actualizaciones se implementan sin problemas, un servicio a la vez, en lugar de tener que volver a compilar, reconstruir y reiniciar una aplicación monolítica completa <cite> [^2].</cite>
[^2]: [edX, Introduction to kubernetes - The Linux Foundation](https://www.edx.org/es/course/introduction-to-kubernetes). 

Resumiendo, estos microservicios se despliegan en contenedores y **kubernetes es un orquestador de contenedores**. Por lo tanto, para comprender qué es kubernetes, es necesario revisar los conceptos básicos sobre contendores y orquestadores de contenedores.

## ¿Qué son los contenedores?

Los **contenedores** son **espacios virtuales a nivel del sistema operativo** que 
agrupan el código de una aplicación con las bibliotecas y los archivos de configuración asociados, junto con las dependencias necesarias para que la aplicación se ejecute. Ofrecen **escalabilidad y alto rendimiento** a las aplicaciones en **cualquier infraestructura** que se elija. Son los más adecuados para ofrecer microservicios al proporcionar **entornos virtuales portátiles y aislados** para que las aplicaciones se ejecuten **sin interferencias de otras aplicaciones en ejecución**.

### Beneficios de usar contenedores

Los beneficios de usar contenedores incluyen<cite> [^3]:</cite>
[^3]: [Documentación Kubernetes, ¿Qué es Kubernetes?](https://kubernetes.io/es/docs/concepts/overview/what-is-kubernetes/)

+ **Ágil creación y despliegue de aplicaciones**: Mayor facilidad y eficiencia al crear imágenes de contenedor en vez de máquinas virtuales.
+ **Desarrollo, integración y despliegue continuo**: Permite que la imagen de contenedor se construya y despliegue de forma frecuente y confiable, facilitando los _rollbacks_ debido a que la imagen es inmutable.
+ **Separación de tareas entre Dev y Ops**: Puedes crear imágenes de contenedor al momento de compilar y no al desplegar, desacoplando la aplicación de la infraestructura.
+ **Observabilidad**: No solamente se presenta la información y métricas del sistema operativo, sino la salud de la aplicación y otras señales.
+ **Consistencia entre los entornos de desarrollo, pruebas y producción**: La aplicación funciona igual en un laptop y en la nube.
+ **Portabilidad entre nubes y distribuciones**: Funciona en Ubuntu, RHEL, CoreOS, tu datacenter físico, Google Kubernetes Engine y todo lo demás.
+ **Administración centrada en la aplicación**: Eleva el nivel de abstracción del sistema operativo y el hardware virtualizado a la aplicación que funciona en un sistema con recursos lógicos.
+ **Microservicios distribuidos, elásticos, liberados y débilmente acoplados**: Las aplicaciones se separan en piezas pequeñas e independientes que pueden ser desplegadas y administradas de forma dinámica, y no como una aplicación monolítica que opera en una sola máquina de gran capacidad.
+ **Aislamiento de recursos**: Hace el rendimiento de la aplicación más predecible.
+ **Utilización de recursos**: Permite mayor eficiencia y densidad.

## ¿Qué son los orquestadores de contenedores?

En entornos de desarrollo, la ejecución de contenedores en un solo host para el desarrollo y la prueba de aplicaciones puede ser una opción. Sin embargo, al **migrar a entornos de control de calidad (QA) y producción (Prod)**, ya no es una opción viable porque las aplicaciones y los **servicios deben cumplir con requisitos** específicos<cite> [^2]</cite>:

* Tolerancia a fallos.
* Escalabilidad bajo demanda.
* Uso óptimo de recursos.
* Descubrimiento automático para descubrir y comunicarse automáticamente entre componentes.
* Accesibilidad desde el mundo exterior.
* Actualizaciones o parches de seguridad sin interrupciones sin tiempo de inactividad.

Los **orquestadores de contenedores** son herramientas que **agrupan sistemas para formar clústeres en los que la implementación y la administración de contenedores se automatizan a escala y cumplen los requisitos** mencionados anteriormente.

Existen varias soluciones de orquestadores de contenedores y algunos de los disponibles son:

* Amazon Elastic Container Service (ECS). 
* Azure Container Instances.
* Azure Service Fabric.
* Kubernetes.
* Marathon.
* Nomad.
* Docker Swarm.

Aunque sea viable mantener algún contenedor manualmente, **los orquestadores facilitan mucho la administración a los operadores**, principalmente cuando se trata de cientos o miles de contenedores. La mayoría de los orquestadores de contenedores **pueden realizar** las siguientes acciones <cite> [^2]</cite>:

* **Agrupar hosts** mientras se crea un clúster.
* Programar contenedores para que se **ejecuten en hosts del clúster según la disponibilidad de recursos**.
* Permitir que los contenedores de un clúster se **comuniquen entre sí independientemente del host en el que estén implementados** en el clúster.
* **Vincular contenedores y recursos de almacenamiento**.
* Agrupar conjuntos de contenedores similares y vincularlos a construcciones de **equilibrio de carga** para simplificar el acceso a las aplicaciones en contenedores, creando un **nivel de abstracción entre los contenedores y el usuario**.
* **Gestionar y optimizar el uso de recursos**.
* Permitir la **implementación de políticas para proteger el acceso** a las aplicaciones que se ejecutan dentro de los contenedores.

Con todas estas características, configurables pero flexibles, los orquestadores de contenedores son una buena opción cuando se trata de administrar aplicaciones en contenedores a gran escala.

## Características de Kubernetes

Kubernetes ofrece un conjunto muy amplio de funciones para la orquestación de contenedores. Sus principales características son<cite> [^2]</cite>:

* **Empaquetado automático de contenedores**: Kubernetes programa automáticamente los contenedores en función de las necesidades y limitaciones de los recursos, para maximizar la utilización sin sacrificar la disponibilidad.

* **Autocuración**: Kubernetes reemplaza y reprograma automáticamente los contenedores de los nodos fallidos. Elimina y reinicia los contenedores que no responden a las comprobaciones de estado, según las reglas o políticas existentes. También evita que el tráfico se enrute a contenedores que no responden.

* **Escalabilidad horizontal**: Con Kubernetes, las aplicaciones se escalan de forma manual o automática según la CPU o el uso de métricas personalizadas.

* **Descubrimiento de servicios y equilibrio de carga**: Los contenedores reciben sus propias direcciones IP de Kubernetes, mientras que este asigna un único nombre de sistema de nombres de dominio (DNS) a un conjunto de contenedores para ayudar a equilibrar la carga de las solicitudes en todos los contenedores del conjunto.

* **Implementaciones y restauraciones automatizadas**: Kubernetes implementa y restaura las actualizaciones de la aplicación y los cambios de configuración sin problemas, monitoreando constantemente el estado de la aplicación para evitar cualquier tiempo de inactividad.

* **Gestión de secretos y configuraciones**: Kubernetes administra los datos confidenciales y los detalles de la configuración de una aplicación por separado de la imagen del contenedor, para evitar una reconstrucción de la imagen respectiva. Los secretos consisten en la información sensible o confidencial que se pasa a la aplicación sin revelar el contenido sensible a la configuración del código.

* **Orquestación de almacenamiento**: Kubernetes monta automáticamente soluciones de almacenamiento definido por software (SDS) en contenedores de almacenamiento local, proveedores de nube externos, almacenamiento distribuido o sistemas de almacenamiento en red.

* **Ejecución por lotes**: Kubernetes admite la ejecución por lotes, trabajos de larga ejecución y reemplaza los contenedores fallidos.

La **arquitectura de Kubernetes es modular e interconectable**. No solo organiza aplicaciones de tipo microservicios como módulos desacoplados, sino que también su arquitectura sigue patrones de microservicios desacoplados. Las funcionalidades de Kubernetes se puede ampliar escribiendo recursos personalizados, operadores, API personalizadas, reglas de programación o extensiones.
