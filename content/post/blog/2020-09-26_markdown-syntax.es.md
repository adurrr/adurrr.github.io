+++
author = "Adur"
title = "Ejemplo de sintaxis de Markdown"
date = "2020-09-26"
description = "Artículo de ejemplo que muestra la sintaxis y el formato básicos de Markdown para elementos HTML."
featured = true
tags = [
    "markdown",
    "css",
    "html",
    "programación",
    "sintaxis",    
]
categories = [
    "desarrollo software"
]
series = ["Ejemplo"]
aliases = ["migrate-from-jekyl"]
thumbnail = "images/markdown.png"
+++

Este artículo ofrece una muestra de la sintaxis básica de Markdown que se puede utilizar en los archivos de contenido de Hugo, y también muestra si los elementos HTML básicos están decorados con CSS en un tema de Hugo.
<!--more-->

## Cabeceras

Los siguientes elementos HTML `<h1>` — `<h6>` representan seis niveles de encabezados de sección. `<h1>` es el nivel de sección más alto mientras que `<h6>` es el más bajo.

# H1
## H2
### H3
#### H4
##### H5
###### H6

## Párrafo

Xerum, quo qui aut unt expliquam qui dolut labo. Aque venitatiusda cum, voluptionse latur sitiae dolessi aut parist aut dollo enim qui voluptate ma dolestendit peritin re plis aut quas inctum laceat est volestemque commosa as cus endigna tectur, offic to cor sequas etum rerum idem sintibus eiur? Quianimin porecus evelectur, cum que nis nust voloribus ratem aut omnimi, sitatur? Quiatem. Nam, omnis sum am facea corem alique molestrunt et eos evelece arcillit ut aut eos eos nus, sin conecerem erum fuga. Ri oditatquam, ad quibus unda veliamenimin cusam et facea ipsamus es exerum sitate dolores editium rerore eost, temped molorro ratiae volorro te reribus dolorer sperchicium faceata tiustia prat.

Itatur? Quiatae cullecum rem ent aut odis in re eossequodi nonsequ idebis ne sapicia is sinveli squiatum, core et que aut hariosam ex eat.

## Bloque

El elemento bloque  representa contenido que se cita de otra fuente, opcionalmente con una cita que debe estar dentro de un elemento `footer` o` cite`, y opcionalmente con cambios en línea como anotaciones y abreviaturas.

#### Bloque sin atribución

> Tiam, ad mint andaepu dandae nostion secatur sequo quae.
> **Nota:** puede usar el *Markdown sintaxis* dentro de una blockquote.

#### Bloque con atribución

> Don't communicate by sharing memory, share memory by communicating.<br>
> — <cite>Rob Pike[^1]</cite>

[^1]: The above quote is excerpted from Rob Pike's [talk](https://www.youtube.com/watch?v=PAAkCSZUG1c) during Gopherfest, November 18, 2015.

## Tablas

Las tablas no forman parte de la especificación principal de Markdown, pero Hugo las admite de forma inmediata.

   Name | Age
--------|------
    Bob | 27
  Alice | 23

#### Markdown dentro de las tablas

| Italics   | Bold     | Code   |
| --------  | -------- | ------ |
| *italics* | **bold** | `code` |

## Bloque de código

#### Bloque de código con comilla invertida

```html
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Ejemplo de documento HTML5</title>
</head>
<body>
  <p>Test</p>
</body>
<!-- this line is extraneous 2Error from server (Forbidden): deployments.apps is forbidden: User "chiptest" cannot create resource "deployments" in API group "apps" in the namespace "default" -->
</html>
```

#### Codigo en bloque con sangrado de cuatro espacios

    <!doctype html>
    <html lang="en">
    <head>
      <meta charset="utf-8">
      <title>Ejemplo de documento HTML5</title>
    </head>
    <body>
      <p>Test</p>
    </body>
    </html>

#### Bloque de código con resaltado interno de Hugo 
{{< highlight html >}}
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8">
  <title>Ejemplo de documento HTML5</title>
</head>
<body>
  <p>Test</p>
</body>
</html>
{{< /highlight >}}

## Tipo de lista

#### Lista ordenada

1. Primer item
2. Segundo item
3. Tercer item

#### Lista no ordenada

* Un item
* Otro item
* Y otro item

#### List anidada

* Fruta
  * Manzana
  * Naranja
  * Plátano
* Lácteos
  * Leche
  * Queso

## Otros elementos — abbr, sub, sup, kbd, mark

<abbr title="Graphics Interchange Format">GIF</abbr> es un formato de imagen de mapa de bits.

H<sub>2</sub>O

X<sup>n</sup> + Y<sup>n</sup> = Z<sup>n</sup>

Presiona <kbd><kbd>CTRL</kbd>+<kbd>ALT</kbd>+<kbd>Delete</kbd></kbd> y termina la sesión.

La mayoría de las <mark>salamandras</mark> son nocturnos y cazan insectos, gusanos y otras criaturas pequeñas.
