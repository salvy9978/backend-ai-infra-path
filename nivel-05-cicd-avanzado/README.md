# Nivel 05 — CI/CD Avanzado con GitHub Actions

> Tu codigo ya funciona en local. Ahora necesitas que cada push compile, testee, construya una imagen Docker y despliegue automaticamente. Un pipeline robusto es la diferencia entre un proyecto hobby y uno de produccion.

---

## Contenido

- [1. GitHub Actions en profundidad](#1-github-actions-en-profundidad)
  - [1.1 Conceptos fundamentales](#11-conceptos-fundamentales)
  - [1.2 Anatomia de un workflow](#12-anatomia-de-un-workflow)
  - [1.3 Triggers (eventos)](#13-triggers-eventos)
  - [1.4 Jobs y steps](#14-jobs-y-steps)
  - [1.5 Expresiones y contextos](#15-expresiones-y-contextos)
- [2. Matrix builds](#2-matrix-builds)
  - [2.1 Para que sirven](#21-para-que-sirven)
  - [2.2 Sintaxis y combinaciones](#22-sintaxis-y-combinaciones)
  - [2.3 Excluir e incluir combinaciones](#23-excluir-e-incluir-combinaciones)
- [3. Reutilizacion de workflows](#3-reutilizacion-de-workflows)
  - [3.1 Reusable workflows](#31-reusable-workflows)
  - [3.2 Composite actions](#32-composite-actions)
  - [3.3 Cuando usar cada uno](#33-cuando-usar-cada-uno)
- [4. Build y push de imagenes Docker](#4-build-y-push-de-imagenes-docker)
  - [4.1 Construir imagen en CI](#41-construir-imagen-en-ci)
  - [4.2 Push a GHCR](#42-push-a-ghcr)
  - [4.3 Push a ECR](#43-push-a-ecr)
  - [4.4 Push a GCR/Artifact Registry](#44-push-a-gcrartifact-registry)
  - [4.5 Multi-platform builds](#45-multi-platform-builds)
- [5. Testing en CI](#5-testing-en-ci)
  - [5.1 Tests unitarios](#51-tests-unitarios)
  - [5.2 Tests de integracion con servicios](#52-tests-de-integracion-con-servicios)
  - [5.3 Tests end-to-end](#53-tests-end-to-end)
- [6. Estrategias de despliegue](#6-estrategias-de-despliegue)
  - [6.1 Rolling update](#61-rolling-update)
  - [6.2 Blue-green](#62-blue-green)
  - [6.3 Canary](#63-canary)
- [7. Environments y approval gates](#7-environments-y-approval-gates)
- [8. Gestion de secretos](#8-gestion-de-secretos)
- [9. Cache de dependencias](#9-cache-de-dependencias)
- [10. Gestion de artefactos](#10-gestion-de-artefactos)
- [11. Resumen](#11-resumen)
- [12. Ejercicios](#12-ejercicios)
- [13. Recursos](#13-recursos)

---

## 1. GitHub Actions en profundidad

### 1.1 Conceptos fundamentales

GitHub Actions es el sistema de CI/CD integrado en GitHub. Ejecuta automatizaciones (_workflows_) en respuesta a eventos del repositorio.

La jerarquia es:

```
Repositorio
  └── .github/workflows/
        └── ci.yml                  <-- archivo de workflow
              ├── on: (trigger)     <-- cuando se ejecuta
              ├── jobs:
              │     ├── build:      <-- un job
              │     │     ├── runs-on: ubuntu-latest
              │     │     └── steps:
              │     │           ├── step 1 (checkout)
              │     │           ├── step 2 (setup JDK)
              │     │           └── step 3 (mvn test)
              │     └── deploy:     <-- otro job
              │           └── needs: [build]  <-- depende del anterior
```

| Concepto | Que es | Analogia |
|----------|--------|----------|
| **Workflow** | Un archivo YAML que define el pipeline completo | La receta entera |
| **Event/Trigger** | Lo que dispara el workflow | El boton de inicio |
| **Job** | Un conjunto de steps que corren en el mismo runner | Una estacion de trabajo |
| **Step** | Una accion individual dentro de un job | Un paso de la receta |
| **Runner** | La maquina (VM) donde ejecuta el job | El servidor fisico |
| **Action** | Un step reutilizable empaquetado | Una herramienta prefabricada |

### 1.2 Anatomia de un workflow

```yaml
# .github/workflows/ci.yml
name: CI Pipeline                    # nombre visible en la UI de GitHub

on:                                  # triggers: cuando se ejecuta
  push:
    branches: [main, develop]        # solo en estas ramas
  pull_request:
    branches: [main]                 # PRs que apuntan a main

permissions:                         # principio de minimo privilegio
  contents: read
  packages: write

env:                                 # variables globales del workflow
  JAVA_VERSION: '21'
  REGISTRY: ghcr.io

jobs:
  build:                             # nombre del job
    runs-on: ubuntu-latest           # tipo de runner
    timeout-minutes: 15              # timeout para evitar jobs colgados

    steps:
      - name: Checkout codigo        # paso 1: clonar el repo
        uses: actions/checkout@v4

      - name: Setup JDK              # paso 2: instalar JDK
        uses: actions/setup-java@v4
        with:
          java-version: ${{ env.JAVA_VERSION }}
          distribution: 'temurin'
          cache: 'maven'             # cachea ~/.m2/repository

      - name: Compilar y testear     # paso 3: ejecutar maven
        run: mvn clean verify -B     # -B = modo batch (no interactivo)

      - name: Subir reporte de tests # paso 4: guardar artefactos
        if: always()                 # ejecuta incluso si el paso anterior falla
        uses: actions/upload-artifact@v4
        with:
          name: test-reports
          path: target/surefire-reports/
```

Cada step puede ser una **action** (`uses:`) o un **comando shell** (`run:`). Nunca ambos en el mismo step.

### 1.3 Triggers (eventos)

Los triggers mas usados en proyectos reales:

```yaml
on:
  # Push a ramas especificas
  push:
    branches: [main, develop]
    paths:                           # solo si cambian estos archivos
      - 'src/**'
      - 'pom.xml'
    tags:
      - 'v*'                        # tags que empiezan con v (releases)

  # Pull requests
  pull_request:
    types: [opened, synchronize, reopened]

  # Ejecucion manual desde la UI de GitHub
  workflow_dispatch:
    inputs:
      environment:
        description: 'Entorno destino'
        required: true
        default: 'staging'
        type: choice
        options: [staging, production]

  # Programado (cron)
  schedule:
    - cron: '0 2 * * 1'             # lunes a las 2:00 AM UTC

  # Cuando otro workflow termina
  workflow_run:
    workflows: ["CI Pipeline"]
    types: [completed]
```

**Gotcha**: `pull_request` desde forks no tiene acceso a secretos del repositorio por seguridad. Usa `pull_request_target` con cuidado si necesitas secretos en PRs de forks.

### 1.4 Jobs y steps

Los jobs corren en **paralelo** por defecto. Para ejecutar en secuencia, usa `needs`:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn test

  lint:
    runs-on: ubuntu-latest           # corre en paralelo con test
    steps:
      - uses: actions/checkout@v4
      - run: mvn checkstyle:check

  build:
    needs: [test, lint]              # espera a que AMBOS terminen
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: mvn package -DskipTests

  deploy:
    needs: build                     # espera solo a build
    if: github.ref == 'refs/heads/main'  # solo en main
    runs-on: ubuntu-latest
    steps:
      - run: echo "Desplegando..."
```

Compartir datos entre steps del mismo job:

```yaml
steps:
  - name: Generar version
    id: version                      # id para referenciar despues
    run: echo "tag=v$(date +%Y%m%d%H%M%S)" >> $GITHUB_OUTPUT

  - name: Usar version
    run: echo "La version es ${{ steps.version.outputs.tag }}"
```

Compartir datos entre jobs diferentes (via artefactos):

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - run: mvn package -DskipTests
      - uses: actions/upload-artifact@v4
        with:
          name: app-jar
          path: target/*.jar

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: app-jar
      - run: ls -la *.jar
```

### 1.5 Expresiones y contextos

GitHub Actions proporciona contextos con informacion del workflow:

| Contexto | Contenido | Ejemplo |
|----------|-----------|---------|
| `github` | Info del evento y repo | `github.ref`, `github.sha`, `github.actor` |
| `env` | Variables de entorno | `env.JAVA_VERSION` |
| `secrets` | Secretos configurados | `secrets.DOCKER_PASSWORD` |
| `steps` | Outputs de steps anteriores | `steps.version.outputs.tag` |
| `matrix` | Valores de la matrix actual | `matrix.java-version` |
| `needs` | Outputs de jobs anteriores | `needs.build.outputs.image-tag` |

Condicionales utiles:

```yaml
# Solo en la rama main
if: github.ref == 'refs/heads/main'

# Solo si el job anterior tuvo exito
if: needs.build.result == 'success'

# Solo en push (no en PR)
if: github.event_name == 'push'

# Ejecutar siempre, incluso si un step anterior fallo
if: always()

# Solo si un step anterior fallo
if: failure()

# Combinar condiciones
if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

---

## 2. Matrix builds

### 2.1 Para que sirven

Una matrix build ejecuta el mismo job con multiples combinaciones de parametros. Clasico ejemplo: testear en varias versiones de Java y varios sistemas operativos.

Sin matrix, necesitas un job por cada combinacion. Con matrix, defines las combinaciones y GitHub Actions genera los jobs automaticamente.

### 2.2 Sintaxis y combinaciones

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest]    # 2 SOs
        java-version: ['17', '21']             # 2 versiones de Java
        # Total: 2 x 2 = 4 jobs en paralelo
      fail-fast: false                         # no cancelar otros si uno falla
      max-parallel: 2                          # maximo 2 jobs simultaneos

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ matrix.java-version }}
          distribution: 'temurin'
      - run: mvn test
```

### 2.3 Excluir e incluir combinaciones

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, windows-latest, macos-latest]
    java-version: ['17', '21']

    # Excluir combinaciones que no interesan
    exclude:
      - os: macos-latest
        java-version: '17'          # no testear Java 17 en macOS

    # Incluir combinaciones extra con propiedades adicionales
    include:
      - os: ubuntu-latest
        java-version: '21'
        experimental: true           # propiedad custom
        maven-args: '-Pintegration'  # argumento extra solo para esta combinacion
```

Acceder a propiedades custom:

```yaml
- run: mvn test ${{ matrix.maven-args }}
  if: matrix.experimental != true    # saltar si es experimental
```

---

## 3. Reutilizacion de workflows

### 3.1 Reusable workflows

Un workflow reutilizable es un workflow completo que otros workflows pueden llamar como si fuera un step. Vive en `.github/workflows/` y usa `workflow_call` como trigger.

Workflow reutilizable (el "template"):

```yaml
# .github/workflows/reusable-build.yml
name: Build Reutilizable

on:
  workflow_call:                     # permite ser llamado por otros workflows
    inputs:
      java-version:
        required: true
        type: string
        default: '21'
      run-tests:
        required: false
        type: boolean
        default: true
    secrets:
      SONAR_TOKEN:                   # secretos que necesita recibir
        required: false
    outputs:
      artifact-name:
        description: "Nombre del artefacto generado"
        value: ${{ jobs.build.outputs.artifact-name }}

jobs:
  build:
    runs-on: ubuntu-latest
    outputs:
      artifact-name: ${{ steps.meta.outputs.name }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: ${{ inputs.java-version }}
          distribution: 'temurin'
          cache: 'maven'
      - name: Build
        run: mvn clean package ${{ inputs.run-tests && '' || '-DskipTests' }}
      - name: Meta
        id: meta
        run: echo "name=app-$(date +%s)" >> $GITHUB_OUTPUT
```

Workflow que lo llama:

```yaml
# .github/workflows/ci.yml
name: CI
on: [push]

jobs:
  call-build:
    uses: ./.github/workflows/reusable-build.yml    # ruta relativa
    with:
      java-version: '21'
      run-tests: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  deploy:
    needs: call-build
    runs-on: ubuntu-latest
    steps:
      - run: echo "Artefacto ${{ needs.call-build.outputs.artifact-name }}"
```

Tambien puedes referenciar workflows de otros repositorios:

```yaml
uses: mi-org/shared-workflows/.github/workflows/build.yml@main
```

### 3.2 Composite actions

Una composite action es un step reutilizable (no un workflow entero). Vive en su propio directorio con un `action.yml`.

```yaml
# .github/actions/setup-proyecto/action.yml
name: 'Setup Proyecto'
description: 'Configura JDK, Maven cache y variables'

inputs:
  java-version:
    description: 'Version de Java'
    required: false
    default: '21'

outputs:
  cache-hit:
    description: 'Si hubo cache hit de Maven'
    value: ${{ steps.cache.outputs.cache-hit }}

runs:
  using: 'composite'                 # tipo composite
  steps:
    - name: Setup JDK
      uses: actions/setup-java@v4
      with:
        java-version: ${{ inputs.java-version }}
        distribution: 'temurin'

    - name: Cache Maven
      id: cache
      uses: actions/cache@v4
      with:
        path: ~/.m2/repository
        key: maven-${{ hashFiles('**/pom.xml') }}

    - name: Mostrar versiones
      shell: bash                    # obligatorio en composite actions
      run: |
        java -version
        mvn --version
```

Uso en un workflow:

```yaml
steps:
  - uses: actions/checkout@v4
  - uses: ./.github/actions/setup-proyecto   # ruta al directorio
    with:
      java-version: '21'
  - run: mvn test
```

### 3.3 Cuando usar cada uno

| Caracteristica | Reusable Workflow | Composite Action |
|----------------|:-----------------:|:----------------:|
| Granularidad | Workflow completo (jobs) | Un step individual |
| Donde vive | `.github/workflows/` | Cualquier directorio con `action.yml` |
| Puede tener jobs | Si | No |
| Se usa con | `uses:` a nivel de job | `uses:` a nivel de step |
| Runners propios | Si | No (usa el del job padre) |
| Ideal para | Pipelines enteros reutilizables | Pasos comunes (setup, lint, etc.) |

**Regla practica**: si necesitas reutilizar un step o grupo de steps, usa composite action. Si necesitas reutilizar un pipeline completo con multiples jobs, usa reusable workflow.

---

## 4. Build y push de imagenes Docker

### 4.1 Construir imagen en CI

El Dockerfile tipico para una app Spring Boot:

```dockerfile
# Dockerfile multi-stage para Spring Boot
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app
COPY pom.xml .
COPY src ./src
# Descargar dependencias primero (aprovecha cache de Docker)
RUN --mount=type=cache,target=/root/.m2 mvn clean package -DskipTests -B

FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar
# Usuario no-root por seguridad
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

### 4.2 Push a GHCR

GitHub Container Registry (GHCR) es el registro de imagenes integrado en GitHub:

```yaml
# .github/workflows/docker.yml
name: Docker Build & Push

on:
  push:
    tags: ['v*']                     # solo cuando se crea un tag de version

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}   # owner/repo

jobs:
  build-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write                # necesario para push a GHCR

    steps:
      - uses: actions/checkout@v4

      - name: Login a GHCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}   # token automatico

      - name: Extraer metadata (tags, labels)
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=semver,pattern={{version}}       # v1.2.3 -> 1.2.3
            type=semver,pattern={{major}}.{{minor}}  # v1.2.3 -> 1.2
            type=sha                              # sha-abc123

      - name: Build y Push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha                    # cache de GitHub Actions
          cache-to: type=gha,mode=max
```

### 4.3 Push a ECR

Amazon Elastic Container Registry:

```yaml
- name: Configurar credenciales AWS
  uses: aws-actions/configure-aws-credentials@v4
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: us-east-1

- name: Login a ECR
  id: ecr-login
  uses: aws-actions/amazon-ecr-login@v2

- name: Build y Push a ECR
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: ${{ steps.ecr-login.outputs.registry }}/mi-app:${{ github.sha }}
```

### 4.4 Push a GCR/Artifact Registry

Google Cloud Artifact Registry (sucesor de GCR):

```yaml
- name: Autenticar con Google Cloud
  uses: google-github-actions/auth@v2
  with:
    credentials_json: ${{ secrets.GCP_SA_KEY }}

- name: Configurar Docker para Artifact Registry
  run: gcloud auth configure-docker us-central1-docker.pkg.dev

- name: Build y Push
  uses: docker/build-push-action@v5
  with:
    context: .
    push: true
    tags: us-central1-docker.pkg.dev/mi-proyecto/mi-repo/mi-app:${{ github.sha }}
```

### 4.5 Multi-platform builds

Para construir imagenes que funcionen en ARM y x86:

```yaml
- name: Setup QEMU
  uses: docker/setup-qemu-action@v3    # emulador para multi-arch

- name: Setup Buildx
  uses: docker/setup-buildx-action@v3  # builder avanzado

- name: Build multi-platform
  uses: docker/build-push-action@v5
  with:
    context: .
    platforms: linux/amd64,linux/arm64  # ambas arquitecturas
    push: true
    tags: ghcr.io/mi-org/mi-app:latest
```

---

## 5. Testing en CI

### 5.1 Tests unitarios

Los tests unitarios son los mas rapidos y deben correr en cada push:

```yaml
- name: Tests unitarios
  run: mvn test -B
  # Maven ejecuta surefire-plugin que busca clases *Test.java

- name: Publicar resultados
  if: always()
  uses: dorny/test-reporter@v1
  with:
    name: Unit Tests
    path: target/surefire-reports/*.xml
    reporter: java-junit
```

### 5.2 Tests de integracion con servicios

GitHub Actions permite levantar servicios (contenedores) junto al job:

```yaml
jobs:
  integration:
    runs-on: ubuntu-latest

    services:
      postgres:                      # nombre del servicio (hostname)
        image: postgres:16-alpine
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: testuser
          POSTGRES_PASSWORD: testpass
        ports:
          - 5432:5432
        options: >-                  # healthcheck para esperar que arranque
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'

      - name: Tests de integracion
        env:
          SPRING_DATASOURCE_URL: jdbc:postgresql://localhost:5432/testdb
          SPRING_DATASOURCE_USERNAME: testuser
          SPRING_DATASOURCE_PASSWORD: testpass
          SPRING_REDIS_HOST: localhost
        run: mvn verify -Pintegration -B
```

### 5.3 Tests end-to-end

Los tests e2e levantan la aplicacion completa y la prueban como un usuario real:

```yaml
- name: Build de la app
  run: mvn package -DskipTests -B

- name: Arrancar aplicacion en background
  run: |
    java -jar target/app.jar &
    sleep 15                         # esperar a que arranque

- name: Health check
  run: curl --fail http://localhost:8080/actuator/health

- name: Tests e2e
  run: mvn test -Pe2e -B             # perfil Maven para tests e2e

- name: Parar aplicacion
  if: always()
  run: kill $(lsof -t -i:8080) || true
```

---

## 6. Estrategias de despliegue

### 6.1 Rolling update

Reemplaza las instancias antiguas gradualmente. Si hay 4 replicas, actualiza una a una.

```
Tiempo 0:  [v1] [v1] [v1] [v1]    <- todas en version 1
Tiempo 1:  [v2] [v1] [v1] [v1]    <- primera actualizada
Tiempo 2:  [v2] [v2] [v1] [v1]    <- segunda actualizada
Tiempo 3:  [v2] [v2] [v2] [v1]    <- tercera actualizada
Tiempo 4:  [v2] [v2] [v2] [v2]    <- todas en version 2
```

En Kubernetes:

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mi-app
spec:
  replicas: 4
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                    # maximo 1 pod extra durante update
      maxUnavailable: 1              # maximo 1 pod no disponible
```

**Ventaja**: zero downtime, bajo uso de recursos extra. **Desventaja**: durante el despliegue conviven v1 y v2, lo que puede causar problemas de compatibilidad.

### 6.2 Blue-green

Mantienes dos entornos identicos. "Blue" es produccion actual, "Green" es la nueva version. Cuando Green esta listo, cambias el trafico de golpe.

```
Estado inicial:
  Load Balancer ----> [Blue: v1]  (produccion)
                      [Green: idle]

Desplegando v2:
  Load Balancer ----> [Blue: v1]  (produccion)
                      [Green: v2] (testeando)

Switch:
  Load Balancer ----> [Green: v2] (nueva produccion)
                      [Blue: v1]  (rollback instantaneo si falla)
```

```yaml
# Workflow simplificado de blue-green
- name: Deploy a Green
  run: |
    kubectl apply -f k8s/green-deployment.yaml
    kubectl rollout status deployment/mi-app-green

- name: Smoke tests en Green
  run: curl --fail http://green.mi-app.internal/health

- name: Switch trafico a Green
  run: |
    kubectl patch service mi-app \
      -p '{"spec":{"selector":{"version":"green"}}}'

- name: Verificar produccion
  run: curl --fail http://mi-app.produccion.com/health
```

**Ventaja**: rollback instantaneo (cambias el selector de vuelta). **Desventaja**: necesitas el doble de infraestructura.

### 6.3 Canary

Envias un porcentaje pequeno del trafico a la nueva version. Si las metricas son buenas, incrementas gradualmente.

```
Fase 1:  95% trafico -> [v1]    5% trafico -> [v2]
Fase 2:  75% trafico -> [v1]   25% trafico -> [v2]
Fase 3:  50% trafico -> [v1]   50% trafico -> [v2]
Fase 4:   0% trafico -> [v1]  100% trafico -> [v2]
```

Con Istio o un service mesh:

```yaml
# VirtualService de Istio para canary
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: mi-app
spec:
  hosts:
    - mi-app
  http:
    - route:
        - destination:
            host: mi-app
            subset: stable           # v1
          weight: 90
        - destination:
            host: mi-app
            subset: canary           # v2
          weight: 10
```

**Ventaja**: riesgo minimo, puedes detectar problemas con poco trafico. **Desventaja**: mas complejo de implementar, necesitas buena observabilidad para decidir si promover o rollback.

---

## 7. Environments y approval gates

GitHub Actions permite definir entornos con reglas de proteccion:

```yaml
jobs:
  deploy-staging:
    runs-on: ubuntu-latest
    environment:
      name: staging                  # entorno sin proteccion
      url: https://staging.mi-app.com
    steps:
      - run: echo "Deploy a staging"

  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production               # entorno con aprobadores configurados
      url: https://mi-app.com
    steps:
      - run: echo "Deploy a produccion"
```

En la configuracion del repositorio (Settings > Environments > production):
- **Required reviewers**: personas que deben aprobar antes del deploy
- **Wait timer**: tiempo de espera minimo (ej: 30 minutos)
- **Deployment branches**: solo permitir deploys desde `main`

Los secretos tambien se pueden configurar por entorno. Un secreto en el entorno `production` solo esta disponible en jobs que usan ese entorno.

---

## 8. Gestion de secretos

Nunca pongas credenciales en el codigo. GitHub Actions ofrece tres niveles de secretos:

| Nivel | Alcance | Donde se configura |
|-------|---------|-------------------|
| **Repository secrets** | Un repositorio | Settings > Secrets > Actions |
| **Environment secrets** | Un entorno especifico | Settings > Environments > Secrets |
| **Organization secrets** | Todos los repos de la org | Org Settings > Secrets |

Uso en workflows:

```yaml
steps:
  - name: Usar secreto
    env:
      DB_PASSWORD: ${{ secrets.DB_PASSWORD }}
    run: echo "Conectando con password..." # nunca imprimas el valor

  - name: Login con secreto
    uses: docker/login-action@v3
    with:
      username: ${{ secrets.DOCKER_USER }}
      password: ${{ secrets.DOCKER_TOKEN }}
```

**Gotchas de seguridad**:
- GitHub enmascara los secretos en los logs, pero si los codificas en base64 o los modificas, pueden filtrarse
- Los secretos no se pasan a workflows de forks (proteccion contra PRs maliciosos)
- Rota los secretos periodicamente
- Usa tokens con el minimo alcance necesario

Para secretos mas complejos (certificados, archivos), codifica en base64:

```yaml
- name: Decodificar certificado
  run: echo "${{ secrets.SSL_CERT_BASE64 }}" | base64 -d > cert.pem
```

---

## 9. Cache de dependencias

El cache evita descargar las mismas dependencias en cada ejecucion. Ahorra minutos en cada build.

Cache integrado en setup-java:

```yaml
- uses: actions/setup-java@v4
  with:
    java-version: '21'
    distribution: 'temurin'
    cache: 'maven'                   # cachea ~/.m2/repository automaticamente
```

Cache manual con mas control:

```yaml
- name: Cache Maven
  uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: maven-${{ runner.os }}-${{ hashFiles('**/pom.xml') }}
    restore-keys: |
      maven-${{ runner.os }}-       # fallback si el pom cambio
```

Cache para Docker layers:

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha             # lee cache de GitHub Actions
    cache-to: type=gha,mode=max      # guarda todas las layers en cache
```

Cache para npm/yarn (frontend en monorepo):

```yaml
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: npm-${{ hashFiles('**/package-lock.json') }}
```

**Limites**: cada repositorio tiene 10 GB de cache total. Las entradas no accedidas en 7 dias se eliminan automaticamente.

---

## 10. Gestion de artefactos

Los artefactos permiten pasar archivos entre jobs y retenerlos despues del workflow:

```yaml
# Subir artefacto
- uses: actions/upload-artifact@v4
  with:
    name: mi-app-jar                 # nombre logico
    path: target/*.jar               # archivos a subir
    retention-days: 5                # cuanto tiempo retener (default: 90)
    if-no-files-found: error         # falla si no encuentra archivos

# Descargar en otro job
- uses: actions/download-artifact@v4
  with:
    name: mi-app-jar
    path: ./artifacts/               # donde descargar
```

Descargar multiples artefactos:

```yaml
- uses: actions/download-artifact@v4
  with:
    path: ./all-artifacts/           # sin name: descarga todos
    merge-multiple: true             # combina en un directorio
```

---

## 11. Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| **Workflow** | Archivo YAML que define el pipeline | `.github/workflows/ci.yml` con triggers y jobs |
| **Matrix build** | Ejecutar job con multiples combinaciones | `strategy.matrix` con OS/versiones |
| **Reusable workflow** | Workflow que otros pueden llamar | `workflow_call` como trigger, `uses:` para invocar |
| **Composite action** | Step reutilizable empaquetado | `action.yml` con `runs.using: composite` |
| **GHCR push** | Subir imagen Docker a GitHub | `docker/login-action` + `docker/build-push-action` |
| **Services** | Contenedores auxiliares en CI | `services:` en el job (postgres, redis) |
| **Rolling update** | Despliegue gradual sin downtime | Reemplazar pods uno a uno |
| **Blue-green** | Dos entornos, switch instantaneo | Cambiar selector del Service |
| **Canary** | Porcentaje del trafico a nueva version | Weights en VirtualService/Ingress |
| **Environments** | Entornos con reglas de proteccion | `environment:` en job + approval gates |
| **Secretos** | Credenciales seguras en CI | `secrets.NOMBRE` a nivel repo/env/org |
| **Cache** | Reutilizar dependencias entre runs | `actions/cache` o cache integrado |
| **Artefactos** | Pasar archivos entre jobs | `upload-artifact` / `download-artifact` |

---

## 12. Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | [Matrix de compatibilidad](ejercicio-01-github-actions-matrix/) | Configurar un workflow que testee en Java 17 y 21 sobre Ubuntu y Windows, con cache de Maven y reporte de tests | matrix builds, fail-fast, cache, test reporter |
| 02 | [Build y push Docker](ejercicio-02-docker-build-push/) | Crear un pipeline que construya una imagen Docker multi-stage y la publique en GHCR con tags semanticos | docker/build-push-action, metadata-action, GHCR login |
| 03 | [Estrategias de deploy](ejercicio-03-deploy-strategies/) | Implementar despliegues rolling, blue-green y canary en un cluster Kubernetes simulado con Kind | rolling update, blue-green switch, canary weights |
| 04 | [Pipeline completo](ejercicio-04-pipeline-completo/) | Construir un pipeline end-to-end: lint, test, build Docker, push, deploy a staging (auto) y produccion (con aprobacion) | reusable workflows, environments, approval gates, secretos |

Haz los ejercicios en orden. Cada uno construye sobre los conceptos del anterior.

---

## 13. Recursos

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Workflow syntax reference](https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitHub Actions Marketplace](https://github.com/marketplace?type=actions)
- [docker/build-push-action](https://github.com/docker/build-push-action)
- [docker/metadata-action](https://github.com/docker/metadata-action)
- [actions/cache](https://github.com/actions/cache)
- [Kubernetes Deployment Strategies](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#strategy)
- [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
