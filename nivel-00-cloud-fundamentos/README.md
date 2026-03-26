# Nivel 00 — Fundamentos de Cloud Computing

> Todo backend moderno acaba desplegado en la nube. Este nivel cubre los conceptos
> fundamentales de cloud computing: modelos de servicio, los dos grandes proveedores
> (AWS y GCP), networking, identidad y acceso, y la configuracion inicial de tu entorno
> de trabajo con CLI. Sin estos fundamentos, Kubernetes y CI/CD no tienen sentido.

---

## Contenido

1. [Modelos de servicio: IaaS vs PaaS vs SaaS](#1-modelos-de-servicio-iaas-vs-paas-vs-saas)
2. [AWS vs GCP: comparativa practica](#2-aws-vs-gcp-comparativa-practica)
3. [Regiones y zonas de disponibilidad](#3-regiones-y-zonas-de-disponibilidad)
4. [VPCs y subnets: networking en la nube](#4-vpcs-y-subnets-networking-en-la-nube)
5. [IAM: usuarios, roles y politicas](#5-iam-usuarios-roles-y-politicas)
6. [CLI: configuracion de aws y gcloud](#6-cli-configuracion-de-aws-y-gcloud)
7. [Free tier y alertas de facturacion](#7-free-tier-y-alertas-de-facturacion)
8. [Buenas practicas de seguridad inicial](#8-buenas-practicas-de-seguridad-inicial)
9. [Resumen](#resumen)
10. [Ejercicios](#ejercicios)
11. [Recursos](#recursos)

---

## 1. Modelos de servicio: IaaS vs PaaS vs SaaS

Cloud computing no es un solo producto. Es un espectro de niveles de abstraccion.
Cuanto mas arriba en la pila, menos gestionas tu y mas gestiona el proveedor.

### 1.1 La analogia de la pizza

Imagina que quieres cenar pizza:

- **On-premises** (tu datacenter): compras los ingredientes, haces la masa, horneas,
  pones la mesa, limpias despues. Controlas todo, pero haces todo.
- **IaaS** (Infrastructure as a Service): te dan la cocina equipada. Tu pones los
  ingredientes y cocinas.
- **PaaS** (Platform as a Service): te dan la pizza hecha, tu solo eliges los toppings
  y la sirves.
- **SaaS** (Software as a Service): te traen la pizza a casa. Solo comes.

### 1.2 IaaS: Infrastructure as a Service

IaaS te proporciona los bloques fundamentales de infraestructura: maquinas virtuales,
almacenamiento, redes. Tu gestionas el sistema operativo, runtime y aplicacion.

```
Lo que el proveedor gestiona:          Lo que tu gestionas:
+----------------------------+         +----------------------------+
| Hardware fisico            |         | Sistema operativo          |
| Virtualizacion             |         | Runtime (JVM, Node, etc.)  |
| Red fisica                 |         | Aplicacion                 |
| Almacenamiento fisico      |         | Datos                      |
+----------------------------+         | Seguridad de la app        |
                                       | Parches del SO             |
                                       +----------------------------+
```

Ejemplos de IaaS:

| Proveedor | Servicio | Descripcion |
|-----------|----------|-------------|
| AWS | EC2 | Maquinas virtuales bajo demanda |
| GCP | Compute Engine | Maquinas virtuales con facturacion por segundo |
| AWS | EBS | Discos persistentes para EC2 |
| GCP | Persistent Disk | Discos persistentes para Compute Engine |

```bash
# Lanzar una instancia EC2 con AWS CLI
aws ec2 run-instances \
    --image-id ami-0c55b159cbfafe1f0 \       # Amazon Linux 2
    --instance-type t3.micro \                # 2 vCPU, 1 GB RAM
    --key-name mi-keypair \                   # para acceso SSH
    --security-group-ids sg-0123456789abcdef \
    --subnet-id subnet-0123456789abcdef \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=mi-servidor}]'

# Lanzar una instancia en GCP
gcloud compute instances create mi-servidor \
    --zone=us-central1-a \                    # zona especifica
    --machine-type=e2-micro \                 # 2 vCPU, 1 GB RAM (free tier)
    --image-family=debian-12 \                # sistema operativo
    --image-project=debian-cloud \
    --tags=http-server                        # etiqueta para firewall
```

### 1.3 PaaS: Platform as a Service

PaaS abstrae la infraestructura. Tu solo despliegas codigo y el proveedor se encarga
del SO, runtime, escalado y mantenimiento.

```
Lo que el proveedor gestiona:          Lo que tu gestionas:
+----------------------------+         +----------------------------+
| Hardware fisico            |         | Codigo de la aplicacion    |
| Virtualizacion             |         | Datos                      |
| Sistema operativo          |         | Configuracion              |
| Runtime                    |         +----------------------------+
| Middleware                 |
| Escalado                   |
+----------------------------+
```

Ejemplos de PaaS:

| Proveedor | Servicio | Descripcion |
|-----------|----------|-------------|
| AWS | Elastic Beanstalk | Despliegue de apps web con auto-scaling |
| AWS | App Runner | Contenedores sin gestionar infraestructura |
| GCP | App Engine | Apps web con escalado automatico |
| GCP | Cloud Run | Contenedores serverless |

```bash
# Desplegar una app Spring Boot en Cloud Run
gcloud run deploy mi-api \
    --image=gcr.io/mi-proyecto/mi-api:v1 \   # imagen Docker
    --platform=managed \                       # GCP gestiona la infra
    --region=us-central1 \
    --allow-unauthenticated \                  # acceso publico
    --port=8080 \                              # puerto de la app
    --memory=512Mi \                           # limite de memoria
    --cpu=1                                    # limite de CPU
```

### 1.4 SaaS: Software as a Service

SaaS es software completo que usas sin gestionar nada de infraestructura ni codigo.
Solo consumes el servicio.

| Ejemplo | Que es | Alternativa self-hosted |
|---------|--------|------------------------|
| Gmail | Email | Configurar Postfix en EC2 |
| GitHub | Control de versiones | Instalar GitLab en tu servidor |
| Datadog | Monitorizacion | Instalar Prometheus + Grafana |
| Auth0 | Autenticacion | Implementar tu propio OAuth2 |

### 1.5 Cuando usar cada modelo

| Criterio | IaaS | PaaS | SaaS |
|----------|------|------|------|
| Control | Maximo | Medio | Minimo |
| Complejidad operativa | Alta | Media | Baja |
| Coste inicial | Bajo (pago por uso) | Bajo | Variable (suscripcion) |
| Escalado | Manual o con autogroups | Automatico | Transparente |
| Caso de uso | Apps con requisitos especificos de SO/red | Apps web estandar | Herramientas de equipo |

**Regla practica**: empieza con PaaS si puedes. Baja a IaaS solo cuando PaaS te limite.

---

## 2. AWS vs GCP: comparativa practica

No hay un proveedor "mejor". Ambos tienen servicios equivalentes. La eleccion depende
de tu equipo, presupuesto y ecosistema existente.

### 2.1 Equivalencias de servicios

| Categoria | AWS | GCP | Nota |
|-----------|-----|-----|------|
| Computo (VMs) | EC2 | Compute Engine | GCP factura por segundo, AWS por hora (minimo) |
| Contenedores gestionados | ECS / EKS | GKE | GKE tiene mas funcionalidades out-of-the-box |
| Serverless (funciones) | Lambda | Cloud Functions | Ambos soportan muchos runtimes |
| Serverless (contenedores) | App Runner / Fargate | Cloud Run | Cloud Run es mas maduro |
| Base de datos relacional | RDS | Cloud SQL | Ambos soportan PostgreSQL, MySQL |
| Base de datos NoSQL | DynamoDB | Firestore / Bigtable | DynamoDB es mas popular |
| Almacenamiento objetos | S3 | Cloud Storage | S3 es el estandar de facto |
| CDN | CloudFront | Cloud CDN | Rendimiento similar |
| DNS | Route 53 | Cloud DNS | Route 53 tiene mas funcionalidades |
| Mensajeria | SQS / SNS | Pub/Sub | Pub/Sub es mas simple y flexible |
| IAM | IAM | IAM + Workload Identity | Modelos diferentes (ver seccion 5) |
| CLI | aws | gcloud | Ambos potentes, gcloud mas interactivo |
| IaC nativo | CloudFormation | Deployment Manager | Terraform es mejor que ambos |

### 2.2 Diferencias clave

```
AWS:
  - Mayor cuota de mercado (~32%)
  - Mas servicios disponibles (200+)
  - Documentacion mas extensa
  - Certificaciones mas valoradas en el mercado
  - Ecosistema mas maduro

GCP:
  - Mejor networking (red privada global de Google)
  - Kubernetes nativo (Google creo K8s, GKE es el mejor K8s gestionado)
  - Mejor para ML/AI (Vertex AI, TPUs)
  - Pricing mas simple y generoso (free tier de Compute Engine)
  - BigQuery para analytics (sin equivalente real en AWS)
```

### 2.3 Estructura de cuentas

Ambos proveedores organizan los recursos en jerarquias:

```
AWS:                                GCP:
Organization                        Organization
  +-- Account (prod)                  +-- Folder (prod)
  |     +-- VPC                       |     +-- Project (api-prod)
  |     +-- EC2, RDS, etc.           |     +-- Project (web-prod)
  +-- Account (dev)                   +-- Folder (dev)
        +-- VPC                             +-- Project (api-dev)
        +-- EC2, RDS, etc.

En AWS la unidad de aislamiento       En GCP la unidad basica es el
es la Account.                         Project. Todo recurso pertenece
                                       a un Project.
```

---

## 3. Regiones y zonas de disponibilidad

### 3.1 Conceptos fundamentales

Los proveedores cloud distribuyen su infraestructura globalmente en regiones y zonas:

```
Region (us-east-1 / us-central1):
  Una ubicacion geografica con multiples datacenters.
  Ejemplo: us-east-1 (Virginia), eu-west-1 (Irlanda)

Zona de disponibilidad (us-east-1a / us-central1-a):
  Un datacenter independiente dentro de una region.
  Tiene su propia alimentacion electrica, red y refrigeracion.
  Si una zona cae, las otras siguen funcionando.

+--------------------------------------------------+
|  Region: us-east-1 (Virginia)                     |
|                                                    |
|  +------------+  +------------+  +------------+   |
|  | AZ: 1a     |  | AZ: 1b     |  | AZ: 1c     |  |
|  | Datacenter |  | Datacenter |  | Datacenter |   |
|  +------------+  +------------+  +------------+   |
|       |               |               |            |
|       +----------- Red de baja latencia -----------+
+--------------------------------------------------+
```

### 3.2 Nomenclatura

| Proveedor | Region | Zona | Ejemplo |
|-----------|--------|------|---------|
| AWS | `us-east-1` | `us-east-1a` | Virginia, zona A |
| AWS | `eu-west-1` | `eu-west-1b` | Irlanda, zona B |
| GCP | `us-central1` | `us-central1-a` | Iowa, zona A |
| GCP | `europe-west1` | `europe-west1-b` | Belgica, zona B |

### 3.3 Como elegir una region

La eleccion de region afecta a latencia, coste, cumplimiento legal y disponibilidad
de servicios:

| Factor | Criterio |
|--------|----------|
| Latencia | Elige la region mas cercana a tus usuarios |
| Coste | us-east-1 (AWS) suele ser la region mas barata |
| Compliance | GDPR puede requerir datos en la UE |
| Servicios | No todos los servicios estan en todas las regiones |
| Disaster recovery | Usa multiples regiones para alta disponibilidad |

```bash
# Listar regiones disponibles en AWS
aws ec2 describe-regions --output table

# Listar regiones en GCP
gcloud compute regions list

# Listar zonas en una region especifica (GCP)
gcloud compute zones list --filter="region:us-central1"
```

### 3.4 Multi-AZ: alta disponibilidad

Para produccion, siempre despliega en multiples zonas de disponibilidad:

```
+--------------------------------------------------+
|  Region: us-east-1                                |
|                                                    |
|  AZ-1a              AZ-1b              AZ-1c      |
|  +----------+       +----------+       +----------+|
|  | App (x2) |       | App (x2) |       | App (x2) ||
|  | DB Master |      | DB Replica|      | DB Replica||
|  +----------+       +----------+       +----------+|
|       |                  |                  |       |
|       +--- Load Balancer (distribuye trafico) ----+ |
+--------------------------------------------------+
```

Si la zona AZ-1a cae completamente (fallo electrico, incendio), las zonas AZ-1b y
AZ-1c siguen sirviendo trafico. Los usuarios ni se enteran.

---

## 4. VPCs y subnets: networking en la nube

### 4.1 VPC: tu red privada en la nube

Una VPC (Virtual Private Cloud) es una red virtual aislada dentro de la nube. Es como
tener tu propia red de datacenter, pero virtualizada. Todo recurso cloud (VMs, bases
de datos, contenedores) vive dentro de una VPC.

```
+----------------------------------------------------------+
|  VPC: 10.0.0.0/16 (65,536 direcciones IP)                |
|                                                            |
|  Subnet publica: 10.0.1.0/24       Subnet privada: 10.0.2.0/24
|  +------------------------+         +------------------------+
|  | Internet Gateway       |         | NAT Gateway            |
|  | Load Balancer          |         | App Servers             |
|  | Bastion Host           |         | Base de datos           |
|  +------------------------+         +------------------------+
|         |                                   |               |
|         +--- Route Table publica ---+       |               |
|         |   0.0.0.0/0 -> IGW       |       |               |
|         +---------------------------+       |               |
|                                             |               |
|         +--- Route Table privada ----------+|               |
|         |   0.0.0.0/0 -> NAT GW           ||               |
|         +----------------------------------+|               |
+----------------------------------------------------------+
```

### 4.2 Subnets: segmentos de red

Las subnets dividen la VPC en segmentos mas pequenos. Cada subnet existe en una sola
zona de disponibilidad.

| Tipo | Acceso a Internet | Uso tipico |
|------|-------------------|------------|
| Publica | Si (via Internet Gateway) | Load balancers, bastion hosts |
| Privada | Solo saliente (via NAT Gateway) | App servers, bases de datos |
| Aislada | Sin acceso a Internet | Datos sensibles, compliance |

```bash
# Crear una VPC en AWS
aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \               # rango de IPs de la VPC
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=mi-vpc}]'

# Crear subnet publica
aws ec2 create-subnet \
    --vpc-id vpc-0123456789abcdef \
    --cidr-block 10.0.1.0/24 \               # 256 IPs
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=publica-1a}]'

# Crear subnet privada
aws ec2 create-subnet \
    --vpc-id vpc-0123456789abcdef \
    --cidr-block 10.0.2.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=privada-1a}]'
```

```bash
# Crear una VPC en GCP
gcloud compute networks create mi-vpc \
    --subnet-mode=custom                       # modo manual (recomendado)

# Crear subnet en GCP
gcloud compute networks subnets create publica-us \
    --network=mi-vpc \
    --region=us-central1 \
    --range=10.0.1.0/24                        # rango CIDR
```

### 4.3 Security Groups y Firewall Rules

Los Security Groups (AWS) y Firewall Rules (GCP) controlan el trafico de red:

```bash
# AWS: crear security group que permita HTTP y SSH
aws ec2 create-security-group \
    --group-name web-sg \
    --description "Security group para servidores web" \
    --vpc-id vpc-0123456789abcdef

# Permitir HTTP desde cualquier IP
aws ec2 authorize-security-group-ingress \
    --group-id sg-0123456789abcdef \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0                           # cualquier origen

# Permitir SSH solo desde tu IP
aws ec2 authorize-security-group-ingress \
    --group-id sg-0123456789abcdef \
    --protocol tcp \
    --port 22 \
    --cidr 203.0.113.50/32                     # solo tu IP publica
```

```bash
# GCP: crear firewall rule para HTTP
gcloud compute firewall-rules create allow-http \
    --network=mi-vpc \
    --allow=tcp:80 \
    --source-ranges=0.0.0.0/0 \                # cualquier origen
    --target-tags=http-server                   # solo aplica a VMs con esta etiqueta
```

### 4.4 CIDR: notacion de rangos de IP

CIDR (Classless Inter-Domain Routing) define rangos de IP con un prefijo:

| CIDR | IPs disponibles | Uso tipico |
|------|----------------|------------|
| `10.0.0.0/16` | 65,536 | VPC completa |
| `10.0.1.0/24` | 256 | Subnet individual |
| `10.0.1.0/28` | 16 | Subnet muy pequena |
| `203.0.113.50/32` | 1 | Una IP especifica |
| `0.0.0.0/0` | Todas | "Cualquier IP" (Internet) |

El numero despues de `/` indica cuantos bits del prefijo son fijos. Cuanto mayor es
el numero, menos IPs tiene el rango.

---

## 5. IAM: usuarios, roles y politicas

### 5.1 Principio de minimo privilegio

IAM (Identity and Access Management) controla quien puede hacer que en tu cuenta cloud.
La regla de oro: **cada identidad debe tener solo los permisos que necesita, nada mas**.

### 5.2 IAM en AWS

AWS IAM tiene cuatro conceptos fundamentales:

```
+------------------------------------------------------------------+
|  AWS IAM                                                          |
|                                                                    |
|  Users: identidades individuales (personas o servicios)           |
|  Groups: colecciones de usuarios con permisos compartidos          |
|  Roles: identidades asumibles (para servicios o cross-account)    |
|  Policies: documentos JSON que definen permisos                    |
+------------------------------------------------------------------+
```

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject"
            ],
            "Resource": "arn:aws:s3:::mi-bucket/*"
        },
        {
            "Effect": "Deny",
            "Action": "s3:DeleteObject",
            "Resource": "arn:aws:s3:::mi-bucket/*"
        }
    ]
}
```

La estructura de una policy es:
- **Effect**: `Allow` o `Deny` (Deny siempre gana)
- **Action**: que operaciones permite/deniega
- **Resource**: sobre que recursos aplica (ARN)
- **Condition** (opcional): condiciones adicionales (IP, hora, MFA, etc.)

```bash
# Crear un usuario IAM
aws iam create-user --user-name developer-juan

# Crear un grupo y asignar una policy
aws iam create-group --group-name developers
aws iam attach-group-policy \
    --group-name developers \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess

# Agregar usuario al grupo
aws iam add-user-to-group \
    --user-name developer-juan \
    --group-name developers

# Crear un rol para EC2 (permite que una instancia acceda a S3)
aws iam create-role \
    --role-name ec2-s3-access \
    --assume-role-policy-document file://trust-policy.json
```

### 5.3 IAM en GCP

GCP maneja IAM de forma diferente. Los permisos se asignan a nivel de proyecto,
carpeta u organizacion, no a nivel de usuario global.

```
GCP IAM:
  Principal (quien): usuario, service account, grupo de Google
  Role (que puede hacer): conjunto de permisos
  Resource (donde): proyecto, carpeta, organizacion, recurso especifico

  Binding: Principal + Role + Resource
```

Tipos de roles en GCP:

| Tipo | Ejemplo | Cuando usar |
|------|---------|-------------|
| Basico | `roles/viewer`, `roles/editor`, `roles/owner` | Nunca en produccion (demasiado amplio) |
| Predefinido | `roles/storage.objectViewer` | La opcion recomendada |
| Custom | `projects/mi-proj/roles/customRole` | Cuando necesitas granularidad extrema |

```bash
# Asignar un rol a un usuario en un proyecto
gcloud projects add-iam-policy-binding mi-proyecto \
    --member="user:juan@empresa.com" \
    --role="roles/compute.viewer"                    # solo lectura de VMs

# Crear una service account (identidad para servicios)
gcloud iam service-accounts create mi-api-sa \
    --display-name="Service account para mi API"

# Asignar rol a la service account
gcloud projects add-iam-policy-binding mi-proyecto \
    --member="serviceAccount:mi-api-sa@mi-proyecto.iam.gserviceaccount.com" \
    --role="roles/cloudsql.client"                   # acceso a Cloud SQL
```

### 5.4 Service Accounts vs Users

| Concepto | User | Service Account |
|----------|------|-----------------|
| Representa | Una persona | Una aplicacion o servicio |
| Autenticacion | Contrasena + MFA | Claves JSON o Workload Identity |
| Rotacion | La persona cambia su contrasena | Automatica con Workload Identity |
| Uso | Acceso a consola web y CLI | Codigo que corre en servidores |

**Regla**: nunca uses credenciales de usuario en aplicaciones. Siempre service accounts.

---

## 6. CLI: configuracion de aws y gcloud

### 6.1 Instalar AWS CLI

```bash
# Linux (x86_64)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# macOS
brew install awscli

# Verificar instalacion
aws --version
# aws-cli/2.15.0 Python/3.11.6 Linux/5.15.0-1053-aws
```

### 6.2 Configurar AWS CLI

```bash
# Configuracion interactiva
aws configure
# AWS Access Key ID [None]: AKIAIOSFODNN7EXAMPLE
# AWS Secret Access Key [None]: wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY
# Default region name [None]: us-east-1
# Default output format [None]: json

# Configurar un perfil con nombre (recomendado para multiples cuentas)
aws configure --profile dev
aws configure --profile prod

# Usar un perfil especifico
aws s3 ls --profile dev
export AWS_PROFILE=dev                          # para toda la sesion
```

Los archivos de configuracion se guardan en:
```
~/.aws/
  credentials    # access key y secret key
  config         # region, output format, perfil por defecto
```

### 6.3 Instalar Google Cloud CLI

```bash
# Linux
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-linux-x86_64.tar.gz
tar -xf google-cloud-cli-linux-x86_64.tar.gz
./google-cloud-sdk/install.sh

# macOS
brew install google-cloud-sdk

# Verificar instalacion
gcloud version
```

### 6.4 Configurar gcloud

```bash
# Login interactivo (abre el navegador)
gcloud auth login

# Configurar proyecto por defecto
gcloud config set project mi-proyecto-id

# Configurar region y zona por defecto
gcloud config set compute/region us-central1
gcloud config set compute/zone us-central1-a

# Ver configuracion actual
gcloud config list

# Configuraciones con nombre (equivalente a perfiles de AWS)
gcloud config configurations create dev
gcloud config configurations activate dev
gcloud config set project mi-proyecto-dev
gcloud config set compute/region us-central1

# Cambiar entre configuraciones
gcloud config configurations activate prod
```

### 6.5 Comandos esenciales

```bash
# --- AWS ---
aws sts get-caller-identity              # quien soy (verificar credenciales)
aws s3 ls                                 # listar buckets S3
aws ec2 describe-instances                # listar instancias EC2
aws ec2 describe-vpcs                     # listar VPCs
aws iam list-users                        # listar usuarios IAM

# --- GCP ---
gcloud auth list                          # cuentas autenticadas
gcloud config list                        # configuracion activa
gcloud projects list                      # proyectos accesibles
gcloud compute instances list             # listar VMs
gcloud compute networks list              # listar VPCs
```

---

## 7. Free tier y alertas de facturacion

### 7.1 Free tier de AWS

AWS ofrece tres tipos de free tier:

| Tipo | Duracion | Ejemplo |
|------|----------|---------|
| 12 meses gratis | Desde la creacion de cuenta | EC2 t2.micro (750h/mes), RDS db.t3.micro |
| Siempre gratis | Permanente | Lambda (1M invocaciones/mes), DynamoDB (25 GB) |
| Prueba | Una sola vez | SageMaker (250h primer mes) |

Servicios mas utiles del free tier:

```
EC2 t2.micro:     750 horas/mes (1 instancia 24/7)
RDS db.t3.micro:  750 horas/mes
S3:               5 GB almacenamiento
Lambda:           1 millon invocaciones/mes
CloudWatch:       10 metricas custom, 10 alarmas
DynamoDB:         25 GB, 25 WCU, 25 RCU
```

### 7.2 Free tier de GCP

GCP ofrece $300 de credito por 90 dias para nuevas cuentas, mas servicios always-free:

```
Compute Engine e2-micro:  1 instancia/mes (us-west1, us-central1, us-east1)
Cloud Storage:             5 GB (Standard en us-central1)
Cloud Functions:           2 millones invocaciones/mes
Firestore:                 1 GB almacenamiento
BigQuery:                  1 TB queries/mes
Cloud Run:                 2 millones requests/mes
Pub/Sub:                   10 GB/mes
```

### 7.3 Configurar alertas de facturacion

Esto es **critico**. Sin alertas, una mala configuracion puede generar facturas enormes.

```bash
# AWS: crear una alerta de presupuesto via CLI
aws budgets create-budget \
    --account-id 123456789012 \
    --budget '{
        "BudgetName": "alerta-mensual",
        "BudgetLimit": {
            "Amount": "10",
            "Unit": "USD"
        },
        "TimeUnit": "MONTHLY",
        "BudgetType": "COST"
    }' \
    --notifications-with-subscribers '[{
        "Notification": {
            "NotificationType": "ACTUAL",
            "ComparisonOperator": "GREATER_THAN",
            "Threshold": 80,
            "ThresholdType": "PERCENTAGE"
        },
        "Subscribers": [{
            "SubscriptionType": "EMAIL",
            "Address": "tu-email@ejemplo.com"
        }]
    }]'
```

```bash
# GCP: configurar alerta de presupuesto
# 1. Ir a Billing -> Budgets & alerts en la consola
# 2. O usar gcloud:
gcloud billing budgets create \
    --billing-account=XXXXXX-XXXXXX-XXXXXX \
    --display-name="alerta-mensual" \
    --budget-amount=10USD \
    --threshold-rule=percent=0.5 \              # alerta al 50%
    --threshold-rule=percent=0.8 \              # alerta al 80%
    --threshold-rule=percent=1.0                # alerta al 100%
```

### 7.4 Errores comunes de facturacion

| Error | Consecuencia | Prevencion |
|-------|-------------|------------|
| Dejar EC2/VMs corriendo | $5-50/mes por instancia olvidada | Etiquetar todo, revisar semanalmente |
| Elastic IP sin asociar (AWS) | $3.60/mes por IP | Liberar IPs no usadas |
| NAT Gateway olvidado | $32+/mes | Eliminar cuando no sea necesario |
| Snapshots acumulados | $0.05/GB/mes | Politica de retencion |
| Load Balancer sin trafico | $16+/mes | Revisar recursos periodicamente |

---

## 8. Buenas practicas de seguridad inicial

### 8.1 Checklist de seguridad para cuenta nueva

```
[ ] Habilitar MFA en la cuenta root/administrador
[ ] Crear un usuario IAM para uso diario (no usar root)
[ ] Crear alertas de facturacion
[ ] Habilitar CloudTrail (AWS) / Audit Logs (GCP)
[ ] No dejar access keys en codigo fuente (usar .gitignore)
[ ] Configurar Security Groups / Firewall restrictivos
[ ] No exponer bases de datos a Internet (subnet privada)
[ ] Rotar access keys periodicamente
[ ] Usar service accounts para aplicaciones
[ ] Revisar permisos trimestralmente
```

### 8.2 Variables de entorno para credenciales

Nunca pongas credenciales en el codigo:

```bash
# MAL: credenciales hardcodeadas (jamas hacer esto)
aws_access_key = "AKIAIOSFODNN7EXAMPLE"

# BIEN: variables de entorno
export AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
export AWS_SECRET_ACCESS_KEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# MEJOR: usar perfiles de AWS CLI
export AWS_PROFILE=dev

# MEJOR AUN: usar IAM roles (en EC2/ECS/EKS)
# La instancia asume el rol automaticamente, sin credenciales
```

### 8.3 Archivo .gitignore para cloud

```gitignore
# Credenciales de AWS
.aws/credentials
*.pem
*.key

# Credenciales de GCP
*-service-account.json
*-credentials.json
application_default_credentials.json

# Terraform state (contiene secrets)
*.tfstate
*.tfstate.backup
.terraform/

# Variables de entorno
.env
.env.local
.env.production
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| IaaS | Infraestructura virtualizada (VMs, redes, discos) | EC2, Compute Engine para control total |
| PaaS | Plataforma gestionada (tu solo despliegas codigo) | Cloud Run, App Engine para apps web |
| SaaS | Software como servicio (solo consumes) | Gmail, GitHub, Datadog |
| Region | Ubicacion geografica con multiples datacenters | Elegir la mas cercana a tus usuarios |
| Zona de disponibilidad | Datacenter independiente dentro de una region | Desplegar en multiples AZs para HA |
| VPC | Red virtual privada aislada | Una VPC por entorno (dev, prod) |
| Subnet | Segmento de red dentro de una VPC | Publica para LBs, privada para apps y DBs |
| Security Group | Firewall virtual a nivel de instancia | Abrir solo puertos necesarios |
| IAM User | Identidad para una persona | Para acceso a consola y CLI |
| IAM Role | Identidad asumible por servicios | Para apps en EC2, ECS, Lambda |
| IAM Policy | Documento JSON con permisos | Principio de minimo privilegio |
| Service Account | Identidad para aplicaciones (GCP) | Para codigo que corre en servidores |
| AWS CLI | Herramienta de linea de comandos de AWS | `aws configure` con perfiles |
| gcloud CLI | Herramienta de linea de comandos de GCP | `gcloud init` con configuraciones |
| Free Tier | Recursos gratuitos del proveedor | Usar para aprender, con alertas de billing |
| CIDR | Notacion para rangos de IP | `/16` para VPC, `/24` para subnets |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **CLI Setup** | Instalar y configurar AWS CLI y gcloud CLI, crear perfiles para dev/prod, verificar conectividad con comandos basicos | AWS CLI, gcloud, perfiles, `aws sts get-caller-identity` |
| 02 | **IAM basico** | Crear usuarios, grupos y politicas IAM en AWS. Crear service accounts en GCP. Aplicar principio de minimo privilegio | IAM users, groups, policies, service accounts, MFA |
| 03 | **VPC Networking** | Crear una VPC con subnets publicas y privadas, configurar Security Groups, desplegar una VM en cada subnet y verificar conectividad | VPC, subnets, CIDR, Security Groups, Internet Gateway, NAT |

---

## Recursos

- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [GCP Architecture Framework](https://cloud.google.com/architecture/framework)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [GCP Free Tier](https://cloud.google.com/free)
- [CIDR Calculator](https://www.ipaddressguide.com/cidr)
- [AWS CLI Command Reference](https://docs.aws.amazon.com/cli/latest/reference/)
- [gcloud CLI Reference](https://cloud.google.com/sdk/gcloud/reference)

---

> **Antes de continuar al siguiente nivel**: asegurate de tener AWS CLI y gcloud
> configurados y funcionando. Ejecuta `aws sts get-caller-identity` y
> `gcloud config list` para verificar. Sin esto, los ejercicios de los siguientes
> niveles no funcionaran.
