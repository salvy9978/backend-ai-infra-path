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
├── README.md              # Teoria completa del nivel (Kotlin/Spring)
├── README-python.md       # Teoria track Python (cuando aplique)
├── ejercicio-01-nombre/   # Ejercicio Kotlin/Spring
├── ejercicio-05-python-*/ # Ejercicio equivalente en Python
└── ...
```

1. Lee el `README.md` de cada nivel (teoria).
2. Haz los ejercicios en orden (Kotlin o Python, o ambos).
3. No saltes niveles salvo que domines el tema.
4. Los niveles 09, 11-14 tienen **track Python** con teoria y ejercicios dedicados.

---

## Dos tracks, un objetivo

Los niveles de AI (09-14) ofrecen dos caminos paralelos:

| | Track Kotlin/Spring | Track Python |
|---|---|---|
| **Framework** | Spring AI, LangChain4j | LangChain, LlamaIndex, OpenAI SDK |
| **Servidor** | Spring Boot | FastAPI |
| **Ejercicios** | 01-04 | 05-08 |
| **Teoria** | README.md | README-python.md |
| **Para quien** | Backend engineers JVM | Devs del ecosistema Python |

Puedes hacer uno, otro, o ambos. Los conceptos son los mismos; cambia la implementacion.

---

## Indice completo

### Bloque 1 — Cloud e Infraestructura

<details>
<summary><strong>Nivel 00 — Cloud: fundamentos</strong></summary>

- [Teoria](nivel-00-cloud-fundamentos/README.md)
- Ejercicios:
  - [01 — CLI setup](nivel-00-cloud-fundamentos/ejercicio-01-cli-setup/)
  - [02 — IAM basico](nivel-00-cloud-fundamentos/ejercicio-02-iam-basico/)
  - [03 — VPC networking](nivel-00-cloud-fundamentos/ejercicio-03-vpc-networking/)
</details>

<details>
<summary><strong>Nivel 01 — Docker avanzado</strong></summary>

- [Teoria](nivel-01-docker-avanzado/README.md)
- Ejercicios:
  - [01 — Multi-stage builds](nivel-01-docker-avanzado/ejercicio-01-multi-stage/)
  - [02 — Networking](nivel-01-docker-avanzado/ejercicio-02-networking/)
  - [03 — Compose produccion](nivel-01-docker-avanzado/ejercicio-03-compose-produccion/)
  - [04 — Security scanning](nivel-01-docker-avanzado/ejercicio-04-security-scanning/)
</details>

<details>
<summary><strong>Nivel 02 — Kubernetes: fundamentos</strong></summary>

- [Teoria](nivel-02-kubernetes-fundamentos/README.md)
- Ejercicios:
  - [01 — Pods y Deployments](nivel-02-kubernetes-fundamentos/ejercicio-01-pods-deployments/)
  - [02 — Services](nivel-02-kubernetes-fundamentos/ejercicio-02-services/)
  - [03 — Namespaces](nivel-02-kubernetes-fundamentos/ejercicio-03-namespaces/)
  - [04 — Deploy Spring app](nivel-02-kubernetes-fundamentos/ejercicio-04-deploy-spring-app/)
</details>

<details>
<summary><strong>Nivel 03 — Kubernetes: avanzado</strong></summary>

- [Teoria](nivel-03-kubernetes-avanzado/README.md)
- Ejercicios:
  - [01 — ConfigMaps y Secrets](nivel-03-kubernetes-avanzado/ejercicio-01-configmaps-secrets/)
  - [02 — HPA scaling](nivel-03-kubernetes-avanzado/ejercicio-02-hpa-scaling/)
  - [03 — Probes y lifecycle](nivel-03-kubernetes-avanzado/ejercicio-03-probes-lifecycle/)
  - [04 — Ingress con TLS](nivel-03-kubernetes-avanzado/ejercicio-04-ingress-tls/)
</details>

<details>
<summary><strong>Nivel 04 — Helm y Kustomize</strong></summary>

- [Teoria](nivel-04-helm-kustomize/README.md)
- Ejercicios:
  - [01 — Helm chart](nivel-04-helm-kustomize/ejercicio-01-helm-chart/)
  - [02 — Values y templates](nivel-04-helm-kustomize/ejercicio-02-values-templates/)
  - [03 — Kustomize overlays](nivel-04-helm-kustomize/ejercicio-03-kustomize-overlays/)
  - [04 — Chart Spring Boot](nivel-04-helm-kustomize/ejercicio-04-chart-spring-boot/)
</details>

### Bloque 2 — Platform Engineering

<details>
<summary><strong>Nivel 05 — CI/CD avanzado</strong></summary>

- [Teoria](nivel-05-cicd-avanzado/README.md)
- Ejercicios:
  - [01 — GitHub Actions matrix](nivel-05-cicd-avanzado/ejercicio-01-github-actions-matrix/)
  - [02 — Docker build y push](nivel-05-cicd-avanzado/ejercicio-02-docker-build-push/)
  - [03 — Deploy strategies](nivel-05-cicd-avanzado/ejercicio-03-deploy-strategies/)
  - [04 — Pipeline completo](nivel-05-cicd-avanzado/ejercicio-04-pipeline-completo/)
</details>

<details>
<summary><strong>Nivel 06 — Terraform</strong></summary>

- [Teoria](nivel-06-terraform/README.md)
- Ejercicios:
  - [01 — Primeros recursos](nivel-06-terraform/ejercicio-01-primeros-recursos/)
  - [02 — Modulos](nivel-06-terraform/ejercicio-02-modulos/)
  - [03 — State remoto](nivel-06-terraform/ejercicio-03-state-remoto/)
  - [04 — Infra completa](nivel-06-terraform/ejercicio-04-infra-completa/)
</details>

<details>
<summary><strong>Nivel 07 — Observabilidad</strong></summary>

- [Teoria](nivel-07-observabilidad/README.md)
- Ejercicios:
  - [01 — Prometheus y Grafana](nivel-07-observabilidad/ejercicio-01-prometheus-grafana/)
  - [02 — Spring Actuator metrics](nivel-07-observabilidad/ejercicio-02-spring-actuator-metrics/)
  - [03 — Logging con Loki](nivel-07-observabilidad/ejercicio-03-logging-loki/)
  - [04 — Tracing distribuido](nivel-07-observabilidad/ejercicio-04-tracing-distribuido/)
</details>

<details>
<summary><strong>Nivel 08 — GitOps</strong></summary>

- [Teoria](nivel-08-gitops/README.md)
- Ejercicios:
  - [01 — ArgoCD setup](nivel-08-gitops/ejercicio-01-argocd-setup/)
  - [02 — App deploy](nivel-08-gitops/ejercicio-02-app-deploy/)
  - [03 — Multi-environment](nivel-08-gitops/ejercicio-03-multi-env/)
  - [04 — Rollback](nivel-08-gitops/ejercicio-04-rollback/)
</details>

### Bloque 3 — Inteligencia Artificial

<details>
<summary><strong>Nivel 09 — LLMs: fundamentos</strong> (Kotlin + Python)</summary>

- [Teoria Kotlin/Spring](nivel-09-llm-fundamentos/README.md)
- [Teoria Python](nivel-09-llm-fundamentos/README-python.md)
- Ejercicios Kotlin:
  - [01 — API OpenAI](nivel-09-llm-fundamentos/ejercicio-01-api-openai/)
  - [02 — Prompt engineering](nivel-09-llm-fundamentos/ejercicio-02-prompt-engineering/)
  - [03 — Embeddings](nivel-09-llm-fundamentos/ejercicio-03-embeddings/)
  - [04 — Costes y optimizacion](nivel-09-llm-fundamentos/ejercicio-04-costes-optimizacion/)
- Ejercicios Python:
  - [05 — OpenAI SDK en Python](nivel-09-llm-fundamentos/ejercicio-05-python-openai-sdk/)
  - [06 — Prompt engineering en Python](nivel-09-llm-fundamentos/ejercicio-06-python-prompt-engineering/)
  - [07 — Embeddings en Python](nivel-09-llm-fundamentos/ejercicio-07-python-embeddings/)
  - [08 — Costes y streaming en Python](nivel-09-llm-fundamentos/ejercicio-08-python-costes-streaming/)
</details>

<details>
<summary><strong>Nivel 10 — Spring AI</strong> (solo Kotlin)</summary>

- [Teoria](nivel-10-spring-ai/README.md)
- Ejercicios:
  - [01 — ChatClient](nivel-10-spring-ai/ejercicio-01-chat-client/)
  - [02 — Prompt templates](nivel-10-spring-ai/ejercicio-02-prompt-templates/)
  - [03 — Output parsers](nivel-10-spring-ai/ejercicio-03-output-parsers/)
  - [04 — Streaming y memoria](nivel-10-spring-ai/ejercicio-04-streaming-memory/)
</details>

<details>
<summary><strong>Nivel 11 — Bases de datos vectoriales</strong> (Kotlin + Python)</summary>

- [Teoria Kotlin/Spring](nivel-11-vector-databases/README.md)
- [Teoria Python](nivel-11-vector-databases/README-python.md)
- Ejercicios Kotlin:
  - [01 — pgvector](nivel-11-vector-databases/ejercicio-01-pgvector/)
  - [02 — Similarity search](nivel-11-vector-databases/ejercicio-02-similarity-search/)
  - [03 — Spring AI VectorStore](nivel-11-vector-databases/ejercicio-03-spring-ai-vectorstore/)
  - [04 — Indexing strategies](nivel-11-vector-databases/ejercicio-04-indexing-strategies/)
- Ejercicios Python:
  - [05 — pgvector en Python](nivel-11-vector-databases/ejercicio-05-python-pgvector/)
  - [06 — Qdrant en Python](nivel-11-vector-databases/ejercicio-06-python-qdrant/)
  - [07 — Similarity search en Python](nivel-11-vector-databases/ejercicio-07-python-similarity-search/)
  - [08 — Hybrid search en Python](nivel-11-vector-databases/ejercicio-08-python-hybrid-search/)
</details>

<details>
<summary><strong>Nivel 12 — RAG</strong> (Kotlin + Python)</summary>

- [Teoria Kotlin/Spring](nivel-12-rag/README.md)
- [Teoria Python](nivel-12-rag/README-python.md)
- Ejercicios Kotlin:
  - [01 — Document loading](nivel-12-rag/ejercicio-01-document-loading/)
  - [02 — Chunking y embedding](nivel-12-rag/ejercicio-02-chunking-embedding/)
  - [03 — Retrieval pipeline](nivel-12-rag/ejercicio-03-retrieval-pipeline/)
  - [04 — RAG evaluacion](nivel-12-rag/ejercicio-04-rag-evaluacion/)
- Ejercicios Python:
  - [05 — LangChain RAG](nivel-12-rag/ejercicio-05-python-langchain-rag/)
  - [06 — LlamaIndex](nivel-12-rag/ejercicio-06-python-llamaindex/)
  - [07 — Chunking avanzado](nivel-12-rag/ejercicio-07-python-chunking-avanzado/)
  - [08 — Evaluacion con RAGAS](nivel-12-rag/ejercicio-08-python-evaluacion-ragas/)
</details>

<details>
<summary><strong>Nivel 13 — Agentes y Tool Calling</strong> (Kotlin + Python)</summary>

- [Teoria Kotlin/Spring](nivel-13-agentes-tools/README.md)
- [Teoria Python](nivel-13-agentes-tools/README-python.md)
- Ejercicios Kotlin:
  - [01 — Function calling](nivel-13-agentes-tools/ejercicio-01-function-calling/)
  - [02 — ReAct agent](nivel-13-agentes-tools/ejercicio-02-react-agent/)
  - [03 — Multi-tool](nivel-13-agentes-tools/ejercicio-03-multi-tool/)
  - [04 — Agent memory](nivel-13-agentes-tools/ejercicio-04-agent-memory/)
- Ejercicios Python:
  - [05 — LangChain tools](nivel-13-agentes-tools/ejercicio-05-python-langchain-tools/)
  - [06 — ReAct agent en Python](nivel-13-agentes-tools/ejercicio-06-python-react-agent/)
  - [07 — CrewAI](nivel-13-agentes-tools/ejercicio-07-python-crewai/)
  - [08 — Agent memory en Python](nivel-13-agentes-tools/ejercicio-08-python-agent-memory/)
</details>

### Bloque 4 — Produccion y Proyecto Final

<details>
<summary><strong>Nivel 14 — Orquestacion avanzada</strong> (Kotlin + Python)</summary>

- [Teoria Kotlin/Spring](nivel-14-orquestacion-avanzada/README.md)
- [Teoria Python](nivel-14-orquestacion-avanzada/README-python.md)
- Ejercicios Kotlin:
  - [01 — LangChain4j](nivel-14-orquestacion-avanzada/ejercicio-01-langchain4j/)
  - [02 — Chains y workflows](nivel-14-orquestacion-avanzada/ejercicio-02-chains-workflows/)
  - [03 — Structured outputs](nivel-14-orquestacion-avanzada/ejercicio-03-structured-outputs/)
  - [04 — Testing AI](nivel-14-orquestacion-avanzada/ejercicio-04-testing-ai/)
- Ejercicios Python:
  - [05 — LangChain chains](nivel-14-orquestacion-avanzada/ejercicio-05-python-langchain-chains/)
  - [06 — LangGraph](nivel-14-orquestacion-avanzada/ejercicio-06-python-langgraph/)
  - [07 — Structured outputs en Python](nivel-14-orquestacion-avanzada/ejercicio-07-python-structured-outputs/)
  - [08 — Eval y testing en Python](nivel-14-orquestacion-avanzada/ejercicio-08-python-eval-testing/)
</details>

<details>
<summary><strong>Nivel 15 — Despliegue de apps AI</strong></summary>

- [Teoria](nivel-15-deploy-ai/README.md)
- Ejercicios:
  - [01 — Spring AI en K8s](nivel-15-deploy-ai/ejercicio-01-spring-ai-k8s/)
  - [02 — Model serving](nivel-15-deploy-ai/ejercicio-02-model-serving/)
  - [03 — Autoscaling AI](nivel-15-deploy-ai/ejercicio-03-autoscaling-ai/)
  - [04 — Cost management](nivel-15-deploy-ai/ejercicio-04-cost-management/)
</details>

<details>
<summary><strong>Nivel 16 — Seguridad y produccion</strong></summary>

- [Teoria](nivel-16-seguridad-produccion/README.md)
- Ejercicios:
  - [01 — Prompt injection](nivel-16-seguridad-produccion/ejercicio-01-prompt-injection/)
  - [02 — Filtros entrada/salida](nivel-16-seguridad-produccion/ejercicio-02-input-output-filters/)
  - [03 — Rate limiting](nivel-16-seguridad-produccion/ejercicio-03-rate-limiting/)
  - [04 — Audit y compliance](nivel-16-seguridad-produccion/ejercicio-04-audit-compliance/)
</details>

<details>
<summary><strong>Nivel 17 — Proyecto final</strong></summary>

- [Teoria](nivel-17-proyecto-final/README.md)
- Hitos:
  - [01 — Arquitectura](nivel-17-proyecto-final/ejercicio-01-arquitectura/)
  - [02 — Implementacion](nivel-17-proyecto-final/ejercicio-02-implementacion/)
  - [03 — Deploy](nivel-17-proyecto-final/ejercicio-03-deploy/)
  - [04 — Observabilidad](nivel-17-proyecto-final/ejercicio-04-observabilidad/)
</details>

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
- [ ] Nivel 09 — LLMs: fundamentos (Kotlin)
- [ ] Nivel 09 — LLMs: fundamentos (Python)
- [ ] Nivel 10 — Spring AI
- [ ] Nivel 11 — Bases de datos vectoriales (Kotlin)
- [ ] Nivel 11 — Bases de datos vectoriales (Python)
- [ ] Nivel 12 — RAG (Kotlin)
- [ ] Nivel 12 — RAG (Python)
- [ ] Nivel 13 — Agentes y Tool Calling (Kotlin)
- [ ] Nivel 13 — Agentes y Tool Calling (Python)

### Bloque 4 — Produccion y Proyecto Final
- [ ] Nivel 14 — Orquestacion avanzada (Kotlin)
- [ ] Nivel 14 — Orquestacion avanzada (Python)
- [ ] Nivel 15 — Despliegue de apps AI
- [ ] Nivel 16 — Seguridad y produccion
- [ ] Nivel 17 — Proyecto final

---

## Estadisticas del repositorio

| Metrica | Valor |
|---------|-------|
| Niveles | 18 (00-17) |
| Ejercicios | 91 (71 Kotlin + 20 Python) |
| READMEs de teoria | 23 (18 Kotlin + 5 Python) |
| Lineas de teoria | 25,000+ |
| Bloques tematicos | 4 |

---

## Requisitos previos

- **Spring Path completado** (o conocimiento equivalente de Spring Boot + Kotlin)
- **JDK 21+** instalado
- **Python 3.11+** (para el track Python, desde nivel 09)
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
| 09 | API keys (OpenAI/Anthropic), `pip` / `uv` (track Python) |
| 10 | Spring AI (dependencias Maven) |
| 11 | PostgreSQL + pgvector (via Docker), Qdrant (track Python) |
| 12 | LangChain / LlamaIndex (track Python) |
| 13-14 | Spring AI / LangChain4j (Kotlin), LangChain / CrewAI / LangGraph (Python) |
| 15 | Ollama (model serving local) |
