+++
author = "Adur"
title = "Seguridad en la cadena de suministro: SBOM y Sigstore"
date = "2025-02-10"
description = "Guía práctica sobre seguridad en la cadena de suministro de software usando SBOM, Sigstore y el framework SLSA."
featured = true
tags = [
    "sbom",
    "sigstore",
    "cosign",
    "cadena-de-suministro",
    "seguridad",
    "slsa",
]
categories = [
    "Seguridad",
    "devops",
]
series = ["Security"]
thumbnail = "images/seguridad.png"
toc = true
+++

Los ataques de cadena de suministro ya no son teóricos. La brecha SolarWinds de 2020 mostró cómo un pipeline de compilación comprometido podía golpear miles de organizaciones, incluyendo agencias gubernamentales. Log4Shell en 2021 probó que una vulnerabilidad profunda en una dependencia transitiva podía amenazar todas las apps Java de la noche a la mañana. El mensaje fue claro: necesitamos visibilidad sobre qué hay en nuestro software y garantías más fuertes de integridad.

Esta guía cubre las herramientas prácticas: SBOMs, Sigstore y framework SLSA.

<!--more-->

## Por qué importa la seguridad de la cadena de suministro

La seguridad tradicional se centra en tu propio código: análisis estático, escaneo de dependencias, pruebas de penetración. La seguridad de la cadena de suministro extiende ese perímetro a todo de lo que depende tu software y a cada paso en el proceso de construcción y entrega.

La superficie de ataque incluye:

- **Repositorios de código fuente**: cuentas de desarrolladores comprometidas, commits maliciosos
- **Dependencias**: typosquatting, confusión de dependencias, paquetes upstream comprometidos
- **Sistemas de compilación**: pipelines de CI/CD manipulados, pasos de build inyectados
- **Registros de artefactos**: binarios reemplazados, paquetes sin firmar
- **Pipelines de despliegue**: manifiestos modificados, ataques man-in-the-middle

Un solo eslabón débil en cualquiera de estas etapas puede comprometer toda la cadena.

## Software Bill of Materials (SBOM)

Un SBOM es un inventario formal y legible por máquinas de todos los componentes de un software. Piensa en ello como la lista de ingredientes de tu aplicación. Incluye dependencias directas, dependencias transitivas, sus versiones, licencias y relaciones.

### Por qué lo necesitas

- **Respuesta ante vulnerabilidades**: Cuando aparece un nuevo CVE (como Log4Shell), puedes comprobar al instante si alguna de tus aplicaciones está afectada.
- **Cumplimiento de licencias**: Saber exactamente qué licencias estás distribuyendo.
- **Requisitos regulatorios**: La Orden Ejecutiva 14028 de EE.UU. y la Ley de Ciberresiliencia de la UE empujan hacia SBOMs obligatorios.
- **Transparencia**: Clientes y partners pueden verificar lo que están ejecutando.

### Formatos de SBOM

Dos formatos principales dominan:

- **SPDX** (Software Package Data Exchange): Un estándar ISO (ISO/IEC 5962:2021), originalmente centrado en cumplimiento de licencias, ahora integral. Soporta formatos JSON, RDF, YAML y tag-value.
- **CycloneDX**: Un proyecto de OWASP, diseñado desde cero para casos de uso de seguridad. Soporta JSON y XML. Más ligero y con opiniones más definidas.

Ambos son opciones sólidas. CycloneDX tiende a ser más fácil de usar para flujos de trabajo centrados en seguridad. SPDX tiene mayor adopción en industrias con requisitos de cumplimiento estrictos.

### Generando SBOMs con Syft

