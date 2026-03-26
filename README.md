# Backend + AI + Infra Learning Path

> De Spring Boot a desplegar aplicaciones con inteligencia artificial en produccion.
> Kubernetes, Terraform, observabilidad, Spring AI, RAG, agentes: el stack completo
> del backend engineer que las empresas buscan en 2026.

> **Prerequisito**: haber completado (o dominar) el [Spring & Backend Learning Path](https://github.com/salvy9978/spring-path).
> Este path asume que ya sabes Java/Kotlin, Spring Boot, REST APIs, JPA, Security, Testing y Docker basico.

---

## Como usar este repositorio

```
nivel-XX-nombre/
├── README.md          # Teoria completa del nivel
├── ejercicio-01/      # Ejercicio practico con instrucciones
├── ejercicio-02/
└── ...
```

1. Lee el `README.md` de cada nivel (teoria).
2. Haz los ejercicios en orden.
3. No saltes niveles salvo que domines el tema.
4. Cada ejercicio tiene su propio `README.md` con instrucciones.

---

## Roadmap completo

### Bloque 1 — Cloud e Infraestructura

| Nivel | Tema | Que aprendes |
|-------|------|-------------|
| 00 | **Cloud: fundamentos** | IaaS/PaaS/SaaS, AWS vs GCP, regiones, VPCs, IAM, CLI setup |
| 01 | **Docker avanzado** | Multi-stage builds, networking, compose produccion, security scanning, optimizacion de imagenes |
| 02 | **Kubernetes: fundamentos** | Arquitectura K8s, pods, deployments, services, namespaces, kubectl, desplegar una app Spring |
| 03 | **Kubernetes: avanzado** | ConfigMaps, Secrets, HPA, probes, PersistentVolumes, Ingress, RBAC |
| 04 | **Helm y Kustomize** | Charts, values/templates, Kustomize overlays, empaquetar apps Spring Boot |

### Bloque 2 — Platform Engineering

| Nivel | Tema | Que aprendes |
|-------|------|-------------|
| 05 | **CI/CD avanzado** | GitHub Actions (matrix, reusable workflows), build+push Docker, estrategias de deploy, pipelines completos |
| 06 | **Terraform** | Infraestructura como codigo, HCL, providers, state remoto, modulos, variables, outputs |
| 07 | **Observabilidad** | Prometheus, Grafana, Loki, OpenTelemetry, Spring Boot Actuator + Micrometer, tracing distribuido |
| 08 | **GitOps** | ArgoCD, sync strategies, rollbacks, multi-environment, Sealed Secrets |

### Bloque 3 — Inteligencia Artificial

| Nivel | Tema | Que aprendes |
|-------|------|-------------|
| 09 | **LLMs: fundamentos** | Transformers, tokens, APIs (OpenAI/Anthropic/Google), prompt engineering, embeddings, costes |
| 10 | **Spring AI** | ChatClient, prompt templates, output parsers, streaming, memoria conversacional, multi-modelo |
| 11 | **Bases de datos vectoriales** | pgvector, similarity search (cosine/euclidean), Spring AI VectorStore, indexing (HNSW/IVF) |
| 12 | **RAG** | Retrieval Augmented Generation: document loading, chunking, embedding, retrieval pipeline, evaluacion |
| 13 | **Agentes y Tool Calling** | Function calling, arquitectura ReAct, multi-tool agents, memoria de agentes, guardrails |

### Bloque 4 — Produccion y Proyecto Final

| Nivel | Tema | Que aprendes |
|-------|------|-------------|
| 14 | **Orquestacion avanzada** | LangChain4j, chains/workflows complejos, structured outputs, testing de sistemas AI |
| 15 | **Despliegue de apps AI** | Spring AI en Kubernetes, model serving (Ollama/vLLM), autoscaling para AI, gestion de costes |
| 16 | **Seguridad y produccion** | Prompt injection, filtros de entrada/salida, rate limiting, audit logging, compliance |
| 17 | **Proyecto final** | Aplicacion completa: Spring Boot + AI + K8s + CI/CD + observabilidad, desplegada y monitorizada |

---

## Filosofia del path

### No es un curso de AI

Este path NO te convierte en data scientist ni en ML engineer. Te convierte en el **backend engineer que sabe integrar AI en productos reales**. La diferencia:

| Data Scientist | Backend + AI Engineer (tu) |
|---------------|---------------------------|
| Entrena modelos | Consume modelos via API |
| Python + notebooks | Kotlin + Spring Boot |
| Experimenta en local | Despliega en produccion |
| Optimiza metricas de modelo | Optimiza latencia, coste y disponibilidad |
| Publica papers | Publica features |

### No es un curso de DevOps

Tampoco te convierte en SRE. Pero un backend engineer que sabe desplegar su propio codigo, monitorizar sus servicios y automatizar su infraestructura vale el doble que uno que tira PRs "al vacio" esperando que alguien los despliegue.

### El perfil que se busca

El mercado en 2026 necesita ingenieros que sepan:

1. **Construir APIs** (ya lo sabes de Spring Path)
2. **Desplegarlas** en Kubernetes con CI/CD automatizado
3. **Monitorizarlas** con metricas, logs y traces
4. **Integrar AI** de forma practica: RAG, agentes, tool calling
5. **Produccionalizarlo todo**: seguridad, costes, escalado

Ese es el perfil. Este path te da cada pieza.

---

## Estructura de cada nivel

```
nivel-XX-nombre/
├── README.md              # Teoria: conceptos, explicaciones, diagramas
├── ejercicio-01-nombre/   # Ejercicio guiado
│   ├── README.md          # Enunciado + criterios de exito
│   ├── pom.xml            # (cuando aplique)
│   ├── Dockerfile         # (cuando aplique)
│   ├── docker-compose.yml # (cuando aplique)
│   ├── k8s/               # (manifiestos Kubernetes cuando aplique)
│   └── src/               # (codigo cuando aplique)
├── ejercicio-02-nombre/
└── ...
```

> **Regla**: intenta el ejercicio antes de buscar la solucion.
> Si te atascas mas de 30 min, mira pistas. Si 1h, busca ayuda.

---

## Progreso

### Bloque 1 — Cloud e Infraestructura
- [ ] Nivel 00 — Cloud: fundamentos
- [ ] Nivel 01 — Docker avanzado
- [ ] Nivel 02 — Kubernetes: fundamentos
- [ ] Nivel 03 — Kubernetes: avanzado
- [ ] Nivel 04 — Helm y Kustomize

### Bloque 2 — Platform Engineering
- [ ] Nivel 05 — CI/CD avanzado
- [ ] Nivel 06 — Terraform
- [ ] Nivel 07 — Observabilidad
- [ ] Nivel 08 — GitOps

### Bloque 3 — Inteligencia Artificial
- [ ] Nivel 09 — LLMs: fundamentos
- [ ] Nivel 10 — Spring AI
- [ ] Nivel 11 — Bases de datos vectoriales
- [ ] Nivel 12 — RAG
- [ ] Nivel 13 — Agentes y Tool Calling

### Bloque 4 — Produccion y Proyecto Final
- [ ] Nivel 14 — Orquestacion avanzada
- [ ] Nivel 15 — Despliegue de apps AI
- [ ] Nivel 16 — Seguridad y produccion
- [ ] Nivel 17 — Proyecto final

---

## Requisitos previos

- **Spring Path completado** (o conocimiento equivalente de Spring Boot + Kotlin)
- **JDK 21+** instalado
- **Docker Desktop** (o Docker Engine + Docker Compose)
- **kubectl** instalado
- **Minikube** o **Kind** (Kubernetes local, desde nivel 02)
- **Terraform** (desde nivel 06)
- **Cuenta AWS o GCP** con free tier (desde nivel 00)
- **API key de OpenAI o Anthropic** (desde nivel 09)
- **Helm** (desde nivel 04)
- **Git** configurado

> No necesitas todo desde el dia 1. Cada nivel indica sus requisitos especificos.

---

## Herramientas por nivel

| Nivel | Herramientas nuevas |
|-------|-------------------|
| 00 | `aws` CLI o `gcloud` CLI |
| 01 | `docker`, `docker compose`, Trivy |
| 02-03 | `kubectl`, Minikube/Kind |
| 04 | `helm`, `kustomize` |
| 05 | GitHub Actions (en el repo) |
| 06 | `terraform` |
| 07 | Prometheus, Grafana, Loki (via Docker Compose) |
| 08 | ArgoCD (en Kubernetes) |
| 09 | API keys (OpenAI/Anthropic) |
| 10-14 | Spring AI, LangChain4j (dependencias Maven) |
| 11 | PostgreSQL + pgvector (via Docker) |
| 15 | Ollama (model serving local) |
