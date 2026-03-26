# Nivel 02 — Kubernetes: Fundamentos

> Docker resuelve el problema de empaquetar una aplicacion. Kubernetes resuelve el
> problema de ejecutar cientos de contenedores en produccion: orquestacion, escalado,
> auto-recuperacion, service discovery. Este nivel cubre la arquitectura de K8s,
> los objetos fundamentales (Pods, Deployments, Services) y como desplegar tu primera
> app Spring Boot en un cluster local con Minikube.

---

## Contenido

1. [Por que Kubernetes](#1-por-que-kubernetes)
2. [Arquitectura de Kubernetes](#2-arquitectura-de-kubernetes)
3. [Pods: la unidad minima](#3-pods-la-unidad-minima)
4. [ReplicaSets y Deployments](#4-replicasets-y-deployments)
5. [Services: exponer aplicaciones](#5-services-exponer-aplicaciones)
6. [Namespaces: aislamiento logico](#6-namespaces-aislamiento-logico)
7. [kubectl: la herramienta esencial](#7-kubectl-la-herramienta-esencial)
8. [Estructura de manifiestos YAML](#8-estructura-de-manifiestos-yaml)
9. [Entorno local: Minikube y Kind](#9-entorno-local-minikube-y-kind)
10. [Desplegando una app Spring Boot](#10-desplegando-una-app-spring-boot)
11. [Resumen](#resumen)
12. [Ejercicios](#ejercicios)
13. [Recursos](#recursos)

---

## 1. Por que Kubernetes

### 1.1 El problema que resuelve

Con Docker puedes ejecutar contenedores en una maquina. Pero en produccion necesitas:

- Ejecutar multiples replicas de tu app (alta disponibilidad)
- Reiniciar automaticamente contenedores que fallan
- Distribuir trafico entre replicas (load balancing)
- Escalar bajo demanda (mas replicas cuando hay carga)
- Desplegar nuevas versiones sin downtime (rolling updates)
- Gestionar configuracion y secrets de forma centralizada

Docker solo no resuelve nada de esto. Kubernetes si.

### 1.2 Que es Kubernetes

Kubernetes (K8s) es un orquestador de contenedores. Le describes el estado deseado
("quiero 3 replicas de mi-api con 512 MB de RAM") y K8s se encarga de que ese estado
se mantenga continuamente.

```
Tu declaras:                          K8s garantiza:
"Quiero 3 replicas de mi-api"   ->   Si una replica muere, crea otra
"Quiero max 80% CPU"            ->   Si se supera, escala horizontalmente
"Nueva version: v2"             ->   Rolling update sin downtime
"Necesito una base de datos"    ->   Provisiona almacenamiento persistente
```

---

## 2. Arquitectura de Kubernetes

### 2.1 Vista general

Un cluster de Kubernetes tiene dos tipos de nodos: el plano de control (control plane)
y los nodos de trabajo (worker nodes).

```
+------------------------------------------------------------------+
|  CLUSTER DE KUBERNETES                                            |
|                                                                    |
|  CONTROL PLANE (Master)                                           |
|  +------------------------------------------------------------+  |
|  |                                                              |  |
|  |  +----------------+  +----------+  +--------------------+   |  |
|  |  | API Server     |  | etcd     |  | Scheduler          |   |  |
|  |  | (puerta de     |  | (base de |  | (decide en que     |   |  |
|  |  |  entrada)      |  |  datos)  |  |  nodo va cada Pod) |   |  |
|  |  +----------------+  +----------+  +--------------------+   |  |
|  |                                                              |  |
|  |  +--------------------+  +----------------------------+     |  |
|  |  | Controller Manager |  | Cloud Controller Manager   |     |  |
|  |  | (reconcilia estado |  | (integra con AWS/GCP)      |     |  |
|  |  |  deseado vs real)  |  |                            |     |  |
|  |  +--------------------+  +----------------------------+     |  |
|  +------------------------------------------------------------+  |
|                                                                    |
|  WORKER NODES                                                     |
|  +------------------------+  +------------------------+           |
|  | Node 1                 |  | Node 2                 |           |
|  |                        |  |                        |           |
|  | +--------+ +--------+ |  | +--------+ +--------+  |           |
|  | | Pod A  | | Pod B  | |  | | Pod C  | | Pod D  |  |           |
|  | | (app)  | | (app)  | |  | | (app)  | | (db)   |  |           |
|  | +--------+ +--------+ |  | +--------+ +--------+  |           |
|  |                        |  |                        |           |
|  | kubelet   kube-proxy   |  | kubelet   kube-proxy   |           |
|  +------------------------+  +------------------------+           |
+------------------------------------------------------------------+
```

### 2.2 Componentes del Control Plane

| Componente | Funcion |
|-----------|---------|
| **API Server** | Punto de entrada para toda interaccion con K8s. kubectl habla con el API Server |
| **etcd** | Base de datos clave-valor que almacena todo el estado del cluster |
| **Scheduler** | Decide en que nodo se ejecuta cada nuevo Pod, basandose en recursos disponibles |
| **Controller Manager** | Ejecuta controladores que reconcilian el estado deseado con el real |
| **Cloud Controller Manager** | Integra con APIs del proveedor cloud (load balancers, discos, etc.) |

### 2.3 Componentes de los Worker Nodes

| Componente | Funcion |
|-----------|---------|
| **kubelet** | Agente que corre en cada nodo. Recibe instrucciones del API Server y gestiona los Pods |
| **kube-proxy** | Gestiona las reglas de red. Permite que los Services funcionen (routing de trafico) |
| **Container Runtime** | Ejecuta los contenedores (containerd, CRI-O). Ya no es Docker directamente |

### 2.4 Flujo de un despliegue

Cuando ejecutas `kubectl apply -f deployment.yaml`, esto es lo que pasa:

```
1. kubectl envia el YAML al API Server
2. API Server valida y almacena en etcd
3. Controller Manager detecta que faltan replicas
4. Scheduler asigna nodos a los Pods pendientes
5. kubelet en cada nodo descarga la imagen y arranca el contenedor
6. kube-proxy configura las reglas de red

kubectl -> API Server -> etcd (almacenar)
                      -> Controller Manager (crear Pods)
                      -> Scheduler (asignar nodos)
                      -> kubelet (ejecutar contenedores)
```

---

## 3. Pods: la unidad minima

### 3.1 Que es un Pod

Un Pod es la unidad minima de ejecucion en Kubernetes. Contiene uno o mas contenedores
que comparten red y almacenamiento. En la practica, la mayoria de Pods tienen un solo
contenedor.

```yaml
# pod-basico.yaml
apiVersion: v1                                   # version del API
kind: Pod                                        # tipo de recurso
metadata:
  name: mi-api                                   # nombre del Pod
  labels:
    app: mi-api                                  # etiqueta para seleccion
    env: dev
spec:
  containers:
    - name: api                                  # nombre del contenedor
      image: mi-api:1.0.0                        # imagen Docker
      ports:
        - containerPort: 8080                    # puerto que expone
      resources:
        requests:                                # recursos garantizados
          memory: "256Mi"
          cpu: "250m"                            # 0.25 CPU
        limits:                                  # recursos maximos
          memory: "512Mi"
          cpu: "500m"                            # 0.5 CPU
```

### 3.2 Ciclo de vida de un Pod

```
Pending -> Running -> Succeeded/Failed

Pending:    El Pod fue aceptado pero los contenedores aun no estan corriendo.
            Puede estar descargando la imagen o esperando recursos.
Running:    Al menos un contenedor esta corriendo.
Succeeded:  Todos los contenedores terminaron con exit code 0 (Jobs).
Failed:     Al menos un contenedor termino con error.
Unknown:    No se puede determinar el estado (problemas de comunicacion con el nodo).
```

### 3.3 Por que no crear Pods directamente

Nunca crees Pods aislados en produccion. Si un Pod muere, nadie lo reemplaza.
Usa Deployments (que crean ReplicaSets, que crean Pods):

```
Deployment -> ReplicaSet -> Pod(s)

Si un Pod muere, el ReplicaSet crea uno nuevo automaticamente.
Si quieres 5 replicas, el ReplicaSet se asegura de que haya 5.
```

---

## 4. ReplicaSets y Deployments

### 4.1 ReplicaSet

Un ReplicaSet garantiza que un numero especifico de replicas de un Pod este corriendo
en todo momento. Raramente lo creas directamente; lo gestiona el Deployment.

```yaml
# replicaset.yaml (normalmente no se crea manualmente)
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: mi-api-rs
spec:
  replicas: 3                                    # mantener 3 Pods
  selector:
    matchLabels:
      app: mi-api                                # seleccionar Pods con esta etiqueta
  template:
    metadata:
      labels:
        app: mi-api
    spec:
      containers:
        - name: api
          image: mi-api:1.0.0
          ports:
            - containerPort: 8080
```

### 4.2 Deployment

El Deployment es el recurso que usaras el 90% del tiempo. Gestiona ReplicaSets y
permite rolling updates, rollbacks y escalado declarativo.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-api
  labels:
    app: mi-api
spec:
  replicas: 3                                    # 3 instancias de la app
  selector:
    matchLabels:
      app: mi-api                                # los Pods deben tener esta etiqueta
  strategy:
    type: RollingUpdate                          # actualizar sin downtime
    rollingUpdate:
      maxUnavailable: 1                          # max 1 Pod fuera durante update
      maxSurge: 1                                # max 1 Pod extra durante update
  template:                                      # plantilla de Pod
    metadata:
      labels:
        app: mi-api
        version: "1.0.0"
    spec:
      containers:
        - name: api
          image: mi-api:1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: JAVA_OPTS
              value: "-Xmx384m -Xms256m"
```

### 4.3 Operaciones con Deployments

```bash
# Aplicar el manifiesto
kubectl apply -f deployment.yaml

# Ver deployments
kubectl get deployments
# NAME     READY   UP-TO-DATE   AVAILABLE   AGE
# mi-api   3/3     3            3           2m

# Ver los Pods creados
kubectl get pods -l app=mi-api
# NAME                      READY   STATUS    RESTARTS   AGE
# mi-api-7b9c8d6f5-abc12   1/1     Running   0          2m
# mi-api-7b9c8d6f5-def34   1/1     Running   0          2m
# mi-api-7b9c8d6f5-ghi56   1/1     Running   0          2m

# Escalar a 5 replicas
kubectl scale deployment mi-api --replicas=5

# Actualizar la imagen (rolling update)
kubectl set image deployment/mi-api api=mi-api:2.0.0

# Ver el progreso del rollout
kubectl rollout status deployment/mi-api

# Rollback a la version anterior
kubectl rollout undo deployment/mi-api

# Ver historial de rollouts
kubectl rollout history deployment/mi-api
```

---

## 5. Services: exponer aplicaciones

### 5.1 Por que necesitas Services

Los Pods son efimeros: nacen, mueren y su IP cambia. No puedes apuntar directamente a
la IP de un Pod. Un Service proporciona una IP estable y un nombre DNS que enruta
trafico a los Pods correctos.

```
Sin Service:                          Con Service:
App -> Pod IP (10.0.0.5)              App -> Service DNS (mi-api.default.svc)
       Pod muere...                          Service enruta a Pods sanos
App -> ?? (10.0.0.5 ya no existe)            Si un Pod muere, se quita del pool
```

### 5.2 Tipos de Service

```
+-------------------------------------------------------------------+
|                                                                     |
|  ClusterIP (default):  Solo accesible dentro del cluster            |
|  +----------+                                                       |
|  | Service  | ---> Pod A                                            |
|  | 10.96.x  | ---> Pod B   (load balancing automatico)              |
|  +----------+ ---> Pod C                                            |
|  Solo trafico interno del cluster                                   |
|                                                                     |
|  NodePort:  Abre un puerto en cada nodo del cluster                 |
|  +----------+                                                       |
|  | Service  | Puerto 30000-32767 en cada nodo                       |
|  | NodePort | ---> Pod A                                            |
|  +----------+ ---> Pod B                                            |
|  Accesible desde fuera via <NodeIP>:<NodePort>                      |
|                                                                     |
|  LoadBalancer:  Crea un LB del proveedor cloud                      |
|  +----------+    +-------------+                                    |
|  | Service  | -> | Cloud LB    | -> Pod A                           |
|  | LB       |    | (AWS ALB,   |    Pod B                           |
|  +----------+    | GCP LB)     |    Pod C                           |
|                  +-------------+                                    |
|  Accesible desde Internet via IP publica del LB                     |
+-------------------------------------------------------------------+
```

| Tipo | Accesible desde | Uso tipico |
|------|----------------|------------|
| `ClusterIP` | Solo dentro del cluster | Comunicacion entre microservicios |
| `NodePort` | IP del nodo + puerto (30000-32767) | Desarrollo local, testing |
| `LoadBalancer` | Internet (IP publica) | Produccion en cloud |
| `ExternalName` | DNS externo | Alias para servicios externos |

### 5.3 Manifiestos de Service

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-api                                   # nombre DNS: mi-api.default.svc.cluster.local
spec:
  type: ClusterIP                                # default
  selector:
    app: mi-api                                  # enruta a Pods con label app=mi-api
  ports:
    - port: 80                                   # puerto del Service
      targetPort: 8080                           # puerto del contenedor
      protocol: TCP
```

```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-api-np
spec:
  type: NodePort
  selector:
    app: mi-api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080                            # puerto fijo en el nodo (opcional)
```

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: mi-api-lb
spec:
  type: LoadBalancer
  selector:
    app: mi-api
  ports:
    - port: 80
      targetPort: 8080
```

### 5.4 DNS interno de Kubernetes

Kubernetes tiene un servidor DNS interno. Cada Service obtiene un registro DNS:

```
<service-name>.<namespace>.svc.cluster.local

Ejemplos:
  mi-api.default.svc.cluster.local       -> Service "mi-api" en namespace "default"
  postgres.database.svc.cluster.local    -> Service "postgres" en namespace "database"

Formas cortas (dentro del mismo namespace):
  mi-api                                 -> funciona si estas en "default"
  mi-api.default                         -> funciona desde cualquier namespace
```

```yaml
# En tu app Spring Boot, la URL de la base de datos seria:
# spring.datasource.url=jdbc:postgresql://postgres.database:5432/midb
# "postgres.database" se resuelve por DNS interno de K8s
```

---

## 6. Namespaces: aislamiento logico

### 6.1 Que son los Namespaces

Los Namespaces dividen un cluster en particiones logicas. Son utiles para separar
entornos (dev, staging, prod) o equipos en el mismo cluster.

```
Cluster
  +-- Namespace: default         (el namespace por defecto)
  +-- Namespace: kube-system     (componentes internos de K8s)
  +-- Namespace: dev             (entorno de desarrollo)
  +-- Namespace: staging         (entorno de staging)
  +-- Namespace: prod            (entorno de produccion)
```

### 6.2 Operaciones con Namespaces

```bash
# Listar namespaces
kubectl get namespaces

# Crear un namespace
kubectl create namespace dev

# O con YAML:
# namespace.yaml
# apiVersion: v1
# kind: Namespace
# metadata:
#   name: dev

# Desplegar en un namespace especifico
kubectl apply -f deployment.yaml -n dev

# Ver Pods en un namespace
kubectl get pods -n dev

# Ver Pods en todos los namespaces
kubectl get pods -A

# Cambiar el namespace por defecto para kubectl
kubectl config set-context --current --namespace=dev
```

### 6.3 Cuando usar Namespaces

| Escenario | Recomendacion |
|-----------|--------------|
| App pequena, un equipo | Un solo namespace (`default`) es suficiente |
| Multiples entornos en un cluster | Un namespace por entorno |
| Multiples equipos | Un namespace por equipo |
| Multi-tenant | Namespaces + RBAC + Network Policies |

---

## 7. kubectl: la herramienta esencial

### 7.1 Instalacion

```bash
# Linux
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/

# macOS
brew install kubectl

# Verificar
kubectl version --client
```

### 7.2 Comandos fundamentales

```bash
# --- Informacion del cluster ---
kubectl cluster-info                             # info del cluster
kubectl get nodes                                # nodos del cluster

# --- CRUD de recursos ---
kubectl apply -f archivo.yaml                    # crear o actualizar
kubectl get <recurso>                            # listar
kubectl describe <recurso> <nombre>              # detalle completo
kubectl delete <recurso> <nombre>                # eliminar
kubectl delete -f archivo.yaml                   # eliminar por manifiesto

# --- Pods ---
kubectl get pods                                 # listar Pods
kubectl get pods -o wide                         # con IP y nodo
kubectl describe pod mi-api-abc12                # detalle de un Pod
kubectl logs mi-api-abc12                        # logs del contenedor
kubectl logs mi-api-abc12 -f                     # follow logs
kubectl logs mi-api-abc12 --previous             # logs del contenedor anterior (si reinicio)
kubectl exec -it mi-api-abc12 -- /bin/sh         # shell interactivo

# --- Deployments ---
kubectl get deployments                          # listar
kubectl describe deployment mi-api               # detalle
kubectl scale deployment mi-api --replicas=5     # escalar
kubectl rollout status deployment mi-api         # estado del rollout
kubectl rollout history deployment mi-api        # historial
kubectl rollout undo deployment mi-api           # rollback

# --- Services ---
kubectl get services                             # listar
kubectl get svc                                  # alias corto
kubectl describe svc mi-api                      # detalle

# --- Depuracion ---
kubectl get events --sort-by=.metadata.creationTimestamp  # eventos recientes
kubectl top pods                                 # uso de CPU/memoria de Pods
kubectl top nodes                                # uso de CPU/memoria de nodos
```

### 7.3 Aliases utiles

```bash
# En tu .bashrc o .zshrc
alias k='kubectl'
alias kgp='kubectl get pods'
alias kgs='kubectl get services'
alias kgd='kubectl get deployments'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete -f'
alias klog='kubectl logs -f'
alias kex='kubectl exec -it'

# Autocompletado (bash)
source <(kubectl completion bash)
complete -o default -F __start_kubectl k
```

---

## 8. Estructura de manifiestos YAML

### 8.1 Los cuatro campos obligatorios

Todo manifiesto YAML de Kubernetes tiene esta estructura:

```yaml
apiVersion: apps/v1              # version del API de Kubernetes
kind: Deployment                  # tipo de recurso
metadata:                         # datos sobre el recurso (nombre, labels, namespace)
  name: mi-api
  namespace: default
  labels:
    app: mi-api
spec:                             # especificacion del estado deseado
  replicas: 3
  # ... (depende del kind)
```

### 8.2 Labels y selectors

Labels son pares clave-valor que se asignan a cualquier recurso. Los selectors filtran
recursos por sus labels. Son fundamentales para que Services encuentren Pods:

```yaml
# El Deployment crea Pods con label app=mi-api
# El Service selecciona Pods con label app=mi-api
# Asi se conectan: por labels, no por nombre

# Deployment
spec:
  selector:
    matchLabels:
      app: mi-api               # selecciona Pods con esta label
  template:
    metadata:
      labels:
        app: mi-api             # los Pods creados tendran esta label

# Service
spec:
  selector:
    app: mi-api                 # enruta trafico a Pods con esta label
```

### 8.3 Multiples recursos en un archivo

Puedes definir multiples recursos en un solo archivo separados por `---`:

```yaml
# app.yaml (Deployment + Service juntos)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-api
spec:
  replicas: 3
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
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: mi-api
spec:
  type: ClusterIP
  selector:
    app: mi-api
  ports:
    - port: 80
      targetPort: 8080
```

---

## 9. Entorno local: Minikube y Kind

### 9.1 Minikube

Minikube crea un cluster K8s de un solo nodo en tu maquina local. Ideal para aprender.

```bash
# Instalar Minikube
# Linux
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# macOS
brew install minikube

# Crear cluster
minikube start                                   # cluster con configuracion por defecto
minikube start --cpus=4 --memory=4096            # con recursos especificos
minikube start --driver=docker                   # usar Docker como driver

# Comandos utiles
minikube status                                  # estado del cluster
minikube dashboard                               # abrir dashboard web
minikube service mi-api                          # abrir un Service en el navegador
minikube tunnel                                  # habilitar LoadBalancer en local

# Usar el Docker daemon de Minikube (para no tener que push a un registry)
eval $(minikube docker-env)
docker build -t mi-api:1.0.0 .                  # la imagen estara disponible en Minikube

# Detener y eliminar
minikube stop
minikube delete
```

### 9.2 Kind (Kubernetes in Docker)

Kind corre clusters K8s dentro de contenedores Docker. Es mas ligero que Minikube y
permite multiples nodos:

```bash
# Instalar Kind
# Linux
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind

# macOS
brew install kind

# Crear cluster basico
kind create cluster --name mi-cluster

# Crear cluster multi-nodo
kind create cluster --name multi --config kind-config.yaml
```

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
# Cargar imagenes locales en Kind (sin registry)
kind load docker-image mi-api:1.0.0 --name mi-cluster

# Eliminar cluster
kind delete cluster --name mi-cluster
```

---

## 10. Desplegando una app Spring Boot

### 10.1 Preparar la imagen

```dockerfile
# Dockerfile
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
RUN addgroup -S app && adduser -S app -G app
USER app
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

```bash
# Construir y cargar en Minikube
eval $(minikube docker-env)
docker build -t mi-spring-api:1.0.0 .
```

### 10.2 Manifiestos completos

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: mi-app
---
# k8s/postgres.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: mi-app
spec:
  replicas: 1
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
            - name: POSTGRES_USER
              value: miapp
            - name: POSTGRES_PASSWORD
              value: secreto                     # en produccion usar Secrets (nivel 03)
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: mi-app
spec:
  type: ClusterIP
  selector:
    app: postgres
  ports:
    - port: 5432
      targetPort: 5432
---
# k8s/api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-spring-api
  namespace: mi-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: mi-spring-api
  template:
    metadata:
      labels:
        app: mi-spring-api
    spec:
      containers:
        - name: api
          image: mi-spring-api:1.0.0
          imagePullPolicy: IfNotPresent          # para imagenes locales
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "prod"
            - name: SPRING_DATASOURCE_URL
              value: "jdbc:postgresql://postgres.mi-app:5432/midb"
            - name: SPRING_DATASOURCE_USERNAME
              value: "miapp"
            - name: SPRING_DATASOURCE_PASSWORD
              value: "secreto"
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: mi-spring-api
  namespace: mi-app
spec:
  type: NodePort
  selector:
    app: mi-spring-api
  ports:
    - port: 80
      targetPort: 8080
      nodePort: 30080
```

### 10.3 Desplegar y verificar

```bash
# Aplicar todos los manifiestos
kubectl apply -f k8s/

# Verificar el despliegue
kubectl get all -n mi-app
# NAME                                READY   STATUS    RESTARTS   AGE
# pod/postgres-xxx                    1/1     Running   0          30s
# pod/mi-spring-api-xxx               1/1     Running   0          30s
# pod/mi-spring-api-yyy               1/1     Running   0          30s
#
# NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
# service/postgres        ClusterIP   10.96.10.1     <none>        5432/TCP
# service/mi-spring-api   NodePort    10.96.10.2     <none>        80:30080/TCP

# Ver logs de la app
kubectl logs -f deployment/mi-spring-api -n mi-app

# Acceder desde Minikube
minikube service mi-spring-api -n mi-app

# O directamente
curl $(minikube ip):30080/actuator/health

# Probar la API
curl $(minikube ip):30080/api/productos

# Escalar la API
kubectl scale deployment mi-spring-api --replicas=4 -n mi-app

# Verificar que hay 4 Pods
kubectl get pods -n mi-app -l app=mi-spring-api
```

### 10.4 Port-forward para depuracion

```bash
# Redirigir un puerto local al Pod (sin necesidad de Service)
kubectl port-forward pod/mi-spring-api-xxx 8080:8080 -n mi-app

# Ahora puedes acceder en http://localhost:8080
curl http://localhost:8080/actuator/health

# Tambien funciona con Services
kubectl port-forward svc/mi-spring-api 8080:80 -n mi-app
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| Kubernetes | Orquestador de contenedores | Gestiona despliegue, escalado y operaciones de contenedores |
| Control Plane | Cerebro del cluster | API Server, etcd, Scheduler, Controller Manager |
| Worker Node | Maquina que ejecuta Pods | kubelet + kube-proxy + container runtime |
| Pod | Unidad minima de ejecucion | Uno o mas contenedores con red y storage compartidos |
| ReplicaSet | Mantiene N replicas de un Pod | Creado automaticamente por Deployments |
| Deployment | Gestiona despliegue declarativo | `replicas`, `strategy`, rolling updates, rollbacks |
| Service ClusterIP | IP estable interna con DNS | Comunicacion entre microservicios |
| Service NodePort | Puerto en cada nodo (30000-32767) | Desarrollo y testing |
| Service LoadBalancer | IP publica via cloud LB | Produccion en AWS/GCP |
| Namespace | Aislamiento logico de recursos | Separar entornos o equipos |
| kubectl | CLI para interactuar con K8s | `apply`, `get`, `describe`, `logs`, `exec` |
| Labels | Pares clave-valor en recursos | Seleccionar Pods desde Services y Deployments |
| Minikube | Cluster K8s local de un nodo | `minikube start` para aprender |
| Kind | Cluster K8s en contenedores Docker | Multi-nodo, mas ligero que Minikube |
| Port-forward | Redirigir puerto local a Pod | `kubectl port-forward` para depuracion |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **Pods y Deployments** | Crear un Pod basico, luego un Deployment con 3 replicas. Matar un Pod y verificar que se recrea. Escalar a 5 y volver a 2 | Pod, Deployment, ReplicaSet, `kubectl scale`, `kubectl delete pod` |
| 02 | **Services** | Crear un Deployment y exponerlo con ClusterIP, NodePort y LoadBalancer. Verificar conectividad con curl y DNS interno | Service types, DNS, `kubectl port-forward`, `minikube service` |
| 03 | **Namespaces** | Crear namespaces dev y prod, desplegar la misma app en ambos con configuraciones diferentes. Verificar aislamiento | Namespaces, `-n`, labels, contextos |
| 04 | **Deploy Spring App** | Desplegar una app Spring Boot con PostgreSQL en Minikube. La app debe conectarse a la BD por DNS interno y responder a peticiones HTTP | Deployment, Service, DNS, env vars, port-forward, logs |

---

## Recursos

- [Kubernetes Official Documentation](https://kubernetes.io/docs/home/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)
- [Minikube Handbook](https://minikube.sigs.k8s.io/docs/handbook/)
- [Kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way)
- [YAML Syntax for K8s](https://kubernetes.io/docs/concepts/overview/working-with-objects/)

---

> **Antes de continuar al siguiente nivel**: asegurate de que puedes desplegar un
> Deployment con un Service y acceder a la app desde fuera del cluster. Practica
> matando Pods y verificando que el ReplicaSet los recrea. Esta auto-recuperacion
> es la base de la fiabilidad en Kubernetes.