[Syft](https://github.com/anchore/syft) de Anchore es una de las mejores herramientas para generar SBOMs. Soporta imágenes de contenedores, sistemas de archivos y archivos comprimidos.

**Instalar syft:**

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
```

**Generar un SBOM desde una imagen de contenedor:**

```bash
# Formato CycloneDX (JSON)
syft packages registry.example.com/myapp:v1.2.3 -o cyclonedx-json > sbom.cdx.json

# Formato SPDX (JSON)
syft packages registry.example.com/myapp:v1.2.3 -o spdx-json > sbom.spdx.json
```

**Generar un SBOM desde un directorio local:**

```bash
syft packages dir:/path/to/project -o cyclonedx-json > sbom.cdx.json
```

Después puedes escanear el SBOM en busca de vulnerabilidades usando [Grype](https://github.com/anchore/grype):

```bash
grype sbom:sbom.cdx.json
```

## El ecosistema Sigstore

[Sigstore](https://www.sigstore.dev/) es un proyecto open source que hace accesible la firma y verificación criptográfica. Elimina la necesidad de gestionar claves de firma de larga duración, que históricamente ha sido la principal barrera para la adopción de la firma de artefactos.

El ecosistema tiene tres componentes centrales:

- **cosign**: Firma y verifica imágenes de contenedores y otros artefactos OCI.
- **Fulcio**: Una autoridad certificadora que emite certificados de corta duración basados en identidad OIDC (tu proveedor de identidad existente).
- **Rekor**: Un log de transparencia que crea un registro inmutable y resistente a manipulaciones de los eventos de firma.

### Cómo funciona

1. Te autentificas con un proveedor OIDC (GitHub, Google, Microsoft, etc.).
2. Fulcio emite un certificado de corta duración vinculado a tu identidad.
3. cosign usa ese certificado para firmar tu artefacto.
4. El evento de firma se registra en el log de transparencia de Rekor.
5. Cualquiera puede verificar la firma usando el log de transparencia, sin necesitar tu clave pública.

Esto se conoce como firma "keyless". Sin claves que rotar, sin secretos que gestionar, sin infraestructura PKI que mantener.

### Firmando imágenes de contenedores con Cosign

**Instalar cosign:**

```bash
# Usando Go
go install github.com/sigstore/cosign/v2/cmd/cosign@latest

# O descargar un release
curl -sSfL https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64 -o /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign
```

**Firmar una imagen (modo keyless):**

```bash
cosign sign registry.example.com/myapp:v1.2.3
```

Esto abrirá un navegador para autenticación OIDC. En CI, puedes usar identidad de carga de trabajo (por ejemplo, el token OIDC de GitHub Actions) para firma no interactiva.

**Verificar una imagen:**

```bash
cosign verify registry.example.com/myapp:v1.2.3 \
  --certificate-identity=user@example.com \
  --certificate-oidc-issuer=https://accounts.google.com
```

**Adjuntar un SBOM a una imagen y firmarlo:**

```bash
# Adjuntar el SBOM
cosign attach sbom --sbom sbom.cdx.json registry.example.com/myapp:v1.2.3

# Firmar el adjunto del SBOM
cosign sign --attachment sbom registry.example.com/myapp:v1.2.3
```

## El framework SLSA

[SLSA](https://slsa.dev/) (Supply-chain Levels for Software Artifacts, pronunciado "salsa") es un framework que define niveles crecientes de garantías de integridad en la cadena de suministro.

### Niveles SLSA

- **Level 0**: Sin garantías. Aquí es donde empiezan la mayoría de proyectos.
- **Level 1**: El proceso de compilación está documentado y produce provenance (metadatos sobre cómo se construyó un artefacto).
- **Level 2**: La compilación se ejecuta en un servicio de build alojado que genera provenance autenticado.
- **Level 3**: La plataforma de compilación proporciona builds reforzados con provenance resistente a manipulaciones. El entorno de build es aislado y efímero.

Cada nivel se construye sobre el anterior. El objetivo no es saltar al Level 3 inmediatamente, sino mejorar tu postura de forma incremental.

### SLSA Provenance

El provenance responde a las preguntas críticas: Quién construyó esto? Qué código fuente se usó? Qué proceso de build se siguió? Era el entorno de build a prueba de manipulaciones?

El SLSA provenance es una attestation firmada en formato [in-toto](https://in-toto.io/) que captura esta información.

## Integración CI/CD: Ejemplo con GitHub Actions

Aquí tienes un workflow práctico de GitHub Actions que construye una imagen, genera un SBOM, firma todo y genera SLSA provenance:

```yaml
name: Build, Sign, and Attest

on:
  push:
    tags:
      - 'v*'

permissions:
  contents: read
  packages: write
  id-token: write  # Requerido para firma keyless

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-sign-attest:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push image
        id: build
        uses: docker/build-push-action@v5
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}

      - name: Install cosign
        uses: sigstore/cosign-installer@v3

      - name: Install syft
        uses: anchore/sbom-action/download-syft@v0

      - name: Sign the image
        run: |
          cosign sign --yes \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}

      - name: Generate SBOM
        run: |
          syft packages \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }} \
            -o cyclonedx-json > sbom.cdx.json

      - name: Attach and sign SBOM
        run: |
          cosign attach sbom --sbom sbom.cdx.json \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}
          cosign sign --yes --attachment sbom \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}

      - name: Verify signature
        run: |
          cosign verify \
            --certificate-identity-regexp="https://github.com/${{ github.repository }}/*" \
            --certificate-oidc-issuer=https://token.actions.githubusercontent.com \
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}@${{ steps.build.outputs.digest }}
```

El permiso `id-token: write` es lo que habilita la firma keyless en GitHub Actions. El token OIDC de GitHub se usa automáticamente por cosign sin ninguna gestión manual de claves.

## Hoja de ruta práctica para la adopción

No necesitas hacer todo de golpe. Aquí tienes una progresión sensata:

**Semana 1-2: Visibilidad**
- Empieza a generar SBOMs para tus aplicaciones más críticas usando syft.
- Integra Grype en tu pipeline de CI para escaneo de vulnerabilidades contra el SBOM.

**Semana 3-4: Firma**
- Configura la firma keyless con cosign en tus pipelines de CI/CD.
- Firma tus imágenes de contenedores en cada build.

**Mes 2: Verificación**
- Aplica verificación de firmas en tus pipelines de despliegue (por ejemplo, controladores de admisión de Kubernetes como Kyverno o Sigstore policy-controller).
- Adjunta SBOMs a las imágenes y fírmalos.

**Mes 3+: SLSA Provenance**
- Añade generación de SLSA provenance usando [slsa-github-generator](https://github.com/slsa-framework/slsa-github-generator).
- Trabaja hacia SLSA Level 2, después Level 3.
- Automatiza la verificación de provenance en tu tooling de despliegue.

## Conclusiones clave

- **SBOMs dan visibilidad** - No aseguras lo que no ves. Genera SBOMs para cada artefacto.
- **Sigstore elimina excusas** - La firma keyless elimina la sobrecarga de gestión de claves. Sin buena razón para no firmar.
- **SLSA es un modelo de madurez** - Úsalo para mejorar la integridad de forma incremental, no como todo o nada.
- **Automatiza todo** - Estas herramientas son para integración CI/CD. Lo manual no escala.

El ecosistema de seguridad de cadena de suministro madura rápido. Las herramientas están listas para producción, los estándares se consolidan, la presión regulatoria sigue subiendo. El mejor momento fue ayer. El segundo mejor es ahora.
