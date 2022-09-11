+++
author = "Adur"
title = "Infraestructura como código con Terraform: guía práctica"
date = "2022-09-10"
description = "Una guía práctica de Terraform que cubre conceptos fundamentales, ejemplos prácticos, gestión de estado y buenas prácticas para producción."
featured = true
tags = [
    "terraform",
    "iac",
    "infraestructura",
    "hcl",
    "automatización"
]
categories = [
    "devops"
]
series = ["DevOps"]
thumbnail = "images/devops.png"
toc = true
+++

## Por qué importa la infraestructura como código

Gestionar infraestructura manualmente a través de consolas web o scripts ad-hoc crea problemas que se acumulan con el tiempo: entornos inconsistentes, cambios sin documentar, rollbacks imposibles y el clásico "funciona en mi máquina" extendido a servidores enteros.

La Infraestructura como Código (IaC) resuelve esto tratando la infraestructura como el código de aplicación: se escribe, versiona, revisa, prueba y aplica mediante flujos de trabajo automatizados. Los beneficios llegan rápidamente:

- **Reproducibilidad**: Levanta entornos idénticos en minutos, no en días.
- **Control de versiones**: Cada cambio de infraestructura pasa por un PR con revisión de código.
- **Documentación por defecto**: El código *es* la documentación de cómo es tu infraestructura.
- **Recuperación ante desastres**: Reconstruye todo desde código si una región cae.
- **Visibilidad de costes**: Revisa los cambios de infraestructura antes de aplicarlos (y antes de que empiecen a costar dinero).

## Terraform vs otras herramientas

Existen varias herramientas de IaC. Así es como Terraform se compara con las principales alternativas:

| Característica | Terraform | Pulumi | CloudFormation | Ansible |
|---|---|---|---|---|
| **Lenguaje** | HCL (declarativo) | Python, TypeScript, Go, etc. | JSON/YAML | YAML (procedural) |
| **Soporte cloud** | Multi-cloud | Multi-cloud | Solo AWS | Multi-cloud (vía módulos) |
| **Gestión de estado** | Fichero de estado explícito | Gestionado por servicio Pulumi | Gestionado por AWS | Sin estado |
| **Curva de aprendizaje** | Moderada | Varía según lenguaje | Moderada | Baja |
| **Ecosistema** | Enorme ecosistema de providers | En crecimiento | Solo AWS pero profundo | Enorme ecosistema de roles |
| **Ideal para** | Infra multi-cloud | Equipos que prefieren lenguajes de propósito general | Entornos solo AWS | Gestión de configuración |

El punto fuerte de Terraform es el aprovisionamiento de infraestructura multi-cloud con un enfoque declarativo. Si estás solo en AWS y quieres integración estrecha, CloudFormation es razonable. Si tu equipo prefiere escribir Python en lugar de HCL, Pulumi merece una mirada. Pero para la mayoría de equipos que gestionan infraestructura entre distintos proveedores, Terraform es la elección pragmática.

## Conceptos fundamentales

### Providers

Los providers son plugins que permiten a Terraform interactuar con APIs — AWS, Azure, GCP, Kubernetes, GitHub, Cloudflare y cientos más.

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}
```

### Resources

Los resources son los bloques de construcción fundamentales. Cada bloque resource describe un objeto de infraestructura.

```hcl
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "web-server"
  }
}
```

### State

Terraform mantiene un **fichero de estado** que mapea tu configuración a recursos del mundo real. Así es como Terraform sabe qué existe, qué necesita cambiar y qué destruir. El fichero de estado es crítico. Perderlo significa que Terraform pierde la pista de tu infraestructura.

### Modules

Los modules son paquetes reutilizables de configuración Terraform. Piensa en ellos como funciones: reciben entradas (variables), crean recursos y producen salidas.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.1.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["eu-west-1a", "eu-west-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
}
```

## Ejemplo práctico: VPC + EC2

Aquí hay un ejemplo completo que aprovisiona una VPC con una subred pública y una instancia EC2:

```hcl
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "eu-west-1"
}

# --- Networking ---

resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "eu-west-1a"
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet"
  }
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.gw.id
  }

  tags = {
    Name = "public-rt"
  }
}

resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.public.id
}

# --- Security Group ---

resource "aws_security_group" "web" {
  name        = "web-sg"
  description = "Allow HTTP and SSH"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["YOUR_IP/32"]  # Restringir a tu IP
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# --- EC2 Instance ---

resource "aws_instance" "web" {
  ami                    = "ami-0c55b159cbfafe1f0"
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.web.id]

  tags = {
    Name = "web-server"
  }
}

# --- Outputs ---

output "instance_public_ip" {
  value = aws_instance.web.public_ip
}

output "vpc_id" {
  value = aws_vpc.main.id
}
```

