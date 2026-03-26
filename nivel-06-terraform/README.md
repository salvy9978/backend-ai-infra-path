# Nivel 06 — Terraform: Infrastructure as Code

> Crear servidores a mano en la consola web es lento, propenso a errores e imposible de auditar. Terraform te permite definir toda tu infraestructura en archivos de texto, versionarla en Git y aplicar cambios de forma predecible y repetible.

---

## Contenido

- [1. Infrastructure as Code](#1-infrastructure-as-code)
  - [1.1 Filosofia y por que importa](#11-filosofia-y-por-que-importa)
  - [1.2 Terraform vs alternativas](#12-terraform-vs-alternativas)
  - [1.3 Como funciona Terraform](#13-como-funciona-terraform)
- [2. Lenguaje HCL](#2-lenguaje-hcl)
  - [2.1 Sintaxis basica](#21-sintaxis-basica)
  - [2.2 Tipos de datos](#22-tipos-de-datos)
  - [2.3 Expresiones y funciones](#23-expresiones-y-funciones)
- [3. Providers](#3-providers)
  - [3.1 Que son los providers](#31-que-son-los-providers)
  - [3.2 Configuracion de AWS](#32-configuracion-de-aws)
  - [3.3 Configuracion de GCP](#33-configuracion-de-gcp)
  - [3.4 Multiples providers](#34-multiples-providers)
- [4. Resources y Data Sources](#4-resources-y-data-sources)
  - [4.1 Resources](#41-resources)
  - [4.2 Data Sources](#42-data-sources)
  - [4.3 Dependencias implicitas y explicitas](#43-dependencias-implicitas-y-explicitas)
  - [4.4 Ciclo de vida de recursos](#44-ciclo-de-vida-de-recursos)
- [5. El ciclo init/plan/apply/destroy](#5-el-ciclo-initplanapplydestroy)
  - [5.1 terraform init](#51-terraform-init)
  - [5.2 terraform plan](#52-terraform-plan)
  - [5.3 terraform apply](#53-terraform-apply)
  - [5.4 terraform destroy](#54-terraform-destroy)
  - [5.5 terraform fmt y validate](#55-terraform-fmt-y-validate)
- [6. State management](#6-state-management)
  - [6.1 Que es el state](#61-que-es-el-state)
  - [6.2 State local vs remoto](#62-state-local-vs-remoto)
  - [6.3 Backend S3 con DynamoDB](#63-backend-s3-con-dynamodb)
  - [6.4 Backend GCS](#64-backend-gcs)
  - [6.5 State locking](#65-state-locking)
- [7. Variables](#7-variables)
  - [7.1 Input variables](#71-input-variables)
  - [7.2 Output values](#72-output-values)
  - [7.3 Local values](#73-local-values)
  - [7.4 terraform.tfvars](#74-terraformtfvars)
- [8. Modulos](#8-modulos)
  - [8.1 Para que sirven](#81-para-que-sirven)
  - [8.2 Estructura de un modulo](#82-estructura-de-un-modulo)
  - [8.3 Crear un modulo](#83-crear-un-modulo)
  - [8.4 Usar un modulo](#84-usar-un-modulo)
  - [8.5 Modulos del registry](#85-modulos-del-registry)
- [9. Workspaces](#9-workspaces)
- [10. Import de recursos existentes](#10-import-de-recursos-existentes)
- [11. Resumen](#11-resumen)
- [12. Ejercicios](#12-ejercicios)
- [13. Recursos](#13-recursos)

---

## 1. Infrastructure as Code

### 1.1 Filosofia y por que importa

Infrastructure as Code (IaC) significa gestionar infraestructura (servidores, redes, bases de datos, DNS) mediante archivos de definicion en lugar de procesos manuales.

Los beneficios son directos:

```
SIN IaC:                                 CON IaC:

1. Abrir consola AWS                     1. Editar main.tf
2. Hacer clic en "Create Instance"       2. git commit -m "add server"
3. Rellenar formulario                   3. terraform apply
4. Repetir en staging y prod             4. (mismo archivo, distintas vars)
5. Documentar en Confluence              5. (el codigo ES la documentacion)
6. Rezar para que coincida               6. (siempre coincide)
```

Principios clave:
- **Declarativo**: describes el estado deseado, no los pasos para llegar
- **Versionado**: los archivos van en Git con historial completo
- **Idempotente**: aplicar dos veces produce el mismo resultado
- **Reproducible**: el mismo codigo crea la misma infra en cualquier cuenta/region

### 1.2 Terraform vs alternativas

| Herramienta | Proveedor | Lenguaje | Enfoque |
|-------------|-----------|----------|---------|
| **Terraform** | Multi-cloud | HCL | Declarativo, state externo |
| **CloudFormation** | Solo AWS | JSON/YAML | Declarativo, state en AWS |
| **Pulumi** | Multi-cloud | Python/TS/Go | Imperativo con lenguajes reales |
| **Ansible** | Multi-cloud | YAML | Imperativo, sin state |
| **CDK** | AWS (o multi con cdktf) | TS/Python/Java | Imperativo, genera CloudFormation |

Terraform destaca por ser multi-cloud, tener un ecosistema enorme de providers (AWS, GCP, Azure, Kubernetes, Cloudflare, GitHub...) y una comunidad masiva.

### 1.3 Como funciona Terraform

```
                    terraform plan
Archivos .tf --------------------------> Plan de cambios
     |                                        |
     |         terraform apply                |
     +-------------------------------------->  |
                                               v
                                     API del Provider (AWS/GCP)
                                               |
                                               v
                                     Infraestructura real
                                               |
                                               v
                                     terraform.tfstate
                                     (registro de lo creado)
```

Terraform compara tres cosas:
1. **Configuracion** (.tf): lo que tu quieres
2. **State** (.tfstate): lo que Terraform cree que existe
3. **Realidad**: lo que realmente existe en el cloud

Del delta entre estas tres fuentes, Terraform calcula que crear, modificar o destruir.

---

## 2. Lenguaje HCL

### 2.1 Sintaxis basica

HCL (HashiCorp Configuration Language) es el lenguaje de Terraform. Es declarativo y legible:

```hcl
# Comentario de una linea

/* Comentario
   multilinea */

# Bloque: tipo "nombre-tipo" "nombre-recurso" { configuracion }
resource "aws_instance" "servidor_web" {
  ami           = "ami-0c55b159cbfafe1f0"   # asignacion con =
  instance_type = "t3.micro"

  tags = {                                   # mapa
    Name        = "web-server"
    Environment = "production"
  }
}
```

Todo en Terraform se organiza en bloques. Los bloques mas comunes:

| Bloque | Proposito | Ejemplo |
|--------|-----------|---------|
| `terraform` | Configuracion de Terraform mismo | version requerida, backend |
| `provider` | Configurar un proveedor cloud | credenciales AWS, region |
| `resource` | Crear un recurso de infraestructura | instancia EC2, bucket S3 |
| `data` | Leer datos existentes | AMI mas reciente, VPC default |
| `variable` | Declarar variable de entrada | tipo de instancia, region |
| `output` | Exportar valores | IP publica del servidor |
| `locals` | Valores calculados reutilizables | nombres con prefijo |
| `module` | Invocar un modulo reutilizable | modulo de red, modulo de base de datos |

### 2.2 Tipos de datos

```hcl
# String
nombre = "mi-servidor"

# Number
puerto = 8080

# Boolean
publico = true

# List (array ordenado)
availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]

# Map (clave-valor)
tags = {
  Name        = "web"
  Environment = "prod"
}

# Object (estructura tipada)
variable "servidor" {
  type = object({
    nombre = string
    cpu    = number
    publica = bool
  })
}

# Tuple (lista con tipos mixtos)
ejemplo = ["hola", 42, true]
```

### 2.3 Expresiones y funciones

```hcl
# Interpolacion de strings
name = "server-${var.environment}-${count.index}"

# Condicional (ternario)
instance_type = var.environment == "production" ? "t3.large" : "t3.micro"

# For expression (generar listas o mapas)
upper_names = [for name in var.names : upper(name)]
name_map    = {for name in var.names : name => upper(name)}

# Funciones comunes
longitud    = length(var.lista)              # longitud de lista/string
unido       = join(",", var.lista)           # unir lista en string
archivo     = file("${path.module}/init.sh") # leer archivo
codificado  = base64encode("hola mundo")     # codificar en base64
formateado  = format("server-%03d", 1)       # "server-001"
buscado     = lookup(var.mapa, "clave", "default")  # buscar en mapa
```

Funciones mas usadas:

| Funcion | Que hace | Ejemplo |
|---------|----------|---------|
| `length()` | Longitud de lista/mapa/string | `length(["a","b"])` = 2 |
| `join()` | Unir lista en string | `join(",", ["a","b"])` = "a,b" |
| `split()` | Dividir string en lista | `split(",", "a,b")` = ["a","b"] |
| `lookup()` | Buscar clave en mapa | `lookup(map, "key", "default")` |
| `file()` | Leer contenido de archivo | `file("script.sh")` |
| `templatefile()` | Renderizar template con variables | `templatefile("init.sh", {port=8080})` |
| `cidrsubnet()` | Calcular subredes CIDR | `cidrsubnet("10.0.0.0/16", 8, 1)` |
| `toset()` | Convertir lista a set (sin duplicados) | `toset(["a","a","b"])` |
| `flatten()` | Aplanar listas anidadas | `flatten([["a"], ["b","c"]])` |
| `try()` | Valor por defecto si falla | `try(var.opcional, "default")` |

---

## 3. Providers

### 3.1 Que son los providers

Un provider es un plugin que permite a Terraform interactuar con una API externa. Cada servicio cloud tiene su provider.

```hcl
# terraform block: define version de Terraform y providers requeridos
terraform {
  required_version = ">= 1.5.0"        # version minima de Terraform

  required_providers {
    aws = {
      source  = "hashicorp/aws"         # registro/organizacion/nombre
      version = "~> 5.0"               # >= 5.0 y < 6.0
    }
  }
}
```

Restricciones de version:
- `= 5.0.0`: exactamente esa version
- `~> 5.0`: mayor o igual a 5.0, menor que 6.0 (patch y minor)
- `>= 5.0, < 5.5`: rango explicito

### 3.2 Configuracion de AWS

```hcl
provider "aws" {
  region = var.aws_region              # region donde crear recursos

  # Credenciales: NUNCA en el codigo. Usar variables de entorno o profiles
  # export AWS_ACCESS_KEY_ID="..."
  # export AWS_SECRET_ACCESS_KEY="..."
  # o usar el archivo ~/.aws/credentials

  default_tags {                       # tags automaticos en todos los recursos
    tags = {
      ManagedBy   = "terraform"
      Project     = var.project_name
      Environment = var.environment
    }
  }
}
```

### 3.3 Configuracion de GCP

```hcl
provider "google" {
  project = var.gcp_project_id
  region  = var.gcp_region

  # Credenciales: usar gcloud auth application-default login
  # o la variable GOOGLE_APPLICATION_CREDENTIALS con ruta al JSON
}
```

### 3.4 Multiples providers

Puedes usar multiples providers o multiples configuraciones del mismo provider:

```hcl
# Provider default para us-east-1
provider "aws" {
  region = "us-east-1"
}

# Provider alias para eu-west-1
provider "aws" {
  alias  = "europa"
  region = "eu-west-1"
}

# Recurso en us-east-1 (provider default)
resource "aws_s3_bucket" "bucket_us" {
  bucket = "mi-bucket-us"
}

# Recurso en eu-west-1 (provider alias)
resource "aws_s3_bucket" "bucket_eu" {
  provider = aws.europa                # referencia al alias
  bucket   = "mi-bucket-eu"
}
```

---

## 4. Resources y Data Sources

### 4.1 Resources

Un resource representa un componente de infraestructura que Terraform crea y gestiona:

```hcl
# Sintaxis: resource "tipo_provider_recurso" "nombre_local" { ... }
resource "aws_instance" "web" {
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = var.instance_type
  subnet_id     = aws_subnet.publica.id  # referencia a otro recurso

  root_block_device {                    # bloque anidado
    volume_size = 20
    volume_type = "gp3"
    encrypted   = true
  }

  user_data = templatefile("${path.module}/init.sh", {
    db_host = aws_db_instance.main.address
  })

  tags = {
    Name = "web-server"
  }
}

# Referencia: aws_instance.web.public_ip
# Referencia: aws_instance.web.id
```

Crear multiples recursos identicos con `count`:

```hcl
resource "aws_instance" "worker" {
  count         = var.worker_count       # crea N instancias
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t3.micro"

  tags = {
    Name = "worker-${count.index}"       # worker-0, worker-1, worker-2
  }
}

# Referencia: aws_instance.worker[0].public_ip
# Referencia: aws_instance.worker[*].public_ip  (lista de todas)
```

Crear multiples recursos con `for_each` (mas flexible que count):

```hcl
variable "servidores" {
  default = {
    web = { type = "t3.small", zona = "us-east-1a" }
    api = { type = "t3.medium", zona = "us-east-1b" }
    worker = { type = "t3.large", zona = "us-east-1c" }
  }
}

resource "aws_instance" "servers" {
  for_each      = var.servidores
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = each.value.type
  availability_zone = each.value.zona

  tags = {
    Name = each.key                      # "web", "api", "worker"
  }
}

# Referencia: aws_instance.servers["web"].public_ip
```

### 4.2 Data Sources

Un data source lee informacion de recursos que ya existen (no los crea):

```hcl
# Leer la AMI mas reciente de Amazon Linux 2023
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

# Leer la VPC default
data "aws_vpc" "default" {
  default = true
}

# Leer subnets de la VPC default
data "aws_subnets" "default" {
  filter {
    name   = "vpc-id"
    values = [data.aws_vpc.default.id]
  }
}

# Usar en un recurso
resource "aws_instance" "web" {
  ami       = data.aws_ami.amazon_linux.id    # AMI del data source
  subnet_id = data.aws_subnets.default.ids[0] # primera subnet
}
```

### 4.3 Dependencias implicitas y explicitas

**Implicita**: Terraform detecta dependencias automaticamente cuando un recurso referencia a otro:

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "publica" {
  vpc_id     = aws_vpc.main.id          # dependencia implicita: crea VPC primero
  cidr_block = "10.0.1.0/24"
}
```

**Explicita**: cuando no hay referencia directa pero el orden importa:

```hcl
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t3.micro"

  depends_on = [aws_iam_role_policy.web_policy]  # crear policy antes
}
```

### 4.4 Ciclo de vida de recursos

```hcl
resource "aws_instance" "web" {
  ami           = "ami-abc123"
  instance_type = "t3.micro"

  lifecycle {
    # Crear el nuevo antes de destruir el viejo (zero downtime)
    create_before_destroy = true

    # Prevenir destruccion accidental
    prevent_destroy = true

    # Ignorar cambios en ciertos atributos (util para tags que otros modifican)
    ignore_changes = [tags, user_data]

    # Reemplazar recurso si cambia algo especifico
    replace_triggered_by = [aws_ami.web.id]
  }
}
```

---

## 5. El ciclo init/plan/apply/destroy

### 5.1 terraform init

Inicializa el directorio de trabajo. Descarga providers y configura el backend:

```bash
# Inicializar proyecto (ejecutar la primera vez o cuando cambian providers/backend)
terraform init

# Reinicializar forzando reconfiguracion del backend
terraform init -reconfigure

# Migrar state a un nuevo backend
terraform init -migrate-state

# Solo actualizar providers a la ultima version compatible
terraform init -upgrade
```

Genera un directorio `.terraform/` con los binarios de los providers y un `.terraform.lock.hcl` con las versiones exactas (commitear este archivo a Git).

### 5.2 terraform plan

Muestra que cambios se aplicarian sin ejecutarlos. Es el paso de revision:

```bash
# Plan normal
terraform plan

# Plan con archivo de variables
terraform plan -var-file="production.tfvars"

# Plan guardado en archivo (para apply posterior)
terraform plan -out=plan.tfplan

# Plan enfocado en un recurso
terraform plan -target=aws_instance.web
```

La salida usa convenciones de color:
- `+` verde: crear recurso nuevo
- `~` amarillo: modificar recurso existente (in-place)
- `-/+` rojo/verde: destruir y recrear (forces replacement)
- `-` rojo: destruir recurso

```
Terraform will perform the following actions:

  # aws_instance.web will be created
  + resource "aws_instance" "web" {
      + ami           = "ami-0c55b159cbfafe1f0"
      + instance_type = "t3.micro"
      + id            = (known after apply)
      + public_ip     = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### 5.3 terraform apply

Ejecuta los cambios. Pide confirmacion a menos que uses `-auto-approve`:

```bash
# Apply interactivo (pide confirmacion)
terraform apply

# Apply sin confirmacion (para CI/CD)
terraform apply -auto-approve

# Apply desde un plan guardado (no pide confirmacion)
terraform apply plan.tfplan

# Apply con variables
terraform apply -var="instance_type=t3.large"
```

**En CI/CD**, el flujo recomendado es:

```bash
terraform init
terraform plan -out=plan.tfplan        # guardar plan
# (revision humana del plan)
terraform apply plan.tfplan            # aplicar exactamente ese plan
```

### 5.4 terraform destroy

Destruye toda la infraestructura gestionada:

```bash
# Destruir todo (pide confirmacion)
terraform destroy

# Destruir sin confirmacion
terraform destroy -auto-approve

# Destruir solo un recurso especifico
terraform destroy -target=aws_instance.web
```

**Gotcha**: `terraform destroy` en produccion puede causar desastres. Usa `prevent_destroy` en recursos criticos y configura approval gates en CI/CD.

### 5.5 terraform fmt y validate

```bash
# Formatear archivos HCL (como prettier para Terraform)
terraform fmt                          # formatea archivos en el directorio actual
terraform fmt -recursive               # incluye subdirectorios
terraform fmt -check                   # solo verifica, no modifica (para CI)

# Validar sintaxis y logica basica
terraform validate                     # valida configuracion sin conectar al cloud
```

Incluir ambos en CI:

```yaml
# En GitHub Actions
- name: Terraform Format
  run: terraform fmt -check -recursive
- name: Terraform Validate
  run: terraform validate
```

---

## 6. State management

### 6.1 Que es el state

El state (`terraform.tfstate`) es un archivo JSON que mapea los recursos definidos en los archivos `.tf` con los recursos reales en el cloud.

```json
{
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0abc123def456",
            "public_ip": "54.123.45.67",
            "instance_type": "t3.micro"
          }
        }
      ]
    }
  ]
}
```

**Nunca edites el state manualmente**. Usa comandos:

```bash
# Listar recursos en el state
terraform state list

# Ver detalles de un recurso
terraform state show aws_instance.web

# Mover recurso (renombrar en config sin recrear)
terraform state mv aws_instance.web aws_instance.servidor_web

# Eliminar recurso del state (sin destruir el recurso real)
terraform state rm aws_instance.web

# Descargar state remoto a local
terraform state pull > state.json
```

### 6.2 State local vs remoto

| Aspecto | Local | Remoto |
|---------|-------|--------|
| Donde vive | `terraform.tfstate` en disco | S3, GCS, Terraform Cloud |
| Colaboracion | Imposible (un solo usuario) | Multiples usuarios |
| Locking | No | Si (DynamoDB, GCS nativo) |
| Backup | Manual | Automatico (versionado) |
| Seguridad | En disco sin cifrar | Cifrado en reposo |
| Uso | Desarrollo local, pruebas | Produccion, equipos |

**Regla**: local solo para aprender. En cualquier proyecto real, usa state remoto.

### 6.3 Backend S3 con DynamoDB

Primero, crear el bucket y la tabla (una sola vez, puede ser manual o con otro Terraform):

```hcl
# bootstrap/main.tf - ejecutar una sola vez para crear el backend
resource "aws_s3_bucket" "terraform_state" {
  bucket = "mi-empresa-terraform-state"

  lifecycle {
    prevent_destroy = true             # nunca destruir el bucket de state
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"                 # mantener historial de states
  }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "aws:kms"        # cifrado con KMS
    }
  }
}

resource "aws_dynamodb_table" "terraform_locks" {
  name         = "terraform-state-locks"
  billing_mode = "PAY_PER_REQUEST"     # sin costo si no se usa
  hash_key     = "LockID"

  attribute {
    name = "LockID"
    type = "S"
  }
}
```

Despues, configurar el backend en tu proyecto:

```hcl
# main.tf
terraform {
  backend "s3" {
    bucket         = "mi-empresa-terraform-state"
    key            = "proyecto/web/terraform.tfstate"   # ruta dentro del bucket
    region         = "us-east-1"
    dynamodb_table = "terraform-state-locks"            # locking
    encrypt        = true
  }
}
```

### 6.4 Backend GCS

```hcl
terraform {
  backend "gcs" {
    bucket = "mi-empresa-terraform-state"
    prefix = "proyecto/web"            # carpeta dentro del bucket
  }
}
```

GCS incluye locking nativo sin necesidad de tabla adicional.

### 6.5 State locking

El locking previene que dos personas ejecuten `terraform apply` simultaneamente sobre el mismo state:

```
Persona A: terraform apply
  -> Adquiere lock en DynamoDB
  -> Ejecuta cambios
  -> Libera lock

Persona B: terraform apply (al mismo tiempo)
  -> Intenta adquirir lock
  -> ERROR: "Error locking state: Lock already held"
  -> Espera a que A termine
```

Si un lock queda "colgado" (por ejemplo, si el proceso se mato):

```bash
# Forzar desbloqueo (con cuidado, asegurar que nadie esta ejecutando)
terraform force-unlock LOCK_ID
```

---

## 7. Variables

### 7.1 Input variables

```hcl
# variables.tf

# Variable simple con tipo y default
variable "environment" {
  description = "Entorno de despliegue"
  type        = string
  default     = "development"

  validation {                         # validacion custom
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "Environment debe ser development, staging o production."
  }
}

# Variable sin default (obligatoria)
variable "db_password" {
  description = "Password de la base de datos"
  type        = string
  sensitive   = true                   # no se muestra en logs ni plan
}

# Variable con tipo complejo
variable "vpc_config" {
  description = "Configuracion de la VPC"
  type = object({
    cidr_block     = string
    num_subnets    = number
    enable_nat     = bool
    tags           = map(string)
  })
  default = {
    cidr_block     = "10.0.0.0/16"
    num_subnets    = 3
    enable_nat     = true
    tags           = {}
  }
}

# Variable tipo lista
variable "allowed_cidrs" {
  description = "CIDRs permitidos en el security group"
  type        = list(string)
  default     = ["0.0.0.0/0"]
}
```

Formas de pasar valores (en orden de precedencia, de menor a mayor):
1. Valor `default` en la variable
2. Archivo `terraform.tfvars`
3. Archivos `*.auto.tfvars`
4. Flag `-var-file="archivo.tfvars"`
5. Flag `-var="nombre=valor"`
6. Variable de entorno `TF_VAR_nombre`

### 7.2 Output values

```hcl
# outputs.tf

output "server_ip" {
  description = "IP publica del servidor web"
  value       = aws_instance.web.public_ip
}

output "db_endpoint" {
  description = "Endpoint de la base de datos"
  value       = aws_db_instance.main.endpoint
  sensitive   = true                   # no mostrar en consola
}

output "all_server_ips" {
  description = "IPs de todos los workers"
  value       = aws_instance.worker[*].public_ip
}
```

Los outputs se muestran al final de `terraform apply` y se pueden consultar con:

```bash
terraform output                       # todos los outputs
terraform output server_ip             # un output especifico
terraform output -json                 # en formato JSON (para scripts)
```

### 7.3 Local values

Valores calculados que se reutilizan dentro del modulo:

```hcl
locals {
  # Prefijo comun para nombres
  name_prefix = "${var.project}-${var.environment}"

  # Tags comunes
  common_tags = {
    Project     = var.project
    Environment = var.environment
    ManagedBy   = "terraform"
  }

  # Calculo: numero de subnets segun entorno
  subnet_count = var.environment == "production" ? 3 : 2
}

# Uso:
resource "aws_instance" "web" {
  tags = merge(local.common_tags, {
    Name = "${local.name_prefix}-web"
  })
}
```

### 7.4 terraform.tfvars

Archivos de variables para cada entorno:

```hcl
# terraform.tfvars (valores por defecto, commitear a Git)
project     = "mi-app"
environment = "development"

# production.tfvars (valores para produccion)
project       = "mi-app"
environment   = "production"
instance_type = "t3.large"
db_instance_class = "db.r6g.xlarge"
```

Uso:

```bash
# Desarrollo (usa terraform.tfvars automaticamente)
terraform apply

# Produccion (especificar archivo)
terraform apply -var-file="production.tfvars"
```

**Gotcha**: `terraform.tfvars` y `*.auto.tfvars` se cargan automaticamente. Otros archivos `.tfvars` requieren `-var-file`. No commitees archivos con secretos; usa variables de entorno o un gestor de secretos.

---

## 8. Modulos

### 8.1 Para que sirven

Un modulo es un contenedor de recursos reutilizable. En lugar de copiar y pegar la configuracion de una VPC en cada proyecto, la encapsulas en un modulo.

```
Sin modulos:                         Con modulos:
proyecto-a/                          modules/vpc/
  vpc.tf (50 lineas)                   main.tf
  subnets.tf (80 lineas)               variables.tf
  nat.tf (30 lineas)                   outputs.tf
proyecto-b/
  vpc.tf (mismas 50 lineas)         proyecto-a/
  subnets.tf (mismas 80 lineas)       main.tf -> module "vpc" { source = "../modules/vpc" }
  nat.tf (mismas 30 lineas)         proyecto-b/
                                       main.tf -> module "vpc" { source = "../modules/vpc" }
```

### 8.2 Estructura de un modulo

```
modules/vpc/
  main.tf           # recursos principales
  variables.tf      # inputs del modulo
  outputs.tf        # outputs del modulo
  versions.tf       # required_providers
  README.md         # documentacion
```

### 8.3 Crear un modulo

```hcl
# modules/vpc/variables.tf
variable "name" {
  description = "Nombre de la VPC"
  type        = string
}

variable "cidr_block" {
  description = "CIDR block de la VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "num_subnets" {
  description = "Numero de subnets publicas"
  type        = number
  default     = 2
}

variable "tags" {
  description = "Tags adicionales"
  type        = map(string)
  default     = {}
}
```

```hcl
# modules/vpc/main.tf
data "aws_availability_zones" "available" {
  state = "available"
}

resource "aws_vpc" "main" {
  cidr_block           = var.cidr_block
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(var.tags, {
    Name = var.name
  })
}

resource "aws_subnet" "public" {
  count                   = var.num_subnets
  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.cidr_block, 8, count.index)
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = merge(var.tags, {
    Name = "${var.name}-public-${count.index}"
  })
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(var.tags, {
    Name = "${var.name}-igw"
  })
}
```

```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID de la VPC"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "IDs de las subnets publicas"
  value       = aws_subnet.public[*].id
}

output "vpc_cidr" {
  description = "CIDR block de la VPC"
  value       = aws_vpc.main.cidr_block
}
```

### 8.4 Usar un modulo

```hcl
# main.tf del proyecto
module "vpc" {
  source = "./modules/vpc"             # ruta local

  name        = "mi-app-vpc"
  cidr_block  = "10.0.0.0/16"
  num_subnets = 3
  tags        = local.common_tags
}

# Usar outputs del modulo
resource "aws_instance" "web" {
  subnet_id = module.vpc.subnet_ids[0]
  # ...
}
```

### 8.5 Modulos del registry

Terraform Registry tiene modulos mantenidos por la comunidad:

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"   # registry publico
  version = "~> 5.0"

  name = "mi-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["us-east-1a", "us-east-1b"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  single_nat_gateway = true
}
```

Tambien se pueden usar modulos de repositorios Git privados:

```hcl
module "vpc" {
  source = "git::https://github.com/mi-org/terraform-modules.git//vpc?ref=v1.2.0"
}
```

---

## 9. Workspaces

Los workspaces permiten gestionar multiples estados desde la misma configuracion. Cada workspace tiene su propio state.

```bash
# Listar workspaces
terraform workspace list

# Crear workspace
terraform workspace new staging
terraform workspace new production

# Cambiar de workspace
terraform workspace select staging

# Ver workspace actual
terraform workspace show
```

Usar el workspace en la configuracion:

```hcl
locals {
  environment = terraform.workspace    # "default", "staging", "production"
}

resource "aws_instance" "web" {
  instance_type = terraform.workspace == "production" ? "t3.large" : "t3.micro"

  tags = {
    Environment = terraform.workspace
  }
}
```

**Gotcha**: muchos equipos prefieren directorios separados (`envs/staging/`, `envs/production/`) en lugar de workspaces porque es mas explicito y permite diferentes backends por entorno. Workspaces es util para diferencias pequenas; directorios separados para infraestructura muy diferente entre entornos.

---

## 10. Import de recursos existentes

Si ya tienes infraestructura creada manualmente, puedes importarla al state de Terraform:

Metodo clasico (Terraform < 1.5):

```bash
# 1. Escribir el resource block en .tf
# 2. Importar al state
terraform import aws_instance.web i-0abc123def456

# 3. Ejecutar plan para verificar que no hay diferencias
terraform plan
# 4. Ajustar .tf hasta que plan diga "No changes"
```

Metodo moderno (Terraform >= 1.5) con `import` block:

```hcl
# import.tf
import {
  to = aws_instance.web
  id = "i-0abc123def456"
}

# Generar configuracion automaticamente
# terraform plan -generate-config-out=generated.tf
```

```bash
# Genera generated.tf con la configuracion del recurso importado
terraform plan -generate-config-out=generated.tf

# Revisar, ajustar y aplicar
terraform apply
```

---

## 11. Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| **IaC** | Infraestructura definida en archivos de texto | Archivos `.tf` en Git |
| **HCL** | Lenguaje declarativo de Terraform | `resource`, `variable`, `output` blocks |
| **Provider** | Plugin para interactuar con un cloud | `provider "aws" { region = "..." }` |
| **Resource** | Componente de infra que Terraform crea | `resource "aws_instance" "web" { ... }` |
| **Data Source** | Lectura de datos existentes | `data "aws_ami" "latest" { ... }` |
| **State** | Mapa entre config y realidad | `terraform.tfstate` (local o remoto) |
| **Backend remoto** | State compartido con locking | S3+DynamoDB o GCS |
| **Variables** | Parametros configurables | `variable`, `terraform.tfvars`, `-var` |
| **Modulos** | Recursos reutilizables empaquetados | `module "vpc" { source = "./modules/vpc" }` |
| **Workspaces** | Multiples states para misma config | `terraform workspace new staging` |
| **Import** | Traer infra existente al state | `terraform import` o bloque `import {}` |
| **Plan** | Preview de cambios sin aplicar | `terraform plan -out=plan.tfplan` |

---

## 12. Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | [Primeros recursos](ejercicio-01-primeros-recursos/) | Crear una VPC con subnets, security group e instancia EC2 usando el provider de AWS (o LocalStack para pruebas locales) | resource, provider, data source, outputs |
| 02 | [Modulos](ejercicio-02-modulos/) | Refactorizar los recursos del ejercicio anterior en modulos reutilizables (vpc, security-group, compute) con inputs y outputs | module, variables, outputs, estructura de modulo |
| 03 | [State remoto](ejercicio-03-state-remoto/) | Configurar backend S3 con DynamoDB para state locking, migrar el state local al remoto | backend s3, dynamodb, terraform init -migrate-state |
| 04 | [Infra completa](ejercicio-04-infra-completa/) | Desplegar una infraestructura multi-entorno (staging/prod) con modulos, variables por entorno y outputs para conectar con CI/CD | workspaces o directorios, tfvars, modulos, import |

Haz los ejercicios en orden. Cada uno construye sobre los conceptos del anterior.

---

## 13. Recursos

- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [Terraform Language Reference](https://developer.hashicorp.com/terraform/language)
- [Terraform Registry (providers y modulos)](https://registry.terraform.io/)
- [AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [Google Provider Documentation](https://registry.terraform.io/providers/hashicorp/google/latest/docs)
- [Terraform Best Practices](https://www.terraform-best-practices.com/)
- [Terraform Up & Running (libro)](https://www.terraformupandrunning.com/)
- [LocalStack (AWS local para testing)](https://localstack.cloud/)
