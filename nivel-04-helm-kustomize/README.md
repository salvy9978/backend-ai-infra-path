# Nivel 04 — Helm y Kustomize

> En los niveles anteriores escribiste manifiestos YAML a mano para cada recurso.
> Eso funciona con 5 archivos, pero con 50 microservicios en 3 entornos (dev, staging,
> prod), la gestion se vuelve insostenible. Helm y Kustomize resuelven este problema
> desde enfoques diferentes: Helm con templates parametrizados y Kustomize con
> transformaciones sobre YAML base. Este nivel cubre ambos, cuando usar cada uno,
> y como empaquetar tu app Spring Boot como un chart de Helm.

---

## Contenido

1. [Por que necesitas un package manager para K8s](#1-por-que-necesitas-un-package-manager-para-k8s)
2. [Helm: conceptos fundamentales](#2-helm-conceptos-fundamentales)
3. [Estructura de un chart](#3-estructura-de-un-chart)
4. [Templates: funciones y helpers](#4-templates-funciones-y-helpers)
5. [Dependencias entre charts](#5-dependencias-entre-charts)
6. [Comandos de Helm](#6-comandos-de-helm)
7. [Kustomize: bases y overlays](#7-kustomize-bases-y-overlays)
8. [Helm vs Kustomize: cuando usar cada uno](#8-helm-vs-kustomize-cuando-usar-cada-uno)
9. [Empaquetando una app Spring Boot con Helm](#9-empaquetando-una-app-spring-boot-con-helm)
10. [Resumen](#resumen)
11. [Ejercicios](#ejercicios)
12. [Recursos](#recursos)

---

## 1. Por que necesitas un package manager para K8s

### 1.1 El problema de los manifiestos crudos

Imagina que tienes una app con Deployment, Service, ConfigMap, Secret, Ingress, HPA,
PVC y ServiceAccount. Son 8 archivos YAML. Ahora multiplica por 3 entornos (dev,
staging, prod) donde solo cambian la imagen, las replicas, los recursos y el dominio.

```
Sin Helm/Kustomize:
k8s/
  dev/
    deployment.yaml       # 90% identico al de prod
    service.yaml          # identico al de prod
    configmap.yaml        # diferente dominio de BD
    secret.yaml           # diferentes credenciales
    ingress.yaml          # diferente dominio
    hpa.yaml              # menos replicas
    pvc.yaml              # menos storage
    serviceaccount.yaml   # identico
  staging/
    (mismos 8 archivos con otros valores)
  prod/
    (mismos 8 archivos con otros valores)

Total: 24 archivos con 80% de duplicacion.
Un cambio en la estructura del Deployment requiere editar 3 archivos.
```

### 1.2 La solucion: templates o transformaciones

```
Helm (templates):
  chart/
    templates/deployment.yaml    # template con {{ .Values.replicas }}
    values.yaml                  # valores por defecto
    values-dev.yaml              # override para dev
    values-prod.yaml             # override para prod

  helm install mi-api ./chart -f values-prod.yaml

Kustomize (transformaciones):
  base/
    deployment.yaml              # manifiesto base
    service.yaml
    kustomization.yaml
  overlays/dev/
    kustomization.yaml           # parches sobre la base
    replicas-patch.yaml
  overlays/prod/
    kustomization.yaml
    replicas-patch.yaml

  kubectl apply -k overlays/prod/
```

---

## 2. Helm: conceptos fundamentales

### 2.1 Que es Helm

Helm es el package manager de Kubernetes. Piensa en el como `apt` para Ubuntu o
`brew` para macOS, pero para aplicaciones de Kubernetes.

### 2.2 Tres conceptos clave

| Concepto | Que es | Analogia |
|----------|--------|----------|
| **Chart** | Paquete de templates YAML + valores + metadata | Un `.deb` o `.rpm` |
| **Release** | Una instancia instalada de un chart en el cluster | Un paquete instalado |
| **Repository** | Coleccion de charts disponibles | Un repositorio `apt` o `brew tap` |

```
Chart (paquete)  -->  helm install  -->  Release (instancia en el cluster)
                                          mi-api-prod (release name)
                                          mi-api-dev  (otra release del mismo chart)
```

### 2.3 Instalar Helm

```bash
# Linux
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# macOS
brew install helm

# Verificar
helm version
# version.BuildInfo{Version:"v3.14.0", ...}
```

### 2.4 Repositorios de charts

```bash
# Agregar repositorios publicos
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo add prometheus https://prometheus-community.github.io/helm-charts

# Actualizar el indice local
helm repo update

# Buscar charts
helm search repo postgresql
# NAME                    CHART VERSION   APP VERSION   DESCRIPTION
# bitnami/postgresql      14.0.5          16.2.0        PostgreSQL...

# Ver valores configurables de un chart
helm show values bitnami/postgresql

# Instalar un chart del repositorio
helm install mi-postgres bitnami/postgresql \
    --namespace mi-app \
    --create-namespace \
    --set auth.postgresPassword=secreto \
    --set primary.persistence.size=10Gi
```

---

## 3. Estructura de un chart

### 3.1 Crear un chart nuevo

```bash
# Scaffold de un chart nuevo
helm create mi-api-chart

# Estructura generada:
mi-api-chart/
  Chart.yaml              # Metadata del chart (nombre, version, descripcion)
  values.yaml             # Valores por defecto
  charts/                 # Charts de los que depende (subdependencias)
  templates/              # Templates YAML de Kubernetes
    deployment.yaml
    service.yaml
    serviceaccount.yaml
    ingress.yaml
    hpa.yaml
    NOTES.txt             # Mensaje post-instalacion
    _helpers.tpl          # Funciones helper reutilizables
    tests/
      test-connection.yaml
  .helmignore             # Archivos a excluir del paquete
```

### 3.2 Chart.yaml

```yaml
# Chart.yaml
apiVersion: v2                                   # Helm 3
name: mi-api-chart                               # nombre del chart
description: Chart de Helm para mi API Spring Boot
type: application                                # "application" o "library"
version: 1.0.0                                   # version del chart (semver)
appVersion: "2.1.0"                              # version de la aplicacion

# Metadata opcional
home: https://github.com/mi-org/mi-api
sources:
  - https://github.com/mi-org/mi-api
maintainers:
  - name: Equipo Backend
    email: backend@empresa.com

# Dependencias (otros charts que necesita)
dependencies:
  - name: postgresql
    version: "14.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled                 # solo si postgresql.enabled=true
```

### 3.3 values.yaml

```yaml
# values.yaml (valores por defecto, sobreescribibles en instalacion)

# Imagen Docker
image:
  repository: mi-registro/mi-api
  tag: "2.1.0"                                   # sobreescribir en cada despliegue
  pullPolicy: IfNotPresent

# Replicas
replicaCount: 2

# Configuracion de Spring Boot
spring:
  profiles: prod
  datasource:
    url: jdbc:postgresql://mi-postgres:5432/midb
    username: miapp

# Recursos
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 8080

# Ingress
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: api.midominio.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls
      hosts:
        - api.midominio.com

# HPA
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

# Probes
probes:
  liveness:
    path: /actuator/health/liveness
    initialDelaySeconds: 30
    periodSeconds: 10
  readiness:
    path: /actuator/health/readiness
    initialDelaySeconds: 10
    periodSeconds: 5
  startup:
    path: /actuator/health/liveness
    failureThreshold: 30
    periodSeconds: 5

# PostgreSQL (dependencia)
postgresql:
  enabled: true
  auth:
    database: midb
    username: miapp
    existingSecret: mi-api-db-secret
  primary:
    persistence:
      size: 10Gi

# ServiceAccount
serviceAccount:
  create: true
  name: ""                                       # si vacio, se genera automaticamente
```

### 3.4 Templates

Los templates usan la sintaxis de Go templates con datos de `values.yaml`:

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "mi-api-chart.fullname" . }}  # helper del _helpers.tpl
  labels:
    {{- include "mi-api-chart.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}           # solo si no hay HPA
  {{- end }}
  selector:
    matchLabels:
      {{- include "mi-api-chart.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "mi-api-chart.selectorLabels" . | nindent 8 }}
      annotations:
        # Forzar rolling restart cuando cambia el ConfigMap
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: {{ include "mi-api-chart.serviceAccountName" . }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: {{ .Values.spring.profiles | quote }}
            - name: SPRING_DATASOURCE_URL
              value: {{ .Values.spring.datasource.url | quote }}
            - name: SPRING_DATASOURCE_USERNAME
              value: {{ .Values.spring.datasource.username | quote }}
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "mi-api-chart.fullname" . }}-db
                  key: password
          startupProbe:
            httpGet:
              path: {{ .Values.probes.startup.path }}
              port: http
            failureThreshold: {{ .Values.probes.startup.failureThreshold }}
            periodSeconds: {{ .Values.probes.startup.periodSeconds }}
          livenessProbe:
            httpGet:
              path: {{ .Values.probes.liveness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.liveness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.liveness.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.probes.readiness.path }}
              port: http
            initialDelaySeconds: {{ .Values.probes.readiness.initialDelaySeconds }}
            periodSeconds: {{ .Values.probes.readiness.periodSeconds }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
```

---

## 4. Templates: funciones y helpers

### 4.1 Sintaxis basica de Go templates

```yaml
# Acceder a valores
{{ .Values.replicaCount }}                       # valor de values.yaml
{{ .Release.Name }}                              # nombre de la release
{{ .Release.Namespace }}                         # namespace
{{ .Chart.Name }}                                # nombre del chart
{{ .Chart.AppVersion }}                          # version de la app

# Condicionales
{{- if .Values.ingress.enabled }}
# ... recurso Ingress ...
{{- end }}

{{- if and .Values.autoscaling.enabled (gt (int .Values.autoscaling.maxReplicas) 1) }}
# ... HPA solo si esta habilitado y maxReplicas > 1 ...
{{- end }}

# Bucles
{{- range .Values.ingress.hosts }}
  - host: {{ .host | quote }}
    http:
      paths:
        {{- range .paths }}
        - path: {{ .path }}
          pathType: {{ .pathType }}
        {{- end }}
{{- end }}

# Valores por defecto
{{ .Values.image.tag | default .Chart.AppVersion }}

# Quitar espacios en blanco (el guion - elimina whitespace)
{{- if .Values.ingress.enabled -}}
# Sin el guion, se generarian lineas vacias
{{- end -}}
```

### 4.2 Funciones utiles

```yaml
# quote: envolver en comillas
value: {{ .Values.name | quote }}                # "mi-valor"

# upper/lower: mayusculas/minusculas
value: {{ .Values.env | upper }}                 # PROD

# default: valor por defecto
value: {{ .Values.tag | default "latest" }}

# required: falla si el valor no existe
value: {{ required "image.repository es obligatorio" .Values.image.repository }}

# toYaml: convertir estructura Go a YAML
resources:
  {{- toYaml .Values.resources | nindent 4 }}

# nindent: nueva linea + indentacion
labels:
  {{- include "mi-chart.labels" . | nindent 4 }}
# indent: indentacion sin nueva linea

# sha256sum: hash (util para forzar rolling restart)
checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}

# tpl: renderizar un string como template
# Util para valores que contienen referencias a otros valores
annotations:
  {{- tpl .Values.podAnnotations . | nindent 8 }}

# ternary: if-else inline
replicas: {{ ternary 1 .Values.replicaCount .Values.autoscaling.enabled }}
```

### 4.3 _helpers.tpl

El archivo `_helpers.tpl` define funciones reutilizables en todos los templates:

```yaml
# templates/_helpers.tpl

# Nombre completo (truncado a 63 chars para cumplir con DNS)
{{- define "mi-api-chart.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

# Labels estandar
{{- define "mi-api-chart.labels" -}}
helm.sh/chart: {{ include "mi-api-chart.chart" . }}
{{ include "mi-api-chart.selectorLabels" . }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

# Selector labels (usados en matchLabels)
{{- define "mi-api-chart.selectorLabels" -}}
app.kubernetes.io/name: {{ include "mi-api-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

# Nombre del ServiceAccount
{{- define "mi-api-chart.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "mi-api-chart.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

### 4.4 NOTES.txt

```
# templates/NOTES.txt
Desplegado {{ .Chart.Name }} version {{ .Chart.AppVersion }}

Release: {{ .Release.Name }}
Namespace: {{ .Release.Namespace }}

{{- if .Values.ingress.enabled }}
Accede a la aplicacion en:
{{- range .Values.ingress.hosts }}
  https://{{ .host }}
{{- end }}
{{- else }}
Para acceder localmente:
  kubectl port-forward svc/{{ include "mi-api-chart.fullname" . }} 8080:{{ .Values.service.port }} -n {{ .Release.Namespace }}
{{- end }}
```

---

## 5. Dependencias entre charts

### 5.1 Declarar dependencias

```yaml
# Chart.yaml
dependencies:
  - name: postgresql
    version: "14.0.5"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled                 # instalar solo si es true
  - name: redis
    version: "18.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: redis.enabled
```

### 5.2 Gestionar dependencias

```bash
# Descargar dependencias (las guarda en charts/)
helm dependency update mi-api-chart/

# Listar dependencias
helm dependency list mi-api-chart/

# La estructura queda:
mi-api-chart/
  charts/
    postgresql-14.0.5.tgz                        # chart descargado
    redis-18.0.0.tgz
```

### 5.3 Configurar dependencias en values.yaml

Los valores para un chart dependiente se anidan bajo su nombre:

```yaml
# values.yaml
postgresql:
  enabled: true                                  # activa la condicion del Chart.yaml
  auth:
    database: midb
    username: miapp
    postgresPassword: admin-secreto
  primary:
    persistence:
      size: 10Gi
      storageClass: fast-ssd

redis:
  enabled: false                                 # no instalar Redis
```

---

## 6. Comandos de Helm

### 6.1 Ciclo de vida completo

```bash
# --- Instalar ---
# Desde un directorio local
helm install mi-api ./mi-api-chart \
    --namespace mi-app \
    --create-namespace \
    --values values-prod.yaml

# Desde un repositorio
helm install mi-postgres bitnami/postgresql \
    --namespace mi-app \
    --set auth.postgresPassword=secreto

# Dry run (ver que se generaria sin instalar)
helm install mi-api ./mi-api-chart \
    --dry-run --debug

# --- Actualizar ---
# Cambiar valores
helm upgrade mi-api ./mi-api-chart \
    --namespace mi-app \
    --set image.tag=2.2.0

# Con archivo de valores
helm upgrade mi-api ./mi-api-chart \
    --namespace mi-app \
    --values values-prod.yaml

# Install or upgrade (idemponente)
helm upgrade --install mi-api ./mi-api-chart \
    --namespace mi-app \
    --create-namespace \
    --values values-prod.yaml

# --- Rollback ---
helm rollback mi-api 1 -n mi-app                # volver a revision 1

# --- Desinstalar ---
helm uninstall mi-api -n mi-app

# --- Informacion ---
helm list -n mi-app                              # releases instaladas
helm status mi-api -n mi-app                     # estado de una release
helm history mi-api -n mi-app                    # historial de revisiones
helm get values mi-api -n mi-app                 # valores usados
helm get manifest mi-api -n mi-app               # YAML generado
```

### 6.2 Renderizar templates localmente

```bash
# Ver el YAML generado sin instalar
helm template mi-api ./mi-api-chart \
    --values values-prod.yaml \
    --namespace mi-app

# Renderizar un solo template
helm template mi-api ./mi-api-chart \
    --show-only templates/deployment.yaml

# Validar el chart
helm lint ./mi-api-chart
# ==> Linting ./mi-api-chart
# [INFO] Chart.yaml: icon is recommended
# 1 chart(s) linted, 0 chart(s) failed
```

### 6.3 Empaquetar y publicar

```bash
# Empaquetar el chart (genera .tgz)
helm package ./mi-api-chart
# mi-api-chart-1.0.0.tgz

# Publicar en un registry OCI (como GitHub Container Registry)
helm push mi-api-chart-1.0.0.tgz oci://ghcr.io/mi-org/charts

# Instalar desde OCI
helm install mi-api oci://ghcr.io/mi-org/charts/mi-api-chart --version 1.0.0
```

---

## 7. Kustomize: bases y overlays

### 7.1 Que es Kustomize

Kustomize es una herramienta integrada en kubectl que permite personalizar manifiestos
YAML sin templates. En lugar de parametrizar con `{{ }}`, defines una base y aplicas
parches (transformaciones) por entorno.

### 7.2 Estructura basica

```
kustomize/
  base/                                          # manifiestos base (comunes a todos los entornos)
    deployment.yaml
    service.yaml
    configmap.yaml
    kustomization.yaml
  overlays/                                      # modificaciones por entorno
    dev/
      kustomization.yaml
      replicas-patch.yaml
    staging/
      kustomization.yaml
      resources-patch.yaml
    prod/
      kustomization.yaml
      replicas-patch.yaml
      ingress.yaml
```

### 7.3 Base

```yaml
# base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:                                       # manifiestos que incluir
  - deployment.yaml
  - service.yaml
  - configmap.yaml

commonLabels:                                    # labels que se aplican a TODOS los recursos
  app: mi-api

commonAnnotations:
  team: backend
```

```yaml
# base/deployment.yaml (manifiesto YAML normal, sin templates)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mi-api
  template:
    metadata:
      labels:
        app: mi-api
    spec:
      containers:
        - name: api
          image: mi-api:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
```

### 7.4 Overlays

```yaml
# overlays/dev/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base                                   # heredar de la base

namespace: dev                                   # sobreescribir namespace

namePrefix: dev-                                 # prefijo para todos los nombres

images:                                          # sobreescribir imagen
  - name: mi-api
    newName: mi-registro/mi-api
    newTag: dev-abc123

patches:                                         # parches sobre la base
  - path: replicas-patch.yaml
```

```yaml
# overlays/dev/replicas-patch.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-api
spec:
  replicas: 1                                    # en dev solo 1 replica
  template:
    spec:
      containers:
        - name: api
          resources:
            requests:
              memory: "128Mi"
              cpu: "100m"
            limits:
              memory: "256Mi"
              cpu: "250m"
```

```yaml
# overlays/prod/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
  - ingress.yaml                                 # recurso adicional solo en prod

namespace: prod

images:
  - name: mi-api
    newName: mi-registro/mi-api
    newTag: "2.1.0"

patches:
  - path: replicas-patch.yaml

configMapGenerator:                              # generar ConfigMaps
  - name: mi-api-config
    literals:
      - SPRING_PROFILES_ACTIVE=prod
      - SERVER_PORT=8080

secretGenerator:                                 # generar Secrets
  - name: mi-api-secrets
    literals:
      - DB_PASSWORD=secreto-prod
    type: Opaque
```

### 7.5 Aplicar con kubectl

```bash
# Ver el YAML generado (sin aplicar)
kubectl kustomize overlays/dev/

# Aplicar directamente
kubectl apply -k overlays/dev/

# Aplicar produccion
kubectl apply -k overlays/prod/

# Eliminar
kubectl delete -k overlays/dev/
```

### 7.6 Transformaciones de Kustomize

| Transformacion | Ejemplo | Efecto |
|---------------|---------|--------|
| `namespace` | `namespace: prod` | Cambia el namespace de todos los recursos |
| `namePrefix` | `namePrefix: prod-` | Agrega prefijo a todos los nombres |
| `nameSuffix` | `nameSuffix: -v2` | Agrega sufijo a todos los nombres |
| `commonLabels` | `app: mi-api` | Agrega labels a todos los recursos |
| `images` | `newTag: "2.0"` | Cambia tag/nombre de imagenes |
| `configMapGenerator` | literales o archivos | Genera ConfigMaps con hash en el nombre |
| `secretGenerator` | literales o archivos | Genera Secrets con hash en el nombre |
| `patches` | parche YAML | Sobreescribir campos especificos |
| `patchesJson6902` | operaciones JSON Patch | Parches mas precisos |

---

## 8. Helm vs Kustomize: cuando usar cada uno

### 8.1 Comparativa directa

| Aspecto | Helm | Kustomize |
|---------|------|-----------|
| Enfoque | Templates con parametros | Transformaciones sobre YAML base |
| Complejidad | Media-alta (Go templates) | Baja (YAML puro) |
| Curva de aprendizaje | Mas empinada | Mas suave |
| Ecosistema | Miles de charts publicos (Bitnami, etc.) | No tiene "paquetes" compartibles |
| Dependencias | Si (charts dentro de charts) | No tiene concepto de dependencias |
| Releases | Si (historial, rollback nativo) | No (usa kubectl apply) |
| Logica condicional | Si (if/else, range, funciones) | Limitada (solo parches) |
| Validacion | `helm lint`, `helm template` | `kubectl kustomize` |
| Integracion | Herramienta separada | Integrado en kubectl (`-k`) |
| Casos de uso | Apps complejas, charts publicos, CI/CD | Apps simples, overlays por entorno |

### 8.2 Cuando usar Helm

- Necesitas instalar software de terceros (PostgreSQL, Redis, NGINX, Prometheus)
- Tu app tiene logica condicional compleja (features opcionales, multiples modos)
- Quieres compartir tu app como paquete reutilizable
- Necesitas gestion de releases (historial, rollback)
- Tu pipeline CI/CD ya esta integrado con Helm

### 8.3 Cuando usar Kustomize

- Tu app es simple y solo cambian unos pocos valores entre entornos
- No quieres aprender Go templates
- Prefieres trabajar con YAML plano que se pueda leer directamente
- Quieres algo integrado en kubectl sin instalar herramientas extra
- Solo necesitas cambiar namespace, imagen, replicas y algunos configs

### 8.4 Combinar ambos

Es posible y comun usar Helm para generar la base y Kustomize para personalizarla:

```bash
# Renderizar Helm a YAML plano, luego aplicar Kustomize
helm template mi-api ./mi-api-chart --values values-base.yaml > base/all.yaml

# Estructura
kustomize-over-helm/
  base/
    all.yaml                                     # output de helm template
    kustomization.yaml
  overlays/prod/
    kustomization.yaml
    patches.yaml
```

---

## 9. Empaquetando una app Spring Boot con Helm

### 9.1 Chart completo para Spring Boot

```bash
# Crear el chart
helm create spring-boot-chart
cd spring-boot-chart
```

```yaml
# Chart.yaml
apiVersion: v2
name: spring-boot-chart
description: Chart generico para apps Spring Boot
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "14.x.x"
    repository: https://charts.bitnami.com/bitnami
    condition: postgresql.enabled
```

```yaml
# values.yaml
replicaCount: 2

image:
  repository: mi-registro/mi-spring-api
  tag: ""                                        # se sobreescribe en CI/CD
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

spring:
  profiles: prod
  javaOpts: "-Xmx384m -Xms256m -XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "1000m"

autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

probes:
  startup:
    enabled: true
    path: /actuator/health/liveness
    failureThreshold: 30
    periodSeconds: 5
  liveness:
    enabled: true
    path: /actuator/health/liveness
    initialDelaySeconds: 0
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 3
  readiness:
    enabled: true
    path: /actuator/health/readiness
    initialDelaySeconds: 0
    periodSeconds: 5
    timeoutSeconds: 3
    failureThreshold: 3

env: []
envFrom: []

configMap:
  enabled: false
  data: {}

secret:
  enabled: false
  stringData: {}

postgresql:
  enabled: false
  auth:
    database: midb
    username: miapp

serviceAccount:
  create: true
  name: ""
```

### 9.2 Archivos de valores por entorno

```yaml
# values-dev.yaml
replicaCount: 1

image:
  tag: "dev-latest"

spring:
  profiles: dev
  javaOpts: "-Xmx256m -Xms128m"

resources:
  requests:
    memory: "128Mi"
    cpu: "100m"
  limits:
    memory: "256Mi"
    cpu: "500m"

autoscaling:
  enabled: false

postgresql:
  enabled: true
  auth:
    postgresPassword: dev-secreto
```

```yaml
# values-prod.yaml
replicaCount: 3

image:
  tag: "2.1.0"

spring:
  profiles: prod
  javaOpts: "-Xmx384m -Xms256m -XX:+UseContainerSupport"

ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: api.midominio.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: api-tls-cert
      hosts:
        - api.midominio.com

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 15
  targetCPUUtilizationPercentage: 70

resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "1Gi"
    cpu: "2000m"
```

### 9.3 Desplegar

```bash
# Descargar dependencias
helm dependency update ./spring-boot-chart

# Desplegar en dev
helm upgrade --install mi-api ./spring-boot-chart \
    --namespace dev \
    --create-namespace \
    --values values-dev.yaml

# Desplegar en prod
helm upgrade --install mi-api ./spring-boot-chart \
    --namespace prod \
    --create-namespace \
    --values values-prod.yaml

# Verificar
helm list -A
# NAME     NAMESPACE   REVISION   STATUS     CHART
# mi-api   dev         1          deployed   spring-boot-chart-1.0.0
# mi-api   prod        1          deployed   spring-boot-chart-1.0.0

# Ver los recursos generados
kubectl get all -n prod
```

### 9.4 Integracion con CI/CD

```yaml
# .github/workflows/deploy.yml (fragmento)
- name: Deploy to Kubernetes
  run: |
    helm upgrade --install mi-api ./spring-boot-chart \
      --namespace prod \
      --set image.tag=${{ github.sha }} \
      --values values-prod.yaml \
      --wait \
      --timeout 5m
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| Helm | Package manager para Kubernetes | `helm install`, `helm upgrade`, `helm rollback` |
| Chart | Paquete de templates YAML + valores | Directorio con `Chart.yaml`, `values.yaml`, `templates/` |
| Release | Instancia instalada de un chart | `helm list` para ver releases activas |
| Repository | Coleccion de charts | `helm repo add`, `helm search repo` |
| values.yaml | Valores por defecto del chart | Sobreescribir con `--values` o `--set` |
| Go templates | Sintaxis de templates en Helm | `{{ .Values.x }}`, `{{ if }}`, `{{ range }}` |
| _helpers.tpl | Funciones helper reutilizables | `{{ include "chart.fullname" . }}` |
| Chart dependencies | Charts que depende el tuyo | `dependencies` en Chart.yaml, `helm dependency update` |
| Kustomize | Transformaciones sobre YAML base | `kubectl apply -k overlays/prod/` |
| Base | Manifiestos YAML comunes | Directorio con `kustomization.yaml` y recursos |
| Overlay | Parches por entorno sobre la base | `namespace`, `images`, `patches` |
| configMapGenerator | Genera ConfigMaps con hash | Fuerza rolling restart cuando cambian |
| `helm template` | Renderizar sin instalar | Depuracion y validacion |
| `helm lint` | Validar un chart | Encontrar errores antes de instalar |
| OCI Registry | Publicar charts como imagenes | `helm push`, `helm pull` con oci:// |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **Crear un Helm chart** | Crear un chart desde cero para una app Spring Boot con Deployment, Service, ConfigMap y probes. Instalar en Minikube y verificar | `helm create`, Chart.yaml, values.yaml, templates, `helm install` |
| 02 | **Values y templates** | Parametrizar el chart con values para multiples entornos (dev, prod). Usar condicionales para habilitar/deshabilitar Ingress y HPA. Renderizar con `helm template` | Go templates, `{{ if }}`, `{{ range }}`, `--values`, `--set` |
| 03 | **Kustomize overlays** | Crear una estructura base/overlays para la misma app. Definir overlays para dev (1 replica, menos recursos) y prod (3 replicas, Ingress). Aplicar con `kubectl -k` | Kustomize, bases, overlays, patches, `configMapGenerator`, `images` |
| 04 | **Chart Spring Boot completo** | Crear un chart completo con dependencia de PostgreSQL, Secret para credenciales, HPA, Ingress con TLS y probes. Desplegar en dev y prod con diferentes values | dependencies, `helm dependency update`, multi-entorno, CI/CD |

---

## Recursos

- [Helm Documentation](https://helm.sh/docs/)
- [Helm Chart Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Go Template Language](https://pkg.go.dev/text/template)
- [Artifact Hub (charts publicos)](https://artifacthub.io/)
- [Kustomize Documentation](https://kustomize.io/)
- [Kustomize Built-in Transformers](https://kubectl.docs.kubernetes.io/references/kustomize/builtins/)
- [Bitnami Charts](https://github.com/bitnami/charts)
- [Helm vs Kustomize (blog)](https://helm.sh/blog/)

---

> **Antes de continuar al siguiente nivel**: asegurate de que puedes crear un chart
> de Helm desde cero, parametrizarlo con values y desplegarlo en diferentes entornos.
> Helm es la herramienta estandar para desplegar aplicaciones en Kubernetes en la
> mayoria de empresas. Si solo aprendes una herramienta de este nivel, que sea Helm.