## El flujo plan/apply

Terraform sigue un flujo de trabajo predecible:

```bash
# 1. Inicializar - descargar providers y modules
terraform init

# 2. Formatear - asegurar estilo de código consistente
terraform fmt

# 3. Validar - comprobar sintaxis y configuración
terraform validate

# 4. Plan - previsualizar qué cambiará (¡paso crítico!)
terraform plan -out=tfplan

# 5. Apply - ejecutar el plan
terraform apply tfplan

# 6. Destroy - destruir todos los recursos (cuando sea necesario)
terraform destroy
```

El paso `terraform plan` es el más importante. Nunca te lo saltes. Siempre revisa la salida del plan antes de aplicar, especialmente en producción. El plan te muestra exactamente qué se creará, modificará o destruirá.

```bash
# Ejemplo de salida del plan
Plan: 6 to add, 0 to change, 0 to destroy.
```

En pipelines CI/CD, guarda el plan en un fichero (`-out=tfplan`) y aplica ese plan exacto. Esto previene condiciones de carrera donde la infraestructura cambia entre el plan y el paso de apply.

## Buenas prácticas de gestión de estado

La gestión de estado es donde se originan la mayoría de problemas con Terraform. Sigue estas prácticas:

### Usa un backend remoto

Nunca almacenes el estado localmente o en Git. Usa un backend remoto con cifrado y bloqueo:

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"
    key            = "prod/networking/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

La tabla DynamoDB proporciona **bloqueo de estado**. Esto previene que dos personas o pipelines modifiquen la misma infraestructura al mismo tiempo.

### Organiza el estado por componente

No pongas toda tu infraestructura en un solo fichero de estado. Divídela por componente o equipo:

```
environments/
├── prod/
│   ├── networking/    # VPC, subnets, routes
│   ├── compute/       # EC2, ASGs, load balancers
│   ├── database/      # Instancias RDS
│   └── monitoring/    # CloudWatch, alertas
└── staging/
    ├── networking/
    ├── compute/
    └── database/
```

Ficheros de estado más pequeños significan planes más rápidos, menor radio de explosión y menos equipos compitiendo por los bloqueos.

### Usa `terraform_remote_state` con moderación

Puedes referenciar outputs de otros ficheros de estado, pero úsalo con cuidado. El uso excesivo de remote state crea acoplamiento fuerte entre componentes. Prefiere pasar valores a través de variables o un parameter store.

## Consejos para uso en producción

1. **Fija las versiones de los providers.** Usa restricciones `~>` para permitir actualizaciones de parche pero prevenir cambios incompatibles: `version = "~> 5.0"`.

2. **Usa workspaces con cuidado.** Los workspaces son útiles para separación simple de entornos pero se vuelven confusos a escala. Directorios separados por entorno suele ser más claro.

3. **Implementa un pipeline CI/CD para Terraform.** Ejecuta `terraform plan` en PRs y publica la salida como comentario en el PR. Ejecuta `terraform apply` solo después del merge y la aprobación.

4. **Usa `prevent_destroy` para recursos críticos.** Esta regla de lifecycle evita la destrucción accidental de bases de datos o almacenamiento persistente:

    ```hcl
    resource "aws_db_instance" "main" {
      # ...
      lifecycle {
        prevent_destroy = true
      }
    }
    ```

5. **Etiqueta todo.** Usa un bloque `default_tags` en el provider para asegurar que cada recurso recibe tags estándar (entorno, equipo, proyecto).

6. **Usa `tflint` y `checkov`.** Haz lint de tu código Terraform y escanea en busca de configuraciones inseguras antes de aplicar.

    ```bash
    tflint --init
    tflint
    checkov -d .
    ```

7. **Importa recursos existentes.** Si tienes infraestructura creada manualmente, usa `terraform import` para traerla bajo gestión en lugar de recrearla.

8. **Revisa el diff del plan cuidadosamente.** Un recurso que muestra "destroy and recreate" puede causar tiempo de inactividad. Entiende qué cambios son in-place versus destructivos.

Terraform es una de esas herramientas que recompensa la disciplina. Cuanto más consistentemente sigas estas prácticas, con más confianza gestionará tu equipo la infraestructura a escala.
