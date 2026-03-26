# Nivel 08 — GitOps con ArgoCD

> Tu infraestructura ya se define en codigo (Terraform) y tu pipeline construye imagenes Docker. Ahora necesitas que el estado deseado de tus aplicaciones en Kubernetes viva en Git y se sincronice automaticamente. Eso es GitOps: Git como fuente unica de verdad para los despliegues.

---

## Contenido

- [1. Principios de GitOps](#1-principios-de-gitops)
  - [1.1 Que es GitOps](#11-que-es-gitops)
  - [1.2 Los cuatro principios](#12-los-cuatro-principios)
  - [1.3 Push vs Pull model](#13-push-vs-pull-model)
  - [1.4 GitOps vs CI/CD tradicional](#14-gitops-vs-cicd-tradicional)
- [2. ArgoCD: arquitectura](#2-argocd-arquitectura)
  - [2.1 Componentes internos](#21-componentes-internos)
  - [2.2 Como funciona el ciclo de sync](#22-como-funciona-el-ciclo-de-sync)
- [3. Instalacion de ArgoCD en Kubernetes](#3-instalacion-de-argocd-en-kubernetes)
  - [3.1 Instalacion basica](#31-instalacion-basica)
  - [3.2 Acceso a la UI](#32-acceso-a-la-ui)
  - [3.3 CLI de ArgoCD](#33-cli-de-argocd)
- [4. Application CRD](#4-application-crd)
  - [4.1 Estructura de una Application](#41-estructura-de-una-application)
  - [4.2 Source: de donde viene la config](#42-source-de-donde-viene-la-config)
  - [4.3 Destination: a donde se despliega](#43-destination-a-donde-se-despliega)
  - [4.4 Ejemplo completo](#44-ejemplo-completo)
- [5. Estrategias de sincronizacion](#5-estrategias-de-sincronizacion)
  - [5.1 Sync manual vs automatico](#51-sync-manual-vs-automatico)
  - [5.2 Sync options](#52-sync-options)
  - [5.3 Self-healing y auto-prune](#53-self-healing-y-auto-prune)
- [6. Sync waves y hooks](#6-sync-waves-y-hooks)
  - [6.1 Sync waves](#61-sync-waves)
  - [6.2 Resource hooks](#62-resource-hooks)
- [7. Health checks](#7-health-checks)
  - [7.1 Health checks nativos](#71-health-checks-nativos)
  - [7.2 Custom health checks](#72-custom-health-checks)
- [8. Rollbacks](#8-rollbacks)
  - [8.1 Rollback desde la UI](#81-rollback-desde-la-ui)
  - [8.2 Rollback desde la CLI](#82-rollback-desde-la-cli)
  - [8.3 Rollback con Git revert](#83-rollback-con-git-revert)
- [9. Multi-environment management](#9-multi-environment-management)
  - [9.1 Estrategias de organizacion](#91-estrategias-de-organizacion)
  - [9.2 Estructura de directorios](#92-estructura-de-directorios)
  - [9.3 Kustomize overlays](#93-kustomize-overlays)
  - [9.4 Helm values por entorno](#94-helm-values-por-entorno)
- [10. Sealed Secrets](#10-sealed-secrets)
  - [10.1 El problema de los secretos en Git](#101-el-problema-de-los-secretos-en-git)
  - [10.2 Como funciona Sealed Secrets](#102-como-funciona-sealed-secrets)
  - [10.3 Uso practico](#103-uso-practico)
- [11. ApplicationSets](#11-applicationsets)
  - [11.1 Que son los ApplicationSets](#111-que-son-los-applicationsets)
  - [11.2 List generator](#112-list-generator)
  - [11.3 Git generator](#113-git-generator)
  - [11.4 Matrix generator](#114-matrix-generator)
- [12. Buenas practicas](#12-buenas-practicas)
- [13. Resumen](#13-resumen)
- [14. Ejercicios](#14-ejercicios)
- [15. Recursos](#15-recursos)

---

## 1. Principios de GitOps

### 1.1 Que es GitOps

GitOps es un paradigma operacional donde el estado deseado de tu infraestructura y aplicaciones se almacena en un repositorio Git. Un agente (como ArgoCD) se encarga de que el estado real en el cluster coincida con lo que dice Git.

```
Flujo tradicional:                    Flujo GitOps:

Developer -> CI -> kubectl apply      Developer -> Git push -> ArgoCD -> Kubernetes
                   (imperativo)                    (declarativo, automatico)
```

### 1.2 Los cuatro principios

| Principio | Significado | En la practica |
|-----------|-------------|---------------|
| **Declarativo** | Describes el estado deseado, no los pasos | Manifiestos YAML de Kubernetes en Git |
| **Versionado** | Todo cambio queda en el historial de Git | `git log` muestra quien cambio que y cuando |
| **Automatico** | Los cambios se aplican automaticamente | ArgoCD detecta cambios en Git y sincroniza |
| **Self-healing** | Si alguien modifica el cluster a mano, se revierte | ArgoCD detecta drift y restaura al estado de Git |

La consecuencia: **Git es la unica fuente de verdad**. Nadie ejecuta `kubectl apply` manualmente. Todo cambio pasa por un PR en Git.

### 1.3 Push vs Pull model

**Push model** (CI/CD tradicional): el pipeline ejecuta `kubectl apply` contra el cluster.

```
CI Pipeline ----push----> Kubernetes
(necesita credenciales del cluster)
```

Problemas:
- El CI tiene credenciales de produccion
- Si el CI se compromete, tienen acceso al cluster
- No hay reconciliacion automatica (si alguien modifica el cluster a mano, no se revierte)

**Pull model** (GitOps): un agente dentro del cluster lee Git y aplica los cambios.

```
Git Repository <----pull---- ArgoCD (dentro del cluster)
                                |
                                v
                            Kubernetes
```

Ventajas:
- Las credenciales del cluster nunca salen del cluster
- ArgoCD reconcilia continuamente (self-healing)
- El CI solo necesita permisos para pushear a Git

### 1.4 GitOps vs CI/CD tradicional

| Aspecto | CI/CD tradicional | GitOps |
|---------|-------------------|--------|
| Quien despliega | El pipeline (push) | El agente en el cluster (pull) |
| Fuente de verdad | Lo que se ejecuto por ultimo | Lo que dice Git |
| Drift detection | No existe | Continua |
| Rollback | Re-ejecutar pipeline antiguo | `git revert` |
| Auditoria | Logs del CI | Historial de Git |
| Seguridad | CI tiene credenciales del cluster | Credenciales no salen del cluster |

---

## 2. ArgoCD: arquitectura

### 2.1 Componentes internos

ArgoCD tiene tres componentes principales:

```
                    Git Repository
                         |
                         v
                  +-------------+
                  | Repo Server  |    <- Clona repos, renderiza manifiestos
                  +-------------+         (Helm, Kustomize, plain YAML)
                         |
                         v
              +---------------------+
              | Application         |  <- Compara estado deseado vs actual
              | Controller          |     Detecta drift, ejecuta sync
              +---------------------+
                         |
                         v
                  Kubernetes API
                         |
                  +-------------+
                  | API Server   |    <- UI, CLI, API REST
                  +-------------+         Autenticacion, RBAC
```

| Componente | Funcion | Detalles |
|------------|---------|----------|
| **API Server** | Interfaz para usuarios | Expone UI web, gRPC API, maneja auth |
| **Repo Server** | Gestiona repositorios Git | Clona, cachea, renderiza templates |
| **Application Controller** | Nucleo de ArgoCD | Compara estado deseado vs real, ejecuta syncs |
| **Redis** | Cache interna | Cachea estado de los repos y apps |
| **Dex** | SSO (opcional) | Integracion con LDAP, GitHub, Google |

### 2.2 Como funciona el ciclo de sync

```
1. Application Controller consulta Git (via Repo Server) cada 3 minutos
2. Repo Server clona el repo y renderiza manifiestos (Helm/Kustomize/YAML)
3. Controller compara manifiestos renderizados con lo que hay en Kubernetes
4. Si hay diferencias:
   - Auto-sync ON:  aplica cambios automaticamente
   - Auto-sync OFF: marca la app como "OutOfSync" (pendiente de accion manual)
5. Despues del sync, verifica health de los recursos
6. Reporta estado: Synced/OutOfSync + Healthy/Degraded/Progressing
```

Tambien puedes configurar **webhooks** en GitHub para que ArgoCD se entere inmediatamente de un push (en lugar de esperar el polling de 3 minutos).

---

## 3. Instalacion de ArgoCD en Kubernetes

### 3.1 Instalacion basica

```bash
# Crear namespace
kubectl create namespace argocd

# Instalar ArgoCD (version estable)
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Verificar que los pods estan corriendo
kubectl get pods -n argocd
# NAME                                  READY   STATUS    RESTARTS   AGE
# argocd-application-controller-0       1/1     Running   0          60s
# argocd-repo-server-xxx                1/1     Running   0          60s
# argocd-server-xxx                     1/1     Running   0          60s
# argocd-redis-xxx                      1/1     Running   0          60s
# argocd-dex-server-xxx                 1/1     Running   0          60s
```

Para produccion, se recomienda instalar con Helm para tener mas control:

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd \
  --namespace argocd \
  --create-namespace \
  --values argocd-values.yaml
```

```yaml
# argocd-values.yaml
server:
  replicas: 2                          # alta disponibilidad
  ingress:
    enabled: true
    hosts:
      - argocd.mi-empresa.com
    tls:
      - secretName: argocd-tls
        hosts:
          - argocd.mi-empresa.com

controller:
  replicas: 1                          # el controller es singleton

repoServer:
  replicas: 2

redis-ha:
  enabled: true                        # Redis en alta disponibilidad
```

### 3.2 Acceso a la UI

```bash
# Opcion 1: Port-forward (desarrollo)
kubectl port-forward svc/argocd-server -n argocd 8443:443

# Obtener password inicial del admin
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Acceder en https://localhost:8443
# Usuario: admin
# Password: (el obtenido arriba)
```

**Importante**: cambiar el password del admin inmediatamente despues de la primera conexion y luego eliminar el Secret `argocd-initial-admin-secret`.

### 3.3 CLI de ArgoCD

```bash
# Instalar CLI
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd && sudo mv argocd /usr/local/bin/

# Login
argocd login localhost:8443 --username admin --password <password> --insecure

# Cambiar password
argocd account update-password

# Listar aplicaciones
argocd app list

# Ver estado de una app
argocd app get mi-app

# Sincronizar una app
argocd app sync mi-app

# Ver historial de syncs
argocd app history mi-app
```

---

## 4. Application CRD

### 4.1 Estructura de una Application

ArgoCD introduce el CRD (Custom Resource Definition) `Application` que define que desplegar, de donde viene y a donde va:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mi-app                        # nombre en ArgoCD
  namespace: argocd                    # siempre en el namespace de ArgoCD
spec:
  project: default                     # proyecto de ArgoCD (RBAC)

  source:                              # DE DONDE viene la configuracion
    repoURL: https://github.com/mi-org/k8s-manifests.git
    targetRevision: main               # rama, tag o commit
    path: apps/mi-app                  # directorio dentro del repo

  destination:                         # A DONDE se despliega
    server: https://kubernetes.default.svc   # cluster (in-cluster)
    namespace: mi-app                  # namespace de Kubernetes

  syncPolicy:                          # COMO se sincroniza
    automated:
      selfHeal: true                   # revertir cambios manuales
      prune: true                      # eliminar recursos huerfanos
    syncOptions:
      - CreateNamespace=true           # crear namespace si no existe
```

### 4.2 Source: de donde viene la config

ArgoCD soporta multiples fuentes:

```yaml
# Plain YAML (directorio con manifiestos)
source:
  repoURL: https://github.com/mi-org/k8s-manifests.git
  targetRevision: main
  path: apps/mi-app

# Helm chart desde un repo
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: postgresql                    # nombre del chart
  targetRevision: 13.2.0              # version del chart
  helm:
    values: |
      primary:
        persistence:
          size: 20Gi
      auth:
        postgresPassword: secreto

# Helm chart desde Git
source:
  repoURL: https://github.com/mi-org/helm-charts.git
  targetRevision: main
  path: charts/mi-app
  helm:
    valueFiles:
      - values-production.yaml

# Kustomize
source:
  repoURL: https://github.com/mi-org/k8s-manifests.git
  targetRevision: main
  path: overlays/production
  kustomize:
    images:
      - mi-app=ghcr.io/mi-org/mi-app:v1.2.3
```

### 4.3 Destination: a donde se despliega

```yaml
# Cluster local (donde esta ArgoCD)
destination:
  server: https://kubernetes.default.svc
  namespace: mi-app

# Cluster externo (registrado en ArgoCD)
destination:
  name: production-cluster             # nombre del cluster registrado
  namespace: mi-app
```

Registrar un cluster externo:

```bash
# Agregar cluster externo
argocd cluster add nombre-del-contexto-kubectl

# Listar clusters
argocd cluster list
```

### 4.4 Ejemplo completo

Repositorio de manifiestos (`k8s-manifests/apps/mi-app/`):

```yaml
# k8s-manifests/apps/mi-app/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
  labels:
    app: mi-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
        - name: mi-app
          image: ghcr.io/mi-org/mi-app:v1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
```

```yaml
# k8s-manifests/apps/mi-app/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-app
spec:
  selector:
    app: mi-app
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

Application de ArgoCD que apunta a estos manifiestos:

```yaml
# argocd/applications/mi-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mi-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io   # limpiar recursos al borrar la app
spec:
  project: default
  source:
    repoURL: https://github.com/mi-org/k8s-manifests.git
    targetRevision: main
    path: apps/mi-app
  destination:
    server: https://kubernetes.default.svc
    namespace: mi-app
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
```

---

## 5. Estrategias de sincronizacion

### 5.1 Sync manual vs automatico

**Manual**: ArgoCD detecta cambios pero no los aplica. El equipo revisa y hace sync desde la UI o CLI.

```yaml
syncPolicy: {}                         # sin automated = manual
```

**Automatico**: ArgoCD aplica cambios automaticamente cuando detecta diferencias.

```yaml
syncPolicy:
  automated:
    prune: false                       # no eliminar recursos extra (conservador)
    selfHeal: false                    # no revertir cambios manuales
```

En produccion, muchos equipos usan:
- **Staging**: auto-sync con selfHeal y prune (automatico total)
- **Production**: manual sync o auto-sync sin prune (requiere revision)

### 5.2 Sync options

```yaml
syncPolicy:
  syncOptions:
    - CreateNamespace=true             # crear namespace si no existe
    - PrunePropagationPolicy=foreground # esperar a que se borren dependientes
    - PruneLast=true                   # borrar recursos obsoletos al final
    - ApplyOutOfSyncOnly=true          # solo aplicar recursos que cambiaron
    - ServerSideApply=true             # usar server-side apply de Kubernetes
    - Validate=false                   # saltar validacion (CRDs que aun no existen)
  retry:
    limit: 5                           # reintentar sync si falla
    backoff:
      duration: 5s
      factor: 2
      maxDuration: 3m
```

### 5.3 Self-healing y auto-prune

**Self-healing**: si alguien ejecuta `kubectl edit` o `kubectl scale` manualmente, ArgoCD revierte los cambios al estado de Git.

```
Estado en Git: replicas: 3
  |
  | Alguien ejecuta: kubectl scale --replicas=5
  |
Estado real: replicas: 5 (drift!)
  |
  | ArgoCD detecta drift (selfHeal: true)
  |
Estado real: replicas: 3 (restaurado)
```

**Auto-prune**: si eliminas un recurso del repositorio Git, ArgoCD lo elimina del cluster.

```
Git: deployment.yaml + service.yaml + ingress.yaml
  |
  | Eliminas ingress.yaml de Git
  |
Git: deployment.yaml + service.yaml
  |
  | ArgoCD detecta recurso huerfano (prune: true)
  |
Cluster: el Ingress se elimina automaticamente
```

---

## 6. Sync waves y hooks

### 6.1 Sync waves

Las sync waves controlan el **orden** en que se aplican los recursos. Los recursos con wave menor se aplican primero:

```yaml
# Wave -1: Namespace y configuracion (primero)
apiVersion: v1
kind: Namespace
metadata:
  name: mi-app
  annotations:
    argocd.argoproj.io/sync-wave: "-1"

---
# Wave 0: ConfigMaps y Secrets (default)
apiVersion: v1
kind: ConfigMap
metadata:
  name: mi-app-config
  annotations:
    argocd.argoproj.io/sync-wave: "0"

---
# Wave 1: Base de datos (antes de la app)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  annotations:
    argocd.argoproj.io/sync-wave: "1"

---
# Wave 2: Aplicacion (despues de la BD)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
  annotations:
    argocd.argoproj.io/sync-wave: "2"

---
# Wave 3: Ingress (al final, cuando la app esta lista)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-app
  annotations:
    argocd.argoproj.io/sync-wave: "3"
```

ArgoCD aplica todas las waves en orden y espera a que los recursos de cada wave esten healthy antes de avanzar a la siguiente.

### 6.2 Resource hooks

Los hooks ejecutan Jobs o Pods en momentos especificos del ciclo de sync:

```yaml
# Job de migracion de BD que corre ANTES del sync
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migration
  annotations:
    argocd.argoproj.io/hook: PreSync           # ejecutar antes del sync
    argocd.argoproj.io/hook-delete-policy: HookSucceeded  # borrar si tuvo exito
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: ghcr.io/mi-org/mi-app:v1.2.0
          command: ["./migrate.sh"]
      restartPolicy: Never
  backoffLimit: 3
```

Hooks disponibles:

| Hook | Cuando ejecuta |
|------|---------------|
| `PreSync` | Antes del sync (migraciones, backups) |
| `Sync` | Durante el sync (con los recursos normales) |
| `PostSync` | Despues del sync exitoso (smoke tests, notificaciones) |
| `SyncFail` | Si el sync falla (cleanup, notificacion de error) |
| `Skip` | Recurso ignorado por ArgoCD |

Politicas de borrado del hook:

| Politica | Cuando se borra el Job/Pod |
|----------|---------------------------|
| `HookSucceeded` | Cuando el hook termina con exito |
| `HookFailed` | Cuando el hook falla |
| `BeforeHookCreation` | Antes de crear una nueva instancia del hook |

---

## 7. Health checks

### 7.1 Health checks nativos

ArgoCD entiende nativamente el estado de salud de los recursos mas comunes de Kubernetes:

| Recurso | Healthy cuando |
|---------|---------------|
| Deployment | Todas las replicas disponibles |
| StatefulSet | Todas las replicas ready |
| DaemonSet | Todos los pods scheduled y ready |
| Service | Siempre healthy |
| Ingress | Tiene IP/hostname asignado |
| PersistentVolumeClaim | Status es Bound |
| Job | Completado exitosamente |
| Pod | Running y Ready |

Estados posibles de una Application:

| Estado | Significado |
|--------|-------------|
| **Healthy** | Todos los recursos estan funcionando correctamente |
| **Progressing** | Algun recurso esta en proceso de cambio (deploy en curso) |
| **Degraded** | Algun recurso tiene problemas |
| **Suspended** | Recurso pausado (ej: CronJob suspendido) |
| **Missing** | El recurso deberia existir pero no esta en el cluster |
| **Unknown** | No se puede determinar el estado |

### 7.2 Custom health checks

Para CRDs o recursos personalizados, puedes definir health checks en el ConfigMap de ArgoCD:

```yaml
# argocd-cm ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
data:
  resource.customizations.health.argoproj.io_Rollout: |
    hs = {}
    if obj.status ~= nil then
      if obj.status.conditions ~= nil then
        for _, condition in ipairs(obj.status.conditions) do
          if condition.type == "Degraded" and condition.status == "True" then
            hs.status = "Degraded"
            hs.message = condition.message
            return hs
          end
        end
      end
      if obj.status.replicas == obj.status.updatedReplicas then
        hs.status = "Healthy"
      else
        hs.status = "Progressing"
      end
    end
    return hs
```

---

## 8. Rollbacks

### 8.1 Rollback desde la UI

En la UI de ArgoCD:
1. Ir a la aplicacion
2. Click en "History and Rollback"
3. Seleccionar la revision anterior
4. Click en "Rollback"

ArgoCD despliega la version anterior del manifiesto.

### 8.2 Rollback desde la CLI

```bash
# Ver historial de deployments
argocd app history mi-app
# ID  DATE                 REVISION
# 0   2024-01-15 10:00:00  abc1234
# 1   2024-01-16 14:30:00  def5678
# 2   2024-01-17 09:00:00  ghi9012  <- actual

# Rollback a una revision anterior
argocd app rollback mi-app 1

# Verificar estado
argocd app get mi-app
```

**Gotcha**: si tienes auto-sync activado, ArgoCD detectara que el cluster no coincide con Git (que sigue en la version nueva) y volvera a sincronizar a la version nueva. Para un rollback permanente, desactiva auto-sync temporalmente o haz `git revert` en el repositorio.

### 8.3 Rollback con Git revert

El metodo recomendado en GitOps es revertir el commit en Git:

```bash
# En el repositorio de manifiestos
git log --oneline
# ghi9012 feat: update mi-app to v1.2.0  <- este rompio algo
# def5678 feat: update mi-app to v1.1.0
# abc1234 feat: initial deploy of mi-app

# Revertir el commit problematico
git revert ghi9012
git push origin main

# ArgoCD detecta el cambio en Git y sincroniza automaticamente
# El cluster vuelve al estado de v1.1.0
```

Ventajas del rollback via Git:
- Queda registro en el historial
- Funciona con auto-sync
- Cualquiera del equipo puede ver por que se revirtio

---

## 9. Multi-environment management

### 9.1 Estrategias de organizacion

Hay tres enfoques principales para gestionar multiples entornos:

| Estrategia | Como funciona | Cuando usarla |
|------------|---------------|---------------|
| **Directorios** | Un directorio por entorno | Cuando las diferencias entre entornos son grandes |
| **Ramas** | Una rama por entorno | Simple pero propenso a drift entre ramas |
| **Overlays (Kustomize)** | Base comun + patches por entorno | Cuando los entornos son similares con pocas diferencias |

La mas recomendada es **overlays con Kustomize** o **Helm values por entorno**.

### 9.2 Estructura de directorios

```
k8s-manifests/
├── apps/
│   └── mi-app/
│       ├── base/                      # configuracion comun
│       │   ├── kustomization.yaml
│       │   ├── deployment.yaml
│       │   ├── service.yaml
│       │   └── hpa.yaml
│       └── overlays/
│           ├── dev/                   # diferencias para dev
│           │   ├── kustomization.yaml
│           │   └── patch-replicas.yaml
│           ├── staging/               # diferencias para staging
│           │   ├── kustomization.yaml
│           │   └── patch-replicas.yaml
│           └── production/            # diferencias para produccion
│               ├── kustomization.yaml
│               ├── patch-replicas.yaml
│               └── patch-resources.yaml
```

### 9.3 Kustomize overlays

```yaml
# apps/mi-app/base/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
commonLabels:
  app: mi-app
```

```yaml
# apps/mi-app/base/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 1                          # default, override en overlays
  selector:
    matchLabels:
      app: mi-app
  template:
    metadata:
      labels:
        app: mi-app
    spec:
      containers:
        - name: mi-app
          image: ghcr.io/mi-org/mi-app
          resources:
            requests:
              cpu: 100m
              memory: 256Mi
```

```yaml
# apps/mi-app/overlays/production/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base
namespace: mi-app-production
images:
  - name: ghcr.io/mi-org/mi-app
    newTag: v1.2.0                     # version de produccion
patches:
  - path: patch-replicas.yaml
  - path: patch-resources.yaml
```

```yaml
# apps/mi-app/overlays/production/patch-replicas.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 5                          # 5 replicas en produccion
```

```yaml
# apps/mi-app/overlays/production/patch-resources.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  template:
    spec:
      containers:
        - name: mi-app
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "1"
              memory: 1Gi
```

Applications de ArgoCD para cada entorno:

```yaml
# argocd/applications/mi-app-dev.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mi-app-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mi-org/k8s-manifests.git
    targetRevision: main
    path: apps/mi-app/overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: mi-app-dev
  syncPolicy:
    automated:
      selfHeal: true
      prune: true
    syncOptions:
      - CreateNamespace=true
```

### 9.4 Helm values por entorno

```yaml
# ArgoCD Application con Helm
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: mi-app-production
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/mi-org/helm-charts.git
    targetRevision: main
    path: charts/mi-app
    helm:
      valueFiles:
        - values.yaml                  # base
        - values-production.yaml       # override para produccion
  destination:
    server: https://kubernetes.default.svc
    namespace: mi-app-production
```

---

## 10. Sealed Secrets

### 10.1 El problema de los secretos en Git

Los Secrets de Kubernetes van en base64 (no cifrado). Si los commiteas a Git, cualquiera con acceso al repo puede decodificarlos:

```bash
echo "bWktcGFzc3dvcmQ=" | base64 -d
# mi-password
```

Sealed Secrets resuelve esto cifrando los secretos con una clave publica. Solo el controlador dentro del cluster puede descifrarlos.

### 10.2 Como funciona Sealed Secrets

```
Tu maquina local:
  Secret (texto plano) --> kubeseal (cifra con clave publica) --> SealedSecret (cifrado)
       |                                                              |
       | (NUNCA commitear)                                           | (seguro en Git)
                                                                      |
                                                                      v
Cluster:
  SealedSecret --> Sealed Secrets Controller (descifra con clave privada) --> Secret
```

### 10.3 Uso practico

Instalar el controlador:

```bash
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system
```

Instalar la CLI `kubeseal`:

```bash
# macOS
brew install kubeseal

# Linux
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/v0.24.0/kubeseal-0.24.0-linux-amd64.tar.gz
tar xfz kubeseal-*.tar.gz
sudo mv kubeseal /usr/local/bin/
```

Crear un SealedSecret:

```bash
# 1. Crear el Secret normal (no commitear este archivo)
kubectl create secret generic mi-app-secrets \
  --from-literal=DB_PASSWORD=super-secreto \
  --from-literal=API_KEY=mi-api-key-123 \
  --dry-run=client -o yaml > secret.yaml

# 2. Cifrar con kubeseal
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# 3. Verificar que el archivo cifrado es seguro
cat sealed-secret.yaml
```

```yaml
# sealed-secret.yaml (seguro para Git)
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: mi-app-secrets
  namespace: mi-app
spec:
  encryptedData:
    DB_PASSWORD: AgBy3i... (cifrado)
    API_KEY: AgCx8j... (cifrado)
```

```bash
# 4. Commitear el SealedSecret (seguro)
git add sealed-secret.yaml
git commit -m "add encrypted secrets for mi-app"
git push

# 5. ArgoCD sincroniza -> Sealed Secrets Controller descifra -> Secret disponible
```

**Gotcha**: hacer backup de la clave privada del controlador. Si la pierdes, no podras descifrar los SealedSecrets existentes.

```bash
kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > sealed-secrets-master-key.yaml
# GUARDAR EN LUGAR SEGURO (vault, 1Password, etc.)
```

---

## 11. ApplicationSets

### 11.1 Que son los ApplicationSets

Un ApplicationSet genera multiples Applications de ArgoCD automaticamente a partir de templates y generadores. En lugar de crear manualmente una Application por cada servicio y entorno, defines un template y los parametros.

### 11.2 List generator

Genera Applications a partir de una lista explicita:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: mi-app-environments
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - env: dev
            cluster: https://kubernetes.default.svc
            replicas: "1"
          - env: staging
            cluster: https://kubernetes.default.svc
            replicas: "2"
          - env: production
            cluster: https://prod-cluster.example.com
            replicas: "5"

  template:
    metadata:
      name: 'mi-app-{{env}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/mi-org/k8s-manifests.git
        targetRevision: main
        path: 'apps/mi-app/overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: 'mi-app-{{env}}'
      syncPolicy:
        automated:
          selfHeal: true
          prune: true
        syncOptions:
          - CreateNamespace=true
```

Esto genera 3 Applications: `mi-app-dev`, `mi-app-staging`, `mi-app-production`.

### 11.3 Git generator

Genera Applications basandose en la estructura del repositorio Git:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: microservices
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/mi-org/k8s-manifests.git
        revision: main
        directories:
          - path: 'apps/*'             # una Application por cada directorio en apps/

  template:
    metadata:
      name: '{{path.basename}}'        # nombre del directorio como nombre de la app
    spec:
      project: default
      source:
        repoURL: https://github.com/mi-org/k8s-manifests.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
```

Si el repo tiene `apps/user-service/`, `apps/order-service/`, `apps/payment-service/`, ArgoCD crea automaticamente tres Applications.

### 11.4 Matrix generator

Combina dos generadores para crear el producto cartesiano:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: all-apps-all-envs
  namespace: argocd
spec:
  generators:
    - matrix:
        generators:
          - git:
              repoURL: https://github.com/mi-org/k8s-manifests.git
              revision: main
              directories:
                - path: 'apps/*/base'
          - list:
              elements:
                - env: dev
                - env: staging
                - env: production

  template:
    metadata:
      name: '{{path[1]}}-{{env}}'      # user-service-dev, user-service-staging, etc.
    spec:
      project: default
      source:
        repoURL: https://github.com/mi-org/k8s-manifests.git
        targetRevision: main
        path: 'apps/{{path[1]}}/overlays/{{env}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path[1]}}-{{env}}'
```

Esto genera N servicios x 3 entornos = 3N Applications automaticamente.

---

## 12. Buenas practicas

| Practica | Por que |
|----------|---------|
| Separar repo de codigo y repo de manifiestos | Un push al codigo no deberia desplegar automaticamente; el CI actualiza la imagen en el repo de manifiestos |
| Usar tags de imagen inmutables | `v1.2.3` en vez de `latest`; reproducibilidad |
| Activar selfHeal en staging, no en produccion | En prod, un cambio manual puede ser un hotfix urgente |
| Usar sync waves para dependencias | BD antes que app, configmaps antes que deployments |
| Cifrar secretos con Sealed Secrets o SOPS | Nunca secrets en texto plano en Git |
| ApplicationSets para escalar | No crear Applications manualmente para cada servicio/entorno |
| Webhooks de GitHub a ArgoCD | Sincronizacion inmediata en lugar de polling cada 3 minutos |
| RBAC con Projects de ArgoCD | Limitar que equipos pueden desplegar donde |
| Notificaciones de sync | ArgoCD Notifications para Slack/Teams cuando hay sync o fallo |
| Health checks en los deployments | readinessProbe y livenessProbe para que ArgoCD sepa si la app esta sana |

---

## 13. Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| **GitOps** | Git como fuente de verdad para despliegues | Manifiestos en Git, agente sincroniza al cluster |
| **ArgoCD** | Controlador GitOps para Kubernetes | Instalar en cluster, crear Applications |
| **Application CRD** | Recurso que define que/donde/como desplegar | Source (repo Git) + Destination (cluster) + SyncPolicy |
| **Auto-sync** | Sincronizacion automatica cuando Git cambia | `syncPolicy.automated` con selfHeal y prune |
| **Sync waves** | Orden de despliegue de recursos | Anotacion `argocd.argoproj.io/sync-wave` |
| **Hooks** | Jobs que corren en momentos del sync | PreSync (migraciones), PostSync (smoke tests) |
| **Rollback** | Volver a una version anterior | `git revert` (recomendado) o `argocd app rollback` |
| **Multi-env** | Gestionar dev/staging/prod | Kustomize overlays o Helm values por entorno |
| **Sealed Secrets** | Secretos cifrados para Git | `kubeseal` cifra, controlador descifra en cluster |
| **ApplicationSets** | Generar Applications automaticamente | Generators (list, git, matrix) + template |

---

## 14. Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | [Setup de ArgoCD](ejercicio-01-argocd-setup/) | Instalar ArgoCD en un cluster Kind local, acceder a la UI, configurar la CLI y registrar un repositorio Git | install, port-forward, argocd login, repo add |
| 02 | [Deploy de aplicacion](ejercicio-02-app-deploy/) | Crear una Application de ArgoCD que despliegue una app Spring Boot desde un repo Git con sync automatico y health checks | Application CRD, auto-sync, selfHeal, probes |
| 03 | [Multi-entorno](ejercicio-03-multi-env/) | Configurar tres entornos (dev/staging/prod) usando Kustomize overlays y ApplicationSets, con Sealed Secrets para las credenciales | Kustomize, overlays, ApplicationSet, SealedSecrets |
| 04 | [Rollback](ejercicio-04-rollback/) | Simular un despliegue fallido, hacer rollback via Git revert, verificar que ArgoCD restaura el estado anterior y configurar notificaciones de sync | git revert, sync waves, hooks PreSync, notifications |

Haz los ejercicios en orden. Cada uno construye sobre los conceptos del anterior.

---

## 15. Recursos

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/en/stable/)
- [ArgoCD Getting Started](https://argo-cd.readthedocs.io/en/stable/getting_started/)
- [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets)
- [Kustomize Documentation](https://kustomize.io/)
- [GitOps Principles (OpenGitOps)](https://opengitops.dev/)
- [Weaveworks GitOps Guide](https://www.weave.works/technologies/gitops/)
- [ArgoCD Best Practices](https://argo-cd.readthedocs.io/en/stable/operator-manual/best_practices/)
