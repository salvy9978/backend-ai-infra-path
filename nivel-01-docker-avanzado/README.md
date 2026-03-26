# Nivel 01 — Docker Avanzado

> Ya sabes construir imagenes Docker basicas y ejecutar contenedores. Este nivel
> cubre las tecnicas avanzadas que separan un Dockerfile de tutorial de uno listo
> para produccion: multi-stage builds, networking entre contenedores, Docker Compose
> con healthchecks y limites de recursos, escaneo de vulnerabilidades y optimizacion
> de imagenes. Todo lo que necesitas antes de saltar a Kubernetes.

---

## Contenido

1. [Multi-stage builds: imagenes ligeras](#1-multi-stage-builds-imagenes-ligeras)
2. [Docker networking](#2-docker-networking)
3. [Docker Compose para produccion](#3-docker-compose-para-produccion)
4. [Escaneo de seguridad con Trivy](#4-escaneo-de-seguridad-con-trivy)
5. [Optimizacion de imagenes](#5-optimizacion-de-imagenes)
6. [Docker BuildKit](#6-docker-buildkit)
7. [Patrones avanzados](#7-patrones-avanzados)
8. [Resumen](#resumen)
9. [Ejercicios](#ejercicios)
10. [Recursos](#recursos)

---

## 1. Multi-stage builds: imagenes ligeras

### 1.1 El problema: imagenes enormes

Un build tipico de una app Spring Boot con Maven incluye el JDK completo, todas las
dependencias de build, los fuentes, el cache de Maven... La imagen resultante puede
pesar 800 MB o mas. En produccion solo necesitas el JRE y el JAR.

```dockerfile
# MAL: imagen unica con todo (800+ MB)
FROM eclipse-temurin:21-jdk
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests
CMD ["java", "-jar", "target/app.jar"]
# Incluye: JDK, Maven, fuentes, dependencias de build, JAR
# Tamano: ~800 MB
```

### 1.2 La solucion: multi-stage builds

Multi-stage builds usan multiples sentencias `FROM` en un solo Dockerfile. Cada `FROM`
inicia una etapa nueva. Solo la etapa final se incluye en la imagen resultante.

```dockerfile
# Etapa 1: BUILD (solo existe durante la construccion)
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app

# Copiar archivos de dependencias primero (mejor cache de capas)
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:resolve                    # descargar dependencias

# Copiar fuentes y compilar
COPY src src
RUN ./mvnw package -DskipTests                   # generar JAR

# Etapa 2: RUNTIME (esta es la imagen final)
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Crear usuario no-root
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Copiar SOLO el JAR desde la etapa de build
COPY --from=build /app/target/*.jar app.jar

# Cambiar a usuario no-root
USER appuser

# Puerto y comando
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
# Tamano final: ~200 MB (solo JRE + JAR)
```

La diferencia de tamano es dramatica:

| Estrategia | Imagen base | Tamano final |
|------------|-------------|-------------|
| Sin multi-stage | `eclipse-temurin:21-jdk` | ~800 MB |
| Multi-stage con JRE | `eclipse-temurin:21-jre` | ~350 MB |
| Multi-stage con JRE Alpine | `eclipse-temurin:21-jre-alpine` | ~200 MB |
| Multi-stage con jlink (custom JRE) | `alpine:3.19` | ~100 MB |

### 1.3 Multi-stage con jlink: JRE a medida

`jlink` crea un JRE personalizado que incluye solo los modulos que tu app necesita:

```dockerfile
# Etapa 1: BUILD
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:resolve
COPY src src
RUN ./mvnw package -DskipTests

# Etapa 2: JRE personalizado con jlink
FROM eclipse-temurin:21-jdk AS jre-build
RUN jlink \
    --add-modules java.base,java.logging,java.sql,java.naming,java.desktop,java.management,java.security.jgss,java.instrument \
    --strip-debug \                              # eliminar info de debug
    --no-man-pages \                             # sin paginas de manual
    --no-header-files \                          # sin headers C
    --compress=zip-6 \                           # compresion
    --output /custom-jre                         # directorio de salida

# Etapa 3: RUNTIME con JRE personalizado
FROM alpine:3.19 AS runtime
COPY --from=jre-build /custom-jre /opt/java
COPY --from=build /app/target/*.jar /app/app.jar

RUN addgroup -S app && adduser -S app -G app
USER app

ENV PATH="/opt/java/bin:${PATH}"
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
# Tamano final: ~100 MB
```

### 1.4 Copiar artefactos entre etapas

La directiva `COPY --from=<etapa>` permite copiar archivos de cualquier etapa anterior:

```dockerfile
# Copiar desde una etapa con nombre
COPY --from=build /app/target/*.jar app.jar

# Copiar desde una etapa por indice (0-based)
COPY --from=0 /app/target/*.jar app.jar

# Copiar desde una imagen externa (no necesita estar en el Dockerfile)
COPY --from=nginx:alpine /etc/nginx/nginx.conf /etc/nginx/nginx.conf
```

---

## 2. Docker networking

### 2.1 Drivers de red

Docker ofrece varios drivers de red. Cada uno resuelve un problema diferente:

```
+-------------------------------------------------------------------+
|  Docker Host                                                       |
|                                                                     |
|  bridge (default):                                                  |
|  +------------------+  +------------------+                         |
|  | Container A      |  | Container B      |                         |
|  | 172.17.0.2       |  | 172.17.0.3       |                         |
|  +--------+---------+  +--------+---------+                         |
|           |                      |                                  |
|           +--- docker0 bridge ---+                                  |
|                                                                     |
|  host:                                                              |
|  +------------------+                                               |
|  | Container C      |  <- usa la red del host directamente          |
|  | (sin aislamiento)|     no hay NAT, maximo rendimiento            |
|  +------------------+                                               |
|                                                                     |
|  none:                                                              |
|  +------------------+                                               |
|  | Container D      |  <- sin red (aislamiento total)               |
|  +------------------+                                               |
+-------------------------------------------------------------------+
```

| Driver | Aislamiento | Rendimiento | Uso tipico |
|--------|-------------|-------------|------------|
| `bridge` | Si (red aislada) | Bueno | Comunicacion entre contenedores en un host |
| `host` | No | Maximo | Apps que necesitan maximo throughput de red |
| `none` | Total | N/A | Contenedores sin necesidad de red |
| `overlay` | Si (multi-host) | Bueno | Docker Swarm / comunicacion entre hosts |
| `macvlan` | Si (IP propia en LAN) | Maximo | Contenedores que necesitan IP en la red fisica |

### 2.2 Redes bridge personalizadas

La red bridge por defecto (`docker0`) no tiene resolucion DNS entre contenedores.
Siempre crea redes bridge personalizadas:

```bash
# Crear una red bridge personalizada
docker network create mi-red

# Ejecutar contenedores en la misma red
docker run -d --name postgres --network mi-red \
    -e POSTGRES_PASSWORD=secret \
    postgres:16-alpine

docker run -d --name mi-api --network mi-red \
    -e SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/midb \
    -p 8080:8080 \
    mi-api:latest

# "postgres" se resuelve automaticamente por DNS interno
# No necesitas IPs, Docker lo resuelve por nombre de contenedor
```

Ventajas de redes bridge personalizadas sobre la red por defecto:

| Caracteristica | Red default | Red personalizada |
|---------------|-------------|-------------------|
| Resolucion DNS | No (solo por IP) | Si (por nombre de contenedor) |
| Aislamiento | Todos los contenedores la comparten | Solo contenedores explicitamente conectados |
| Conexion en caliente | No | Si (`docker network connect/disconnect`) |

### 2.3 Comunicacion entre contenedores

```bash
# Ver redes disponibles
docker network ls

# Inspeccionar una red (ver que contenedores estan conectados)
docker network inspect mi-red

# Conectar un contenedor existente a una red
docker network connect mi-red contenedor-existente

# Desconectar
docker network disconnect mi-red contenedor-existente

# Verificar conectividad desde dentro de un contenedor
docker exec mi-api ping postgres
docker exec mi-api nslookup postgres
```

### 2.4 Publicacion de puertos

```bash
# Mapear puerto del host al contenedor
docker run -p 8080:8080 mi-api           # host:contenedor

# Mapear solo a localhost (no accesible desde fuera)
docker run -p 127.0.0.1:8080:8080 mi-api

# Puerto aleatorio del host
docker run -p 8080 mi-api                # Docker asigna puerto del host
docker port mi-api                        # ver el mapeo asignado

# Multiples puertos
docker run -p 8080:8080 -p 8443:8443 mi-api
```

---

## 3. Docker Compose para produccion

### 3.1 Estructura basica de produccion

Un `docker-compose.yml` para produccion necesita healthchecks, restart policies,
limites de recursos y logging configurado:

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    image: mi-api:${APP_VERSION:-latest}         # version parametrizable
    container_name: mi-api
    restart: unless-stopped                       # reiniciar si falla
    ports:
      - "8080:8080"
    environment:
      SPRING_PROFILES_ACTIVE: prod
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/midb
      SPRING_DATASOURCE_USERNAME: ${DB_USER}     # desde .env
      SPRING_DATASOURCE_PASSWORD: ${DB_PASS}
    depends_on:
      postgres:
        condition: service_healthy               # esperar a que postgres este sano
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 30s                              # cada 30 segundos
      timeout: 10s                               # maximo 10s por intento
      retries: 3                                 # 3 fallos = unhealthy
      start_period: 40s                          # gracia inicial (arranque de Spring)
    deploy:
      resources:
        limits:
          cpus: "1.0"                            # maximo 1 CPU
          memory: 512M                           # maximo 512 MB RAM
        reservations:
          cpus: "0.5"                            # garantizado 0.5 CPU
          memory: 256M                           # garantizado 256 MB RAM
    logging:
      driver: json-file
      options:
        max-size: "10m"                          # maximo 10 MB por archivo de log
        max-file: "3"                            # rotar despues de 3 archivos
    networks:
      - backend

  postgres:
    image: postgres:16-alpine
    container_name: postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: midb
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASS}
    volumes:
      - postgres_data:/var/lib/postgresql/data   # datos persistentes
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql  # script inicial
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d midb"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 1G
    networks:
      - backend

  redis:
    image: redis:7-alpine
    container_name: redis
    restart: unless-stopped
    command: redis-server --requirepass ${REDIS_PASS}
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASS}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3
    volumes:
      - redis_data:/data
    deploy:
      resources:
        limits:
          memory: 256M
    networks:
      - backend

volumes:
  postgres_data:                                 # volumen con nombre (persistente)
  redis_data:

networks:
  backend:
    driver: bridge
```

### 3.2 Archivo .env para variables

```bash
# .env (mismo directorio que docker-compose.yml)
APP_VERSION=1.2.0
DB_USER=miapp
DB_PASS=secreto-seguro-123
REDIS_PASS=redis-secreto-456
```

### 3.3 Restart policies

| Politica | Comportamiento |
|----------|---------------|
| `no` | Nunca reiniciar (default) |
| `always` | Siempre reiniciar, incluso si se detuvo manualmente |
| `unless-stopped` | Reiniciar excepto si se detuvo manualmente |
| `on-failure` | Reiniciar solo si el proceso salio con error (exit code != 0) |

Para produccion: usa `unless-stopped` en la mayoria de servicios.

### 3.4 Healthchecks en detalle

El healthcheck define como Docker verifica si un contenedor esta sano:

```yaml
healthcheck:
  # Comando que se ejecuta dentro del contenedor
  test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]

  # Alternativa: usando shell
  test: ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health || exit 1"]

  # Alternativa: sin curl (para imagenes minimas)
  test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8080/actuator/health"]

  interval: 30s       # frecuencia de verificacion
  timeout: 10s        # tiempo maximo por verificacion
  retries: 3          # intentos antes de marcar como unhealthy
  start_period: 60s   # tiempo de gracia para arranque (importante para Spring Boot)
```

Estados del contenedor:
- `starting`: dentro del `start_period`, los fallos no cuentan
- `healthy`: el healthcheck pasa
- `unhealthy`: el healthcheck fallo `retries` veces consecutivas

### 3.5 Comandos de Compose esenciales

```bash
# Arrancar todo en background
docker compose up -d

# Ver estado de los servicios (incluye health)
docker compose ps

# Ver logs de un servicio
docker compose logs -f api                       # follow logs de api

# Reconstruir e iniciar
docker compose up -d --build

# Escalar un servicio (multiples replicas)
docker compose up -d --scale api=3

# Detener y eliminar todo (volumenes NO)
docker compose down

# Detener y eliminar todo INCLUYENDO volumenes (cuidado: borra datos)
docker compose down -v

# Ejecutar un comando en un servicio
docker compose exec postgres psql -U miapp -d midb
```

---

## 4. Escaneo de seguridad con Trivy

### 4.1 Por que escanear imagenes

Las imagenes Docker contienen un SO base con paquetes que pueden tener vulnerabilidades
conocidas (CVEs). Un escaneo regular es obligatorio en produccion.

### 4.2 Instalar Trivy

```bash
# Linux (Debian/Ubuntu)
sudo apt-get install wget apt-transport-https gnupg lsb-release
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
sudo apt-get update && sudo apt-get install trivy

# macOS
brew install trivy

# Como contenedor Docker (sin instalar nada)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image mi-api:latest
```

### 4.3 Escanear una imagen

```bash
# Escaneo basico
trivy image mi-api:latest

# Solo vulnerabilidades criticas y altas
trivy image --severity CRITICAL,HIGH mi-api:latest

# Formato tabla (por defecto) vs JSON
trivy image --format json --output report.json mi-api:latest

# Escanear un Dockerfile (sin construir la imagen)
trivy config Dockerfile

# Ignorar vulnerabilidades especificas (aceptar riesgo)
trivy image --ignorefile .trivyignore mi-api:latest
```

Ejemplo de salida:
```
mi-api:latest (alpine 3.19.1)
==============================
Total: 3 (CRITICAL: 1, HIGH: 2, MEDIUM: 0, LOW: 0)

+-----------+------------------+----------+-------------------+
| Library   | Vulnerability    | Severity | Fixed Version     |
+-----------+------------------+----------+-------------------+
| libcrypto | CVE-2024-0727    | HIGH     | 3.1.4-r5          |
| libssl    | CVE-2024-0727    | HIGH     | 3.1.4-r5          |
| zlib      | CVE-2023-45853   | CRITICAL | 1.3.1-r0          |
+-----------+------------------+----------+-------------------+
```

### 4.4 Archivo .trivyignore

```
# .trivyignore
# Vulnerabilidades aceptadas con justificacion
CVE-2024-0727    # No aplica: no usamos esa funcion de OpenSSL
CVE-2023-12345   # Fix no disponible, monitorizando
```

### 4.5 Integrar en CI/CD

```yaml
# GitHub Actions: escaneo en cada push
- name: Escanear imagen con Trivy
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: mi-api:${{ github.sha }}
    format: table
    severity: CRITICAL,HIGH
    exit-code: 1                                 # falla el pipeline si hay criticas
```

---

## 5. Optimizacion de imagenes

### 5.1 Layer caching: orden de instrucciones

Docker cachea cada capa del Dockerfile. Si una capa cambia, todas las capas posteriores
se invalidan. Ordena las instrucciones de menos a mas cambiante:

```dockerfile
# MAL: copiar todo al principio (cualquier cambio invalida la cache de dependencias)
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app
COPY . .                                         # si cambias cualquier archivo, rebuild total
RUN ./mvnw package -DskipTests

# BIEN: copiar dependencias primero, fuentes despues
FROM eclipse-temurin:21-jdk AS build
WORKDIR /app

# Capa 1: archivos de dependencias (cambia poco)
COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN ./mvnw dependency:go-offline                 # descargar TODAS las dependencias

# Capa 2: fuentes (cambia frecuentemente)
COPY src src
RUN ./mvnw package -DskipTests                   # solo compila, deps ya estan
```

### 5.2 .dockerignore

El archivo `.dockerignore` excluye archivos del build context. Sin el, Docker envia
todo el directorio al daemon, incluyendo `.git`, `target/`, `node_modules/`, etc.

```dockerignore
# .dockerignore
.git
.gitignore
.github
.idea
*.md
LICENSE

# Build artifacts
target/
build/
out/

# Dependencias locales
node_modules/

# Docker files
docker-compose*.yml
Dockerfile*

# Entorno
.env
.env.*
*.pem
*.key
```

### 5.3 Usuarios no-root

Por defecto, los procesos en Docker corren como root. Esto es un riesgo de seguridad:
si un atacante compromete la app, tiene acceso root al contenedor.

```dockerfile
# Alpine
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Debian/Ubuntu
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

# Verificar que la app funciona como no-root
# docker exec mi-api whoami
# -> appuser
```

### 5.4 Imagenes base ligeras

| Imagen base | Tamano | Notas |
|------------|--------|-------|
| `ubuntu:22.04` | ~77 MB | Completa, mucho mas de lo necesario |
| `debian:bookworm-slim` | ~74 MB | Debian sin extras |
| `alpine:3.19` | ~7 MB | Minimalista, usa musl libc |
| `eclipse-temurin:21-jre` | ~267 MB | JRE completo sobre Ubuntu |
| `eclipse-temurin:21-jre-alpine` | ~155 MB | JRE sobre Alpine |
| `distroless/java21-debian12` | ~220 MB | Sin shell ni paquetes extra |

**Distroless** son imagenes de Google que no tienen shell, package manager ni
utilidades. Solo contienen la app y su runtime. Son mas seguras pero mas dificiles
de depurar (no puedes hacer `docker exec ... bash`).

```dockerfile
# Imagen distroless (maxima seguridad, minimo ataque)
FROM gcr.io/distroless/java21-debian12
COPY --from=build /app/target/*.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/app.jar"]
# No hay shell, no hay apt-get, no hay nada mas
```

### 5.5 Reducir capas

Cada instruccion RUN, COPY y ADD crea una capa. Menos capas = imagen mas pequena:

```dockerfile
# MAL: muchas capas
RUN apt-get update
RUN apt-get install -y curl
RUN apt-get install -y wget
RUN apt-get clean
RUN rm -rf /var/lib/apt/lists/*

# BIEN: una sola capa
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        curl \
        wget && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*
```

---

## 6. Docker BuildKit

### 6.1 Que es BuildKit

BuildKit es el motor de build moderno de Docker. Desde Docker 23.0 es el default.
Ofrece builds mas rapidos, cache mejorado y funcionalidades avanzadas.

```bash
# Activar BuildKit explicitamente (si tu version no lo trae por defecto)
export DOCKER_BUILDKIT=1
docker build -t mi-api .

# O con docker buildx
docker buildx build -t mi-api .
```

### 6.2 Cache de dependencias con --mount

BuildKit permite montar caches que persisten entre builds:

```dockerfile
# syntax=docker/dockerfile:1

FROM eclipse-temurin:21-jdk AS build
WORKDIR /app
COPY pom.xml mvnw ./
COPY .mvn .mvn

# Cache de Maven: persiste entre builds
RUN --mount=type=cache,target=/root/.m2/repository \
    ./mvnw dependency:go-offline

COPY src src
RUN --mount=type=cache,target=/root/.m2/repository \
    ./mvnw package -DskipTests
```

El directorio `/root/.m2/repository` se monta desde la cache de BuildKit. Las
dependencias descargadas persisten entre builds, ahorrando minutos en cada build.

### 6.3 Secrets en build time

BuildKit permite pasar secrets al build sin que queden en capas de la imagen:

```dockerfile
# En el Dockerfile
RUN --mount=type=secret,id=npm_token \
    NPM_TOKEN=$(cat /run/secrets/npm_token) && \
    npm install
# El secret NO queda en ninguna capa de la imagen

# Al construir
docker buildx build --secret id=npm_token,src=.npm_token -t mi-app .
```

### 6.4 Builds multi-plataforma

BuildKit permite construir imagenes para multiples arquitecturas (ARM, AMD64):

```bash
# Crear un builder multi-plataforma
docker buildx create --name multibuilder --use

# Construir para multiples plataformas
docker buildx build \
    --platform linux/amd64,linux/arm64 \         # AMD64 y ARM64
    -t mi-registro/mi-api:latest \
    --push \                                      # push directo al registro
    .
```

---

## 7. Patrones avanzados

### 7.1 Health check nativo en Dockerfile

```dockerfile
# Healthcheck directo en el Dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1
```

### 7.2 ENTRYPOINT vs CMD

```dockerfile
# ENTRYPOINT: comando fijo que siempre se ejecuta
# CMD: argumentos por defecto (se pueden sobreescribir)

# Patron recomendado:
ENTRYPOINT ["java", "-jar", "app.jar"]
CMD ["--spring.profiles.active=prod"]

# docker run mi-api                               -> java -jar app.jar --spring.profiles.active=prod
# docker run mi-api --spring.profiles.active=dev   -> java -jar app.jar --spring.profiles.active=dev
```

### 7.3 Script de entrypoint personalizado

Para logica compleja de arranque, usa un script:

```bash
#!/bin/sh
# entrypoint.sh

# Esperar a que la base de datos este disponible
echo "Esperando a la base de datos..."
while ! nc -z ${DB_HOST:-postgres} ${DB_PORT:-5432}; do
    sleep 1
done
echo "Base de datos disponible"

# Configurar JVM segun la memoria disponible
JAVA_OPTS="${JAVA_OPTS:--Xmx256m -Xms128m}"

# Arrancar la aplicacion
exec java ${JAVA_OPTS} -jar /app/app.jar "$@"
```

```dockerfile
COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

### 7.4 Labels para metadata

```dockerfile
# Labels OCI estandar
LABEL org.opencontainers.image.title="Mi API"
LABEL org.opencontainers.image.version="1.2.0"
LABEL org.opencontainers.image.created="2025-01-15T10:00:00Z"
LABEL org.opencontainers.image.source="https://github.com/mi-org/mi-api"
LABEL org.opencontainers.image.authors="equipo@empresa.com"
```

### 7.5 Dockerfile completo de produccion

```dockerfile
# syntax=docker/dockerfile:1

# --- Etapa 1: Build ---
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

COPY pom.xml mvnw ./
COPY .mvn .mvn
RUN --mount=type=cache,target=/root/.m2/repository \
    ./mvnw dependency:go-offline -B

COPY src src
RUN --mount=type=cache,target=/root/.m2/repository \
    ./mvnw package -DskipTests -B

# Extraer capas de Spring Boot para optimizar cache de Docker
RUN java -Djarmode=layertools -jar target/*.jar extract --destination extracted

# --- Etapa 2: Runtime ---
FROM eclipse-temurin:21-jre-alpine AS runtime

LABEL org.opencontainers.image.title="Mi API"
LABEL org.opencontainers.image.version="1.0.0"

RUN addgroup -S app && adduser -S app -G app
WORKDIR /app

# Copiar capas de Spring Boot (de menos a mas cambiante)
COPY --from=build /app/extracted/dependencies/ ./
COPY --from=build /app/extracted/spring-boot-loader/ ./
COPY --from=build /app/extracted/snapshot-dependencies/ ./
COPY --from=build /app/extracted/application/ ./

USER app
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-XX:+UseContainerSupport", "-XX:MaxRAMPercentage=75.0", "org.springframework.boot.loader.launch.JarLauncher"]
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| Multi-stage build | Multiples etapas FROM en un Dockerfile | Etapa de build + etapa de runtime ligera |
| `COPY --from` | Copiar archivos de una etapa anterior | `COPY --from=build /app/target/*.jar app.jar` |
| Bridge network | Red aislada con DNS entre contenedores | `docker network create mi-red` |
| Host network | Sin aislamiento, red del host | `docker run --network host` para maximo rendimiento |
| Overlay network | Red entre multiples hosts Docker | Para Docker Swarm o comunicacion multi-host |
| Healthcheck | Verificacion periodica de salud | `test`, `interval`, `timeout`, `retries`, `start_period` |
| Restart policy | Que hacer cuando un contenedor falla | `unless-stopped` para produccion |
| Resource limits | Limites de CPU y memoria | `deploy.resources.limits` en Compose |
| Trivy | Escaner de vulnerabilidades para imagenes | `trivy image mi-api:latest` |
| .dockerignore | Excluir archivos del build context | `.git`, `target/`, `node_modules/` |
| Usuario no-root | Proceso corre sin privilegios root | `RUN adduser` + `USER appuser` |
| BuildKit cache mount | Cache de dependencias entre builds | `RUN --mount=type=cache,target=/root/.m2` |
| Spring Boot layers | Extraer capas del JAR para mejor cache | `java -Djarmode=layertools -jar *.jar extract` |
| Distroless | Imagenes sin shell ni package manager | `gcr.io/distroless/java21-debian12` |
| Multi-plataforma | Imagenes para ARM y AMD64 | `docker buildx build --platform linux/amd64,linux/arm64` |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **Multi-stage build** | Crear un Dockerfile multi-stage para una app Spring Boot. Comparar tamanos de imagen con JDK, JRE y JRE-Alpine. Usar jlink para crear un JRE personalizado | Multi-stage, `COPY --from`, jlink, capas de Spring Boot |
| 02 | **Networking** | Crear una red bridge personalizada. Desplegar una app Spring Boot y PostgreSQL, verificar la comunicacion por DNS. Experimentar con redes host y none | `docker network create`, DNS interno, bridge vs host |
| 03 | **Compose produccion** | Escribir un docker-compose.yml completo con healthchecks, restart policies, limites de recursos, variables de entorno y volumenes persistentes | Compose, healthcheck, deploy.resources, depends_on, .env |
| 04 | **Security scanning** | Escanear una imagen con Trivy, interpretar los resultados, crear un .trivyignore, integrar escaneo en un script de CI | Trivy, CVEs, severidades, .trivyignore |

---

## Recursos

- [Docker Multi-stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Docker Networking Overview](https://docs.docker.com/network/)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [Spring Boot Docker Guide](https://spring.io/guides/topicals/spring-boot-docker)
- [Docker BuildKit](https://docs.docker.com/build/buildkit/)
- [Distroless Container Images](https://github.com/GoogleContainerTools/distroless)

---

> **Antes de continuar al siguiente nivel**: asegurate de que puedes construir una
> imagen multi-stage y que entiendes la diferencia entre bridge y host networking.
> En Kubernetes cada Pod tiene su propia IP, y entender networking aqui es la base
> para entender networking en K8s.
