+++
author = "Adur"
title = "Base de datos SQL o NoSQL"
date = "2020-10-08"
description = "Descripción y comparación de los modelos relacionales y no relacionales de las bases de datos."
featured = true
tags = [
    "programación",
    "sql",
    "nosql",
    "bases de datos"
]
categories = [
    "desarrollo software"
]
series = ["Programación"]
aliases = [""]
thumbnail = "images/sql_nosql.png"
+++

En esta publicación se detallará una **comparación con las ventajas y desventajas de SQL y NoSQL**. Para la elección de una base de datos que se ajuste a las necesidades de cada proyecto, es necesario conocer las diferencias entre los distintos modelos.  

# SQL

<abbr title="Structured Query Language">SQL</abbr> (Structured Query Language) o **lenguaje de consulta estructurada** se utiliza en programación para administrar y recuperar información de **sistemas de gestión de bases de datos relacionales**. Se caracteriza por su manejo del **álgebra y el cálculo relacional** para efectuar consultas con el fin de operar con la información de bases de datos.<cite> [^1]</cite>
[^1]: [Wikipedia, SQL](https://es.wikipedia.org/wiki/SQL)

Se pueden garantizar las características
ACID (Atomicidad, Consistencia, Aislamiento y Durabilidad):<cite> [^2]</cite>
[^2]: [Wikipedia, ACID](https://es.wikipedia.org/wiki/ACID)
* **Atomicidad:** Si cuando una operación consiste en una serie de pasos, de los que o bien se ejecutan todos o ninguno, es decir, las transacciones son completas.
* **Consistencia:** (Integridad). Es la propiedad que asegura que sólo se empieza aquello que se puede acabar. Por lo tanto se ejecutan aquellas operaciones que no van a romper las reglas y directrices de Integridad de la base de datos. La propiedad de consistencia sostiene que cualquier transacción llevará a la base de datos desde un estado válido a otro también válido. "La Integridad de la Base de Datos nos permite asegurar que los datos son exactos y consistentes, es decir que estén siempre intactos, sean siempre los esperados y que de ninguna manera cambian ni se deformen. De esta manera podemos garantizar que la información que se presenta al usuario será siempre la misma."
* **Aislamiento:** Esta propiedad asegura que una operación no puede afectar a otras. Esto asegura que la realización de dos transacciones sobre la misma información sean independientes y no generen ningún tipo de error. Esta propiedad define cómo y cuándo los cambios producidos por una operación se hacen visibles para las demás operaciones concurrentes. El aislamiento puede alcanzarse en distintos niveles, siendo el parámetro esencial a la hora de seleccionar SGBDs.
* **Durabilidad:** (Persistencia). Esta propiedad asegura que una vez realizada la operación, esta persistirá y no se podrá deshacer aunque falle el sistema y que de esta forma los datos sobrevivan de alguna manera.


Por lo tanto, un resumen sobre las ventajas y desventajas podría ser el siguiente:

## Ventajas

* **Mayor eficiencia para extraer información** relacionada.
* **Estructuración y jerarquía de datos**.
* Respeto de la **integridad de los datos**.
* **Operaciones atómicas** con cambios que afecta a múltiples entidades de la base de datos al mismo tiempo.

## Desventajas
* **Escalabilidad vertical**, se incrementa en hardware potente y caro.
* **Necesidad de mayor procesamiento**.
* **No es flexible**, añadir nuevos datos puede resultar la modificación de esquema o relleno de datos.
* **Herramientas más asentadas y mayor documentación**.

# NoSQL

Los sistemas NoSQL se denominan a veces **"no solo SQL"** para subrayar el hecho de que también pueden soportar lenguajes de consulta de tipo SQL. Los **datos almacenados no requieren estructuras fijas** como tablas, normalmente **no soportan operaciones JOIN**, **ni garantizan completamente ACID** (atomicidad, consistencia, aislamiento y durabilidad) y habitualmente escalan bien horizontalmente. Las bases de datos NoSQL están **altamente optimizadas para las operaciones recuperar y agregar**, y normalmente no ofrecen mucho más que la funcionalidad de almacenar los registros (p.ej. almacenamiento clave-valor). La **pérdida de flexibilidad en tiempo de ejecución**, comparado con los sistemas SQL clásicos, se ve compensada por **ganancias significativas en escalabilidad y rendimiento** cuando se trata con ciertos modelos de datos.<cite> [^3]</cite>
[^3]: [Wikipedia, NoSQL](https://es.wikipedia.org/wiki/NoSQL)

Por último, un resumen con las ventajas y desventajas podría ser el siguiente:

## Ventajas
* **Flexibilidad** al añadir nuevos datos.
* **Carácter descentralizado**, soporta estructuras distribuidas.
* **Escalabilidad horizontal**, se crece en número de máquinas en vez de hardware más potente.
* Ejecución en **máquinas con pocos recursos**.

## Desventajas
* **No se permiten consultas complejas**.
* Posibilidad de **inconsistencia en los datos**.

# Conclusiones

Las bases de datos SQL y NoSQL tienen características distintas y conviene estudiar cual se adapta mejor a las necesidades de cada proyecto. Se recomienda evaluar las diferentes ventajas y desventajas de cada sistema de gestión de base de datos.
