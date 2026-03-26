# Nivel 17 — Proyecto Final

> Este es tu proyecto de graduacion. No es un ejercicio guiado: es un sistema real que integra todo lo que has aprendido en este path. Disena, implementa, despliega y monitorea una aplicacion Spring Boot + Kotlin con capacidades de IA en produccion. Construye algo real.

---

## Contenido

- [1. Vision General](#1-vision-general)
- [2. El Proyecto: Asistente Empresarial Inteligente](#2-el-proyecto-asistente-empresarial-inteligente)
  - [2.1 Descripcion](#21-descripcion)
  - [2.2 Arquitectura](#22-arquitectura)
  - [2.3 Stack tecnologico](#23-stack-tecnologico)
- [3. Requisitos Funcionales](#3-requisitos-funcionales)
  - [3.1 Modulo de RAG](#31-modulo-de-rag)
  - [3.2 Modulo de Agentes](#32-modulo-de-agentes)
  - [3.3 Modulo de Seguridad](#33-modulo-de-seguridad)
  - [3.4 API REST](#34-api-rest)
- [4. Requisitos No Funcionales](#4-requisitos-no-funcionales)
  - [4.1 Infraestructura](#41-infraestructura)
  - [4.2 Observabilidad](#42-observabilidad)
  - [4.3 CI/CD](#43-cicd)
  - [4.4 Testing](#44-testing)
- [5. Milestones](#5-milestones)
  - [5.1 Milestone 1: Arquitectura y setup](#51-milestone-1-arquitectura-y-setup)
  - [5.2 Milestone 2: Implementacion core](#52-milestone-2-implementacion-core)
  - [5.3 Milestone 3: Deploy y CI/CD](#53-milestone-3-deploy-y-cicd)
  - [5.4 Milestone 4: Observabilidad y hardening](#54-milestone-4-observabilidad-y-hardening)
- [6. Criterios de Evaluacion](#6-criterios-de-evaluacion)
- [7. Consejos Practicos](#7-consejos-practicos)
- [8. Ejercicios (Milestones)](#8-ejercicios-milestones)
- [9. Recursos](#9-recursos)

---

## 1. Vision General

Este proyecto integra los conocimientos de los 16 niveles anteriores:

| Nivel | Que aporta al proyecto |
|-------|----------------------|
| 07-08 | Spring Boot, IoC, configuracion |
| 09 | REST API con validacion |
| 10 | Persistencia con JPA y Flyway |
| 11 | Testing (JUnit, MockMvc, Testcontainers) |
| 12 | Spring Security, JWT |
| 13 | AOP para auditoria, cache, eventos |
| 14 | Orquestacion de IA, chains, structured outputs |
| 15 | Deploy en Kubernetes, model serving |
| 16 | Seguridad de IA, prompt injection, GDPR |

No puedes completar este proyecto sin haber entendido los niveles anteriores. Si algo no te suena, vuelve al nivel correspondiente.

---

## 2. El Proyecto: Asistente Empresarial Inteligente

### 2.1 Descripcion

Construiras un **asistente empresarial inteligente** que:

1. **Responde preguntas** sobre documentacion interna de la empresa (RAG)
2. **Ejecuta acciones** como buscar en la BD, crear tickets, consultar metricas (Agentes con tools)
3. **Se protege** contra prompt injection y abuso (Seguridad)
4. **Se despliega** en Kubernetes con CI/CD automatizado (Infra)
5. **Se monitorea** con metricas de negocio y de IA (Observabilidad)

Es el tipo de sistema que empresas reales estan construyendo hoy. Si lo completas, tienes un proyecto de portfolio que demuestra competencia en backend, IA e infraestructura.

### 2.2 Arquitectura

```
                                    ┌─────────────────────────┐
                                    │      Kubernetes          │
                                    │                          │
 ┌──────────┐    HTTPS    ┌─────────┴──────────┐              │
 │  Cliente  │ ─────────> │   Ingress / Gateway │              │
 │  (SPA)    │            └─────────┬──────────┘              │
 └──────────┘                       │                          │
                                    v                          │
                           ┌────────────────┐                  │
                           │   AI Service   │                  │
                           │  (Spring Boot) │                  │
                           │                │                  │
                           │ ┌────────────┐ │                  │
                           │ │ Security   │ │                  │
                           │ │ Pipeline   │ │                  │
                           │ └─────┬──────┘ │                  │
                           │       v        │                  │
                           │ ┌────────────┐ │    ┌───────────┐ │
                           │ │ Router /   │─┼──> │  Ollama   │ │
                           │ │ Orchestr.  │ │    │  (LLM)    │ │
                           │ └──┬────┬────┘ │    └───────────┘ │
                           │    │    │      │                  │
                           │    v    v      │                  │
                           │ ┌────┐┌────┐  │                  │
                           │ │RAG ││Agt.│  │                  │
                           │ └──┬─┘└──┬─┘  │                  │
                           │    │     │    │                  │
                           └────┼─────┼────┘                  │
                                │     │                        │
                     ┌──────────┘     └──────────┐            │
                     v                           v            │
              ┌─────────────┐            ┌─────────────┐      │
              │  PostgreSQL │            │    Redis     │      │
              │  + pgvector │            │   (Cache)    │      │
              └─────────────┘            └─────────────┘      │
                                                               │
              ┌─────────────┐            ┌─────────────┐      │
              │ Prometheus  │            │   Grafana    │      │
              └─────────────┘            └─────────────┘      │
                                    └─────────────────────────┘
```

### 2.3 Stack tecnologico

| Componente | Tecnologia |
|-----------|------------|
| **Lenguaje** | Kotlin |
| **Framework** | Spring Boot 3.x |
| **IA** | Spring AI + LangChain4j |
| **LLM** | Ollama (local) o OpenAI (API) |
| **Base de datos** | PostgreSQL + pgvector |
| **Cache** | Redis + cache semantico |
| **Seguridad** | Spring Security + JWT |
| **Contenedores** | Docker + Docker Compose |
| **Orquestacion** | Kubernetes (Minikube o kind) |
| **CI/CD** | GitHub Actions |
| **Metricas** | Micrometer + Prometheus |
| **Dashboards** | Grafana |
| **Testing** | JUnit 5 + Testcontainers + MockMvc |
| **Migraciones** | Flyway |
| **Build** | Maven |

---

## 3. Requisitos Funcionales

### 3.1 Modulo de RAG

El sistema debe poder:

- Ingestar documentos (PDF, Markdown, texto plano) via API REST
- Generar embeddings y almacenarlos en pgvector
- Responder preguntas buscando en los documentos relevantes
- Citar las fuentes usadas en cada respuesta
- Soportar multiples "colecciones" de documentos (ej. RRHH, Tecnico, Legal)

```kotlin
// Ejemplo de endpoint esperado
POST /api/documents/upload
Content-Type: multipart/form-data
collection: "rrhh"
file: politica-vacaciones.pdf

POST /api/chat
{
    "message": "Cuantos dias de vacaciones tengo?",
    "collection": "rrhh"
}

// Respuesta esperada
{
    "message": "Segun la politica de vacaciones, cada empleado tiene 22 dias laborables...",
    "sources": [
        {"document": "politica-vacaciones.pdf", "page": 3, "similarity": 0.94}
    ],
    "metadata": {
        "generatedByAi": true,
        "model": "llama3.1:8b",
        "tokensUsed": 450
    }
}
```

### 3.2 Modulo de Agentes

El sistema debe tener agentes con herramientas:

- **Agente de consultas**: buscar empleados, proyectos, informacion en la BD
- **Agente de tickets**: crear tickets de soporte, consultar estado
- **Orquestador**: clasificar la intencion del usuario y delegar al agente correcto

```kotlin
// Ejemplo de interaccion con agente
POST /api/chat
{
    "message": "Crea un ticket de soporte: mi laptop no enciende"
}

// El orquestador clasifica -> delega al agente de tickets -> crea el ticket
{
    "message": "He creado el ticket #1234 con prioridad alta. Un tecnico te contactara en las proximas 2 horas.",
    "actions": [
        {"type": "TICKET_CREATED", "ticketId": 1234}
    ]
}
```

### 3.3 Modulo de Seguridad

- Defensa multicapa contra prompt injection
- Deteccion y redaccion de PII en salidas
- Rate limiting por tokens y coste
- Audit logging de todas las interacciones con el LLM
- Autenticacion JWT para todos los endpoints

### 3.4 API REST

Endpoints minimos:

| Metodo | Endpoint | Descripcion |
|--------|----------|-------------|
| POST | `/api/auth/login` | Autenticacion, devuelve JWT |
| POST | `/api/chat` | Chat con el asistente (RAG + agentes) |
| GET | `/api/chat/history` | Historial de conversaciones del usuario |
| POST | `/api/documents/upload` | Subir documento para RAG |
| GET | `/api/documents` | Listar documentos indexados |
| DELETE | `/api/documents/{id}` | Eliminar documento |
| GET | `/api/tickets` | Listar tickets del usuario |
| GET | `/api/admin/audit` | Logs de auditoria (solo admin) |
| GET | `/api/admin/metrics` | Metricas de uso de IA (solo admin) |
| DELETE | `/api/gdpr/my-data` | Derecho al olvido (GDPR) |

---

## 4. Requisitos No Funcionales

### 4.1 Infraestructura

- **Docker Compose** para desarrollo local (app + PostgreSQL + Redis + Ollama + Prometheus + Grafana)
- **Kubernetes manifiestos** para deploy en produccion (Deployments, Services, Secrets, ConfigMaps, PVC)
- **Dockerfile** multi-stage optimizado
- **Migraciones** de BD con Flyway (incluyendo extension pgvector)

### 4.2 Observabilidad

- Metricas con Micrometer exportadas a Prometheus:
  - Tokens consumidos (input/output) por modelo
  - Latencia de requests al LLM (p50, p95, p99)
  - Coste estimado acumulado
  - Cache hit ratio
  - Rate limit rejections
- Dashboard de Grafana con al menos 6 paneles
- Health checks (liveness, readiness) que verifican el LLM
- Logging estructurado con traceId en cada request

### 4.3 CI/CD

Pipeline de GitHub Actions que:

1. Ejecuta tests unitarios
2. Ejecuta tests de integracion (con Testcontainers)
3. Construye imagen Docker
4. Publica la imagen en un registry
5. (Opcional) Deploy automatico a Kubernetes

```yaml
# Ejemplo de estructura esperada del workflow
name: CI/CD
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Unit tests
        run: mvn test
      - name: Integration tests
        run: mvn verify -Pintegration

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Build Docker image
        run: docker build -t ai-assistant .
      - name: Push to registry
        run: docker push registry/ai-assistant:${{ github.sha }}
```

### 4.4 Testing

Cobertura minima:

| Tipo | Que testear | Cobertura |
|------|-------------|-----------|
| **Unitarios** | Servicios, logica de negocio, security pipeline | > 70% |
| **Integracion** | RAG con pgvector real, endpoints REST | > 50% |
| **AI-specific** | Snapshot tests, LLM mock, evaluacion de calidad | Al menos 5 test cases |
| **Security** | Prompt injection, rate limiting, auth | Al menos 10 test cases |

---

## 5. Milestones

### 5.1 Milestone 1: Arquitectura y setup

**Duracion estimada: 1 semana**

- [ ] Crear proyecto Spring Boot con Kotlin y Maven
- [ ] Configurar dependencias (Spring AI, LangChain4j, Spring Security, Flyway)
- [ ] Definir estructura de paquetes (hexagonal o por capas)
- [ ] Docker Compose con PostgreSQL + pgvector + Redis + Ollama
- [ ] Migraciones iniciales de Flyway
- [ ] Documentar decisiones de arquitectura en un ADR (Architecture Decision Record)

Entregable: proyecto arranca, docker-compose up funciona, migraciones se ejecutan.

### 5.2 Milestone 2: Implementacion core

**Duracion estimada: 2-3 semanas**

- [ ] API REST con autenticacion JWT
- [ ] Ingesta de documentos y generacion de embeddings
- [ ] Pipeline de RAG funcional (busqueda + generacion)
- [ ] Agentes con tools (al menos 2 herramientas funcionales)
- [ ] Orquestador que rutea entre RAG y agentes
- [ ] Tests unitarios y de integracion

Entregable: puedes chatear con el asistente via API, hace RAG y ejecuta acciones.

### 5.3 Milestone 3: Deploy y CI/CD

**Duracion estimada: 1 semana**

- [ ] Dockerfile multi-stage optimizado
- [ ] Manifiestos de Kubernetes (Deployment, Service, Secret, ConfigMap, PVC)
- [ ] La aplicacion funciona en Minikube o kind
- [ ] Pipeline de CI/CD en GitHub Actions (test + build + push)
- [ ] Secrets gestionados correctamente (no en el repo)

Entregable: push a main -> tests -> build -> imagen publicada -> deploy funcional.

### 5.4 Milestone 4: Observabilidad y hardening

**Duracion estimada: 1 semana**

- [ ] Security pipeline: filtros de entrada/salida, deteccion de prompt injection
- [ ] Rate limiting por tokens
- [ ] Audit logging de todas las interacciones
- [ ] Metricas de IA en Prometheus
- [ ] Dashboard de Grafana (al menos 6 paneles)
- [ ] Health checks que verifican el LLM
- [ ] GDPR: endpoint de deletion

Entregable: sistema seguro, monitoreable y compliant.

---

## 6. Criterios de Evaluacion

| Criterio | Peso | Excelente | Aceptable | Insuficiente |
|----------|:----:|-----------|-----------|-------------|
| **Funcionalidad** | 30% | RAG + agentes + orquestacion completa | RAG basico funcional | No funciona o solo chat simple |
| **Seguridad** | 20% | Defensa multicapa, rate limiting, audit, GDPR | Autenticacion + filtros basicos | Sin seguridad de IA |
| **Infraestructura** | 20% | K8s + CI/CD + Docker optimizado | Docker Compose funcional | Solo ejecucion local |
| **Observabilidad** | 15% | Metricas IA + dashboard + alertas + tracing | Metricas basicas + health checks | Sin metricas |
| **Testing** | 10% | Unit + integration + AI-specific tests | Tests unitarios basicos | Sin tests |
| **Calidad de codigo** | 5% | Codigo limpio, bien estructurado, documentado | Codigo funcional | Codigo desordenado |

**Nota minima para aprobar**: 60% (Aceptable en todos los criterios).

**Nota para destacar**: 85%+ (Excelente en al menos 3 criterios).

---

## 7. Consejos Practicos

1. **Empieza por lo minimo**: haz que el chat basico funcione primero. Despues anade RAG, despues agentes, despues seguridad.

2. **Docker Compose es tu amigo**: no intentes Kubernetes hasta que todo funcione en Docker Compose. K8s solo anade complejidad, no funcionalidad.

3. **Usa Ollama**: no dependas de APIs externas. Con Ollama puedes trabajar offline, es gratis y suficiente para el proyecto. Usa `llama3.1:8b` para chat y `nomic-embed-text` para embeddings.

4. **No subestimes el testing**: los tests de IA son diferentes. No puedes hacer `assertEquals`. Usa mocks del LLM para tests unitarios y snapshot testing para tests de integracion.

5. **El security pipeline es critico**: no es un "nice to have". Un sistema de IA sin proteccion contra prompt injection no esta listo para produccion.

6. **Monitorea desde el principio**: no dejes las metricas para el final. Instrumenta con Micrometer desde el primer endpoint.

7. **Documenta tus decisiones**: un breve ADR (Architecture Decision Record) por cada decision importante te ahorrara preguntas despues.

8. **Git discipline**: commits atomicos con mensajes descriptivos. Feature branches. Pull requests. Esto es parte de la evaluacion.

---

## 8. Ejercicios (Milestones)

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | [Arquitectura](ejercicio-01-arquitectura/) | Setup del proyecto, Docker Compose, migraciones Flyway, estructura de paquetes, ADR | Spring Boot init, Docker Compose, Flyway, pgvector |
| 02 | [Implementacion](ejercicio-02-implementacion/) | RAG pipeline, agentes con tools, orquestador, API REST con JWT, tests | Spring AI, LangChain4j, RAG, `@Tool`, Spring Security |
| 03 | [Deploy](ejercicio-03-deploy/) | Dockerfile, manifiestos K8s, CI/CD con GitHub Actions | Docker multi-stage, K8s, GitHub Actions |
| 04 | [Observabilidad](ejercicio-04-observabilidad/) | Security pipeline, rate limiting, audit logging, metricas Prometheus, dashboard Grafana | Micrometer, Prometheus, Grafana, `AiAuditAspect` |

Cada milestone es un ejercicio. Completalos en orden. Al terminar los cuatro, tendras un sistema completo de produccion.

---

## 9. Recursos

- [Spring AI Reference](https://docs.spring.io/spring-ai/reference/)
- [LangChain4j Documentation](https://docs.langchain4j.dev/)
- [Ollama Documentation](https://ollama.com/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Spring Security Reference](https://docs.spring.io/spring-security/reference/)
- [Flyway Documentation](https://documentation.red-gate.com/fd)
- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Testcontainers for Java](https://testcontainers.com/guides/getting-started-with-testcontainers-for-java/)
