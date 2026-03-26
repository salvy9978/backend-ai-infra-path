# Nivel 03 — Kubernetes Avanzado

> Ya sabes desplegar Pods, Deployments y Services. Ahora toca lo que separa un cluster
> de tutorial de uno de produccion: configuracion externalizada con ConfigMaps y Secrets,
> limites de recursos, autoescalado, probes de salud, almacenamiento persistente, Ingress
> para exponer servicios con dominio y TLS, y RBAC para controlar quien puede hacer que
> en el cluster. Sin esto, tu cluster no esta listo para cargas reales.

---

## Contenido

1. [ConfigMaps y Secrets](#1-configmaps-y-secrets)
2. [Resource limits y requests](#2-resource-limits-y-requests)
3. [HPA: Horizontal Pod Autoscaler](#3-hpa-horizontal-pod-autoscaler)
4. [Probes: liveness, readiness y startup](#4-probes-liveness-readiness-y-startup)
5. [PersistentVolumes y PersistentVolumeClaims](#5-persistentvolumes-y-persistentvolumeclaims)
6. [Ingress: exponer servicios al exterior](#6-ingress-exponer-servicios-al-exterior)
7. [RBAC: control de acceso basado en roles](#7-rbac-control-de-acceso-basado-en-roles)
8. [Rolling updates y rollbacks](#8-rolling-updates-y-rollbacks)
9. [Resumen](#resumen)
10. [Ejercicios](#ejercicios)
11. [Recursos](#recursos)

---

## 1. ConfigMaps y Secrets

### 1.1 El problema: configuracion hardcodeada

En el nivel anterior, las variables de entorno estaban directamente en el manifiesto
del Deployment. Esto tiene varios problemas:

- Si cambias una variable, tienes que editar el Deployment y redesplegar
- Las credenciales quedan en texto plano en el YAML (y posiblemente en Git)
- La misma app en dev y prod necesita manifiestos diferentes

ConfigMaps y Secrets externalizan la configuracion del Deployment.

### 1.2 ConfigMaps

Un ConfigMap almacena datos de configuracion no sensibles como pares clave-valor
o archivos completos.

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mi-api-config
  namespace: mi-app
data:
  # Pares clave-valor simples
  SPRING_PROFILES_ACTIVE: "prod"
  SERVER_PORT: "8080"
  LOGGING_LEVEL_ROOT: "INFO"
  LOGGING_LEVEL_COM_MIAPP: "DEBUG"

  # Archivos completos como valores
  application-prod.yml: |
    spring:
      datasource:
        url: jdbc:postgresql://postgres:5432/midb
        hikari:
          maximum-pool-size: 10
      jpa:
        hibernate:
          ddl-auto: validate
    server:
      port: 8080
    management:
      endpoints:
        web:
          exposure:
            include: health,info,prometheus
```

Crear ConfigMaps con kubectl:

```bash
# Desde literales
kubectl create configmap mi-api-config \
    --from-literal=SPRING_PROFILES_ACTIVE=prod \
    --from-literal=SERVER_PORT=8080 \
    -n mi-app

# Desde un archivo
kubectl create configmap mi-api-config \
    --from-file=application-prod.yml \
    -n mi-app

# Desde un directorio (cada archivo se convierte en una entrada)
kubectl create configmap mi-api-config \
    --from-file=config/ \
    -n mi-app

# Ver el contenido
kubectl describe configmap mi-api-config -n mi-app
kubectl get configmap mi-api-config -n mi-app -o yaml
```

### 1.3 Usar ConfigMaps en Pods

Hay dos formas de inyectar un ConfigMap en un Pod:

```yaml
# Forma 1: como variables de entorno
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
          image: mi-api:1.0.0
          # Inyectar variables individuales
          env:
            - name: SPRING_PROFILES_ACTIVE
              valueFrom:
                configMapKeyRef:
                  name: mi-api-config            # nombre del ConfigMap
                  key: SPRING_PROFILES_ACTIVE    # clave especifica
          # Inyectar TODAS las claves como variables de entorno
          envFrom:
            - configMapRef:
                name: mi-api-config              # todas las claves -> variables de entorno
```

```yaml
# Forma 2: como archivos montados en un volumen
spec:
  containers:
    - name: api
      image: mi-api:1.0.0
      volumeMounts:
        - name: config-volume
          mountPath: /app/config                 # directorio donde se montan los archivos
          readOnly: true
  volumes:
    - name: config-volume
      configMap:
        name: mi-api-config                      # nombre del ConfigMap
        items:                                    # (opcional) seleccionar claves especificas
          - key: application-prod.yml
            path: application-prod.yml           # nombre del archivo dentro del volumen
```

### 1.4 Secrets

Los Secrets son similares a ConfigMaps pero para datos sensibles: contrasenas, tokens,
certificados TLS, claves SSH. Los valores se almacenan codificados en base64 (no es
cifrado, es solo codificacion).

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: mi-api-secrets
  namespace: mi-app
type: Opaque                                     # tipo generico
data:
  # Valores en base64
  DB_PASSWORD: c2VjcmV0by1zZWd1cm8tMTIz          # echo -n "secreto-seguro-123" | base64
  REDIS_PASSWORD: cmVkaXMtc2VjcmV0by00NTY=        # echo -n "redis-secreto-456" | base64
---
# Alternativa: stringData (texto plano, K8s lo codifica automaticamente)
apiVersion: v1
kind: Secret
metadata:
  name: mi-api-secrets-v2
  namespace: mi-app
type: Opaque
stringData:
  DB_PASSWORD: secreto-seguro-123                # K8s lo convierte a base64
  REDIS_PASSWORD: redis-secreto-456
```

Crear Secrets con kubectl:

```bash
# Desde literales
kubectl create secret generic mi-api-secrets \
    --from-literal=DB_PASSWORD=secreto-seguro-123 \
    --from-literal=REDIS_PASSWORD=redis-secreto-456 \
    -n mi-app

# Secret TLS (para certificados)
kubectl create secret tls mi-tls-secret \
    --cert=tls.crt \
    --key=tls.key \
    -n mi-app

# Secret para Docker registry
kubectl create secret docker-registry mi-registry \
    --docker-server=ghcr.io \
    --docker-username=mi-usuario \
    --docker-password=mi-token \
    -n mi-app
```

### 1.5 Usar Secrets en Pods

```yaml
spec:
  containers:
    - name: api
      image: mi-api:1.0.0
      env:
        - name: SPRING_DATASOURCE_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mi-api-secrets               # nombre del Secret
              key: DB_PASSWORD                   # clave especifica
      envFrom:
        - secretRef:
            name: mi-api-secrets                 # todas las claves como env vars
  # Para imagenes de registries privados
  imagePullSecrets:
    - name: mi-registry
```

### 1.6 Buenas practicas con Secrets

```
[ ] Nunca commitear Secrets en Git (usar Sealed Secrets o External Secrets)
[ ] Habilitar cifrado en reposo de etcd (encryption at rest)
[ ] Usar RBAC para restringir acceso a Secrets
[ ] Rotar Secrets periodicamente
[ ] Considerar soluciones externas: HashiCorp Vault, AWS Secrets Manager
[ ] Usar stringData en los manifiestos (mas legible, K8s codifica automaticamente)
```

---

## 2. Resource limits y requests

### 2.1 Requests vs Limits

Kubernetes permite definir cuantos recursos (CPU, memoria) necesita cada contenedor:

| Concepto | Significado | Efecto |
|----------|-------------|--------|
| `requests` | Recursos garantizados | El Scheduler usa esto para decidir en que nodo colocar el Pod |
| `limits` | Recursos maximos | Si se supera: CPU se throttlea, memoria causa OOMKill |

```yaml
spec:
  containers:
    - name: api
      image: mi-api:1.0.0
      resources:
        requests:
          memory: "256Mi"                        # garantizados 256 MB
          cpu: "250m"                            # garantizados 0.25 CPU
        limits:
          memory: "512Mi"                        # maximo 512 MB (OOMKill si se supera)
          cpu: "1000m"                           # maximo 1 CPU (throttle si se supera)
```

### 2.2 Unidades de CPU y memoria

```
CPU:
  1 CPU = 1000m (milicores)
  "250m" = 0.25 CPU = 25% de un nucleo
  "1"    = 1 CPU completa
  "1500m" = 1.5 CPUs

Memoria:
  "128Mi" = 128 MiB (mebibytes, base 2: 128 * 1024 * 1024 bytes)
  "256M"  = 256 MB  (megabytes, base 10: 256 * 1000 * 1000 bytes)
  "1Gi"   = 1 GiB   (gibibyte)

  Usa Mi/Gi (base 2) para consistencia con el sistema operativo.
```

### 2.3 QoS Classes

Kubernetes asigna una clase de calidad de servicio (QoS) a cada Pod segun sus
resources. Esto afecta a que Pods se matan primero cuando el nodo se queda sin memoria:

| QoS Class | Condicion | Prioridad de eviccion |
|-----------|-----------|----------------------|
| `Guaranteed` | requests == limits (para todos los contenedores) | Ultima en ser eviccionada |
| `Burstable` | requests < limits (al menos un contenedor) | Eviccionada si supera requests |
| `BestEffort` | Sin requests ni limits | Primera en ser eviccionada |

```yaml
# Guaranteed (maxima proteccion)
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "512Mi"                              # igual que requests
    cpu: "500m"                                  # igual que requests

# Burstable (la mas comun)
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"                              # mayor que requests
    cpu: "1000m"

# BestEffort (evitar en produccion)
# No definir resources
```

### 2.4 LimitRange: defaults por namespace

Un LimitRange define valores por defecto y limites para un namespace:

```yaml
# limitrange.yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: mi-app
spec:
  limits:
    - type: Container
      default:                                   # limits por defecto
        cpu: "500m"
        memory: "256Mi"
      defaultRequest:                            # requests por defecto
        cpu: "100m"
        memory: "128Mi"
      max:                                       # maximo permitido
        cpu: "2"
        memory: "2Gi"
      min:                                       # minimo permitido
        cpu: "50m"
        memory: "64Mi"
```

### 2.5 ResourceQuota: limites por namespace

ResourceQuota limita el total de recursos que un namespace puede consumir:

```yaml
# resourcequota.yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: mi-app-quota
  namespace: mi-app
spec:
  hard:
    requests.cpu: "4"                            # total de CPU requests en el namespace
    requests.memory: "8Gi"
    limits.cpu: "8"
    limits.memory: "16Gi"
    pods: "20"                                   # maximo 20 Pods
    services: "10"
    persistentvolumeclaims: "5"
```

---

## 3. HPA: Horizontal Pod Autoscaler

### 3.1 Que es el HPA

El HPA ajusta automaticamente el numero de replicas de un Deployment basandose en
metricas observadas (CPU, memoria, metricas custom).

```
                     +-----+
   Metricas -------->| HPA |-------> Escalar Deployment
   (CPU, memoria)    +-----+         (mas o menos replicas)

   CPU al 80%  ->  HPA detecta  ->  Escala de 3 a 5 replicas
   CPU al 20%  ->  HPA detecta  ->  Escala de 5 a 2 replicas
```

### 3.2 Prerequisito: Metrics Server

El HPA necesita el Metrics Server para obtener metricas de CPU y memoria:

```bash
# En Minikube, habilitarlo como addon
minikube addons enable metrics-server

# En un cluster real, instalar con kubectl
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verificar que funciona
kubectl top nodes
kubectl top pods -n mi-app
```

### 3.3 Crear un HPA

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: mi-api-hpa
  namespace: mi-app
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: mi-api                                 # Deployment a escalar
  minReplicas: 2                                 # minimo 2 replicas
  maxReplicas: 10                                # maximo 10 replicas
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70                 # escalar si CPU > 70%
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80                 # escalar si memoria > 80%
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300            # esperar 5 min antes de reducir
      policies:
        - type: Pods
          value: 1                               # reducir de 1 en 1
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0              # escalar inmediatamente
      policies:
        - type: Pods
          value: 4                               # escalar hasta 4 Pods a la vez
          periodSeconds: 60
```

```bash
# Crear HPA con kubectl (forma rapida)
kubectl autoscale deployment mi-api \
    --min=2 --max=10 \
    --cpu-percent=70 \
    -n mi-app

# Ver estado del HPA
kubectl get hpa -n mi-app
# NAME         REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS
# mi-api-hpa   Deployment/mi-api  35%/70%   2         10        3

# Ver detalle
kubectl describe hpa mi-api-hpa -n mi-app
```

### 3.4 Probar el autoescalado

```bash
# Generar carga con un Pod de prueba
kubectl run load-generator \
    --image=busybox \
    --restart=Never \
    -n mi-app \
    -- /bin/sh -c "while true; do wget -q -O- http://mi-api/actuator/health; done"

# Observar el HPA en tiempo real
kubectl get hpa -n mi-app --watch
# NAME         TARGETS    MINPODS   MAXPODS   REPLICAS
# mi-api-hpa   35%/70%    2         10        2
# mi-api-hpa   72%/70%    2         10        2          <- carga detectada
# mi-api-hpa   72%/70%    2         10        4          <- escalado
# mi-api-hpa   45%/70%    2         10        4          <- carga distribuida

# Detener la carga
kubectl delete pod load-generator -n mi-app

# Despues de 5 min (stabilizationWindowSeconds), reducira replicas
```

---

## 4. Probes: liveness, readiness y startup

### 4.1 Por que necesitas probes

Sin probes, Kubernetes no sabe si tu app esta realmente funcionando. Un contenedor puede
estar corriendo (proceso activo) pero la app dentro puede estar en deadlock, sin conexion
a la BD, o aun arrancando.

| Probe | Pregunta que responde | Si falla |
|-------|-----------------------|----------|
| **Liveness** | "Esta vivo el proceso?" | K8s reinicia el contenedor |
| **Readiness** | "Puede recibir trafico?" | K8s lo quita del Service (no recibe requests) |
| **Startup** | "Ha terminado de arrancar?" | Desactiva liveness/readiness hasta que pase |

### 4.2 Tipos de verificacion

```yaml
# HTTP GET: la mas comun para APIs
livenessProbe:
  httpGet:
    path: /actuator/health/liveness              # endpoint de Spring Boot Actuator
    port: 8080
  initialDelaySeconds: 30                        # esperar 30s antes del primer check
  periodSeconds: 10                              # verificar cada 10s
  timeoutSeconds: 5                              # maximo 5s por verificacion
  failureThreshold: 3                            # 3 fallos = reiniciar

# TCP Socket: verificar que el puerto esta abierto
livenessProbe:
  tcpSocket:
    port: 5432                                   # verificar que PostgreSQL escucha
  periodSeconds: 10

# Exec: ejecutar un comando dentro del contenedor
livenessProbe:
  exec:
    command:
      - /bin/sh
      - -c
      - pg_isready -U miapp -d midb              # verificar PostgreSQL
  periodSeconds: 10
```

### 4.3 Configuracion completa para Spring Boot

```yaml
spec:
  containers:
    - name: api
      image: mi-api:1.0.0
      ports:
        - containerPort: 8080

      # Startup probe: desactiva liveness/readiness hasta que la app arranque
      # Spring Boot puede tardar 30-60s en arrancar
      startupProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        initialDelaySeconds: 10
        periodSeconds: 5
        failureThreshold: 30                     # 30 * 5s = 150s maximo para arrancar

      # Liveness probe: esta vivo el proceso?
      livenessProbe:
        httpGet:
          path: /actuator/health/liveness
          port: 8080
        periodSeconds: 10
        timeoutSeconds: 5
        failureThreshold: 3                      # 3 fallos = reiniciar

      # Readiness probe: puede recibir trafico?
      readinessProbe:
        httpGet:
          path: /actuator/health/readiness
          port: 8080
        periodSeconds: 5
        timeoutSeconds: 3
        failureThreshold: 3                      # 3 fallos = quitar del Service
```

Configurar Spring Boot Actuator para probes de K8s:

```yaml
# application.yml de Spring Boot
management:
  endpoint:
    health:
      probes:
        enabled: true                            # habilitar /liveness y /readiness
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState,db             # incluir check de base de datos
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
```

### 4.4 Escenarios comunes

| Escenario | Que pasa sin probes | Que pasa con probes |
|-----------|---------------------|---------------------|
| App en deadlock | Sigue recibiendo trafico, errores 500 | Liveness falla, K8s reinicia el Pod |
| BD caida | App acepta requests pero falla | Readiness falla, trafico va a otras replicas |
| Arranque lento | K8s cree que fallo, la mata | Startup probe espera el tiempo necesario |
| Memory leak | Se queda sin memoria, OOMKill | Liveness detecta antes del OOMKill |

---

## 5. PersistentVolumes y PersistentVolumeClaims

### 5.1 El problema del almacenamiento efimero

Por defecto, los datos dentro de un Pod se pierden cuando el Pod muere. Esto es un
problema para bases de datos y cualquier servicio que necesite datos persistentes.

```
Pod muere -> Datos perdidos

Pod con PersistentVolume -> Pod muere -> Nuevo Pod se conecta al mismo volumen -> Datos intactos
```

### 5.2 Conceptos: PV, PVC, StorageClass

```
+-------------------------------------------------------------------+
|  StorageClass:  Define COMO se provisiona el almacenamiento        |
|  (SSD, HDD, NFS, EBS, etc.)                                       |
|                                                                     |
|  PersistentVolume (PV):  Un trozo de almacenamiento en el cluster  |
|  (puede ser un disco EBS, un NFS share, etc.)                      |
|                                                                     |
|  PersistentVolumeClaim (PVC):  Una solicitud de almacenamiento     |
|  de un Pod. K8s busca un PV que cumpla los requisitos.             |
|                                                                     |
|  Pod -> PVC -> PV -> Almacenamiento fisico                         |
+-------------------------------------------------------------------+
```

### 5.3 StorageClass

```yaml
# storageclass.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs               # proveedor de almacenamiento
parameters:
  type: gp3                                      # tipo de disco EBS
  fsType: ext4
reclaimPolicy: Retain                            # que hacer cuando se borra el PVC
allowVolumeExpansion: true                       # permitir expandir el volumen
volumeBindingMode: WaitForFirstConsumer          # provisionar cuando un Pod lo necesite
```

| reclaimPolicy | Efecto |
|--------------|--------|
| `Retain` | El PV se conserva despues de borrar el PVC (manual cleanup) |
| `Delete` | El PV y el disco se borran automaticamente |
| `Recycle` | Deprecado. No usar |

### 5.4 PersistentVolumeClaim

```yaml
# pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-data
  namespace: mi-app
spec:
  accessModes:
    - ReadWriteOnce                              # un solo nodo puede montarlo
  storageClassName: fast-ssd                     # usar la StorageClass "fast-ssd"
  resources:
    requests:
      storage: 10Gi                              # 10 GB de almacenamiento
```

| accessModes | Significado |
|------------|-------------|
| `ReadWriteOnce` (RWO) | Un solo nodo puede leer/escribir |
| `ReadOnlyMany` (ROX) | Multiples nodos pueden leer |
| `ReadWriteMany` (RWX) | Multiples nodos pueden leer/escribir (NFS, EFS) |

### 5.5 Usar PVC en un Deployment

```yaml
# postgres-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: mi-app
spec:
  replicas: 1                                    # bases de datos: generalmente 1 replica
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          ports:
            - containerPort: 5432
          env:
            - name: POSTGRES_DB
              value: midb
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mi-api-secrets
                  key: DB_PASSWORD
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          volumeMounts:
            - name: postgres-storage
              mountPath: /var/lib/postgresql/data # donde PostgreSQL guarda datos
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1"
      volumes:
        - name: postgres-storage
          persistentVolumeClaim:
            claimName: postgres-data              # referencia al PVC
```

```bash
# Ver PVCs
kubectl get pvc -n mi-app
# NAME            STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
# postgres-data   Bound    pv-xxx     10Gi       RWO            fast-ssd

# Ver PVs
kubectl get pv
# NAME     CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
# pv-xxx   10Gi       RWO            Retain           Bound    mi-app/postgres-data
```

---

## 6. Ingress: exponer servicios al exterior

### 6.1 Por que Ingress en lugar de LoadBalancer

Un Service tipo LoadBalancer crea un LB por cada servicio. Con 10 microservicios,
tendrias 10 load balancers (costoso). Ingress consolida todo en un solo punto de entrada:

```
Sin Ingress:                          Con Ingress:
LB1 -> svc/api                        +----------+
LB2 -> svc/web                        | Ingress  | -> /api    -> svc/api
LB3 -> svc/admin                      | (1 solo  | -> /web    -> svc/web
(3 load balancers = $$$)               |  LB)     | -> /admin  -> svc/admin
                                       +----------+
```

### 6.2 Ingress Controller

El recurso Ingress solo define reglas. Necesitas un Ingress Controller que las ejecute.
Los mas populares:

| Controller | Proveedor | Notas |
|-----------|-----------|-------|
| NGINX Ingress Controller | Open source | El mas usado, muy configurable |
| Traefik | Open source | Auto-discovery, dashboard incluido |
| AWS ALB Ingress Controller | AWS | Crea ALBs nativos de AWS |
| GCE Ingress Controller | GCP | Incluido por defecto en GKE |

```bash
# Instalar NGINX Ingress Controller en Minikube
minikube addons enable ingress

# En un cluster real, instalar con Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx --create-namespace
```

### 6.3 Reglas de Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-app-ingress
  namespace: mi-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx                        # que Ingress Controller usar
  tls:
    - hosts:
        - api.midominio.com
      secretName: tls-secret                     # Secret con certificado TLS
  rules:
    - host: api.midominio.com                    # dominio
      http:
        paths:
          - path: /api                           # ruta
            pathType: Prefix
            backend:
              service:
                name: mi-api                     # Service destino
                port:
                  number: 80
          - path: /admin
            pathType: Prefix
            backend:
              service:
                name: admin-panel
                port:
                  number: 80
    - host: web.midominio.com                    # segundo dominio
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mi-web
                port:
                  number: 80
```

### 6.4 TLS con cert-manager

cert-manager automatiza la obtencion y renovacion de certificados TLS (Let's Encrypt):

```bash
# Instalar cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/latest/download/cert-manager.yaml
```

```yaml
# clusterissuer.yaml (configurar Let's Encrypt)
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@midominio.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

```yaml
# Ingress con TLS automatico
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mi-app-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod   # cert-manager genera el certificado
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.midominio.com
      secretName: api-tls-cert                   # cert-manager crea este Secret
  rules:
    - host: api.midominio.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: mi-api
                port:
                  number: 80
```

### 6.5 Annotations utiles de NGINX Ingress

```yaml
annotations:
  # Rate limiting
  nginx.ingress.kubernetes.io/limit-rps: "10"              # max 10 requests/segundo
  nginx.ingress.kubernetes.io/limit-connections: "5"        # max 5 conexiones simultaneas

  # Timeouts
  nginx.ingress.kubernetes.io/proxy-read-timeout: "60"      # timeout de lectura
  nginx.ingress.kubernetes.io/proxy-send-timeout: "60"      # timeout de envio

  # Body size
  nginx.ingress.kubernetes.io/proxy-body-size: "10m"        # max 10 MB de body

  # CORS
  nginx.ingress.kubernetes.io/enable-cors: "true"
  nginx.ingress.kubernetes.io/cors-allow-origin: "https://midominio.com"

  # Redirect HTTP -> HTTPS
  nginx.ingress.kubernetes.io/ssl-redirect: "true"
```

---

## 7. RBAC: control de acceso basado en roles

### 7.1 Que es RBAC

RBAC (Role-Based Access Control) controla quien puede hacer que en el cluster.
Define permisos como combinaciones de verbos y recursos.

```
Quien (Subject)  +  Que puede hacer (Role)  +  Donde (Namespace/Cluster)
     |                      |                         |
  User/Group/          Verbs + Resources         Role vs ClusterRole
  ServiceAccount       (get, list, create...)    (namespace vs cluster-wide)
```

### 7.2 Roles y ClusterRoles

```yaml
# Role: permisos dentro de un namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: mi-app
rules:
  - apiGroups: [""]                              # "" = core API group
    resources: ["pods", "pods/log"]
    verbs: ["get", "list", "watch"]              # solo lectura
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list"]
```

```yaml
# ClusterRole: permisos en todo el cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: namespace-admin
rules:
  - apiGroups: ["", "apps", "batch"]
    resources: ["*"]                             # todos los recursos
    verbs: ["*"]                                 # todas las operaciones
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list"]                       # pero secrets solo lectura
```

### 7.3 RoleBindings y ClusterRoleBindings

```yaml
# RoleBinding: asignar Role a un Subject dentro de un namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-pod-reader
  namespace: mi-app
subjects:
  - kind: User
    name: juan@empresa.com
    apiGroup: rbac.authorization.k8s.io
  - kind: ServiceAccount
    name: mi-api-sa
    namespace: mi-app
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

```yaml
# ClusterRoleBinding: asignar ClusterRole a nivel de cluster
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-binding
subjects:
  - kind: Group
    name: platform-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: namespace-admin
  apiGroup: rbac.authorization.k8s.io
```

### 7.4 ServiceAccounts para aplicaciones

```yaml
# serviceaccount.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: mi-api-sa
  namespace: mi-app
---
# Asignar al Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-api
spec:
  template:
    spec:
      serviceAccountName: mi-api-sa              # usar esta ServiceAccount
      automountServiceAccountToken: false         # no montar token si no lo necesita
      containers:
        - name: api
          image: mi-api:1.0.0
```

### 7.5 Verificar permisos

```bash
# Puedo hacer X?
kubectl auth can-i create deployments -n mi-app
# yes

kubectl auth can-i delete secrets -n mi-app
# no

# Como otro usuario
kubectl auth can-i create pods -n mi-app --as=juan@empresa.com
# no

# Listar todos los permisos de un ServiceAccount
kubectl auth can-i --list --as=system:serviceaccount:mi-app:mi-api-sa -n mi-app
```

---

## 8. Rolling updates y rollbacks

### 8.1 Estrategias de despliegue

```yaml
# RollingUpdate (default): reemplazar Pods gradualmente
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1                          # max 1 Pod fuera a la vez
      maxSurge: 1                                # max 1 Pod extra a la vez

# Recreate: matar todos los Pods y crear los nuevos
# Causa downtime. Usar solo si la app no soporta multiples versiones simultaneas.
spec:
  strategy:
    type: Recreate
```

Ejemplo visual de un rolling update con 3 replicas:

```
Estado inicial:  [v1] [v1] [v1]
Paso 1:          [v1] [v1] [v2]    <- crear 1 nuevo, matar 1 viejo
Paso 2:          [v1] [v2] [v2]    <- crear 1 nuevo, matar 1 viejo
Paso 3:          [v2] [v2] [v2]    <- completado, zero downtime
```

### 8.2 Ejecutar un rolling update

```bash
# Opcion 1: cambiar la imagen
kubectl set image deployment/mi-api api=mi-api:2.0.0 -n mi-app

# Opcion 2: editar el YAML y aplicar
# (cambiar image: mi-api:2.0.0 en deployment.yaml)
kubectl apply -f deployment.yaml

# Ver progreso
kubectl rollout status deployment/mi-api -n mi-app
# Waiting for deployment "mi-api" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "mi-api" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "mi-api" successfully rolled out

# Pausar un rollout (si ves problemas)
kubectl rollout pause deployment/mi-api -n mi-app

# Reanudar
kubectl rollout resume deployment/mi-api -n mi-app
```

### 8.3 Historial y rollbacks

```bash
# Ver historial de revisiones
kubectl rollout history deployment/mi-api -n mi-app
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
# 3         <none>

# Para que CHANGE-CAUSE tenga valor, anotar el Deployment
kubectl annotate deployment/mi-api \
    kubernetes.io/change-cause="Actualizar a v2.0.0" -n mi-app

# Ver detalle de una revision
kubectl rollout history deployment/mi-api --revision=2 -n mi-app

# Rollback a la revision anterior
kubectl rollout undo deployment/mi-api -n mi-app

# Rollback a una revision especifica
kubectl rollout undo deployment/mi-api --to-revision=1 -n mi-app

# Verificar el rollback
kubectl rollout status deployment/mi-api -n mi-app
kubectl get pods -n mi-app -l app=mi-api -o wide
```

### 8.4 Configurar el historial de revisiones

```yaml
spec:
  revisionHistoryLimit: 10                       # guardar 10 revisiones (default)
  # Reducir si quieres ahorrar recursos en etcd
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| ConfigMap | Configuracion no sensible (clave-valor o archivos) | `envFrom`, `volumeMounts`, `kubectl create configmap` |
| Secret | Datos sensibles codificados en base64 | `secretKeyRef`, `stringData`, nunca en Git |
| requests | Recursos garantizados para un contenedor | `resources.requests.cpu`, `resources.requests.memory` |
| limits | Recursos maximos para un contenedor | CPU se throttlea, memoria causa OOMKill |
| QoS Class | Prioridad de eviccion del Pod | Guaranteed > Burstable > BestEffort |
| LimitRange | Defaults y limites por namespace | Evita Pods sin resource limits |
| ResourceQuota | Cuota total de recursos por namespace | Limitar consumo por equipo/entorno |
| HPA | Autoescalado horizontal por metricas | `minReplicas`, `maxReplicas`, `averageUtilization` |
| Liveness Probe | Verifica si el proceso esta vivo | Si falla, K8s reinicia el contenedor |
| Readiness Probe | Verifica si puede recibir trafico | Si falla, se quita del Service |
| Startup Probe | Espera el arranque inicial | Desactiva liveness/readiness hasta que pase |
| PersistentVolume | Almacenamiento en el cluster | Provisionado por StorageClass o manualmente |
| PersistentVolumeClaim | Solicitud de almacenamiento | `accessModes`, `storageClassName`, `storage` |
| Ingress | Punto de entrada HTTP/HTTPS al cluster | Reglas por host y path, TLS con cert-manager |
| RBAC Role | Permisos dentro de un namespace | `rules` con `apiGroups`, `resources`, `verbs` |
| ClusterRole | Permisos en todo el cluster | Para recursos cluster-wide (nodes, namespaces) |
| RoleBinding | Asignar Role a usuario/grupo/SA | `subjects` + `roleRef` |
| Rolling Update | Despliegue gradual sin downtime | `maxUnavailable`, `maxSurge` |
| Rollback | Volver a una version anterior | `kubectl rollout undo` |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **ConfigMaps y Secrets** | Externalizar la configuracion de una app Spring Boot con ConfigMaps (application.yml) y Secrets (credenciales de BD). Verificar que la app arranca con la configuracion inyectada | ConfigMap, Secret, envFrom, volumeMounts, stringData |
| 02 | **HPA y Scaling** | Configurar HPA para una app, generar carga con un Pod busybox, observar el escalado automatico. Verificar que reduce replicas cuando baja la carga | HPA, Metrics Server, `kubectl top`, `kubectl get hpa --watch` |
| 03 | **Probes y Lifecycle** | Configurar las tres probes para una app Spring Boot con Actuator. Simular un fallo (endpoint que causa deadlock) y verificar que K8s reinicia el Pod | Liveness, Readiness, Startup, Actuator health groups |
| 04 | **Ingress con TLS** | Instalar NGINX Ingress Controller, crear reglas de Ingress con multiples paths, configurar TLS con un certificado auto-firmado. Verificar acceso HTTPS | Ingress, IngressClass, TLS, annotations, cert-manager |

---

## Recursos

- [Kubernetes ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/)
- [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
- [Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Horizontal Pod Autoscaler](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/)
- [Configure Liveness, Readiness and Startup Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [cert-manager Documentation](https://cert-manager.io/docs/)

---

> **Antes de continuar al siguiente nivel**: asegurate de que tu app Spring Boot tiene
> probes configuradas y usa ConfigMaps/Secrets para la configuracion. Estos son requisitos
> minimos para cualquier despliegue en produccion. Sin probes, K8s no puede auto-recuperar
> tu app. Sin ConfigMaps/Secrets, no puedes cambiar configuracion sin redesplegar.
