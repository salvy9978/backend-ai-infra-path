# Nivel 15 — Deploy de Aplicaciones de IA

> Tu sistema de IA funciona en local. Ahora necesitas que funcione en produccion: con Kubernetes, auto-scaling inteligente, costes controlados y latencia aceptable. Desplegar IA no es como desplegar un CRUD: los recursos, la latencia y el coste son ordenes de magnitud diferentes.

---

## Contenido

- [1. Desplegando Spring AI en Kubernetes](#1-desplegando-spring-ai-en-kubernetes)
  - [1.1 Diferencias con un deploy tradicional](#11-diferencias-con-un-deploy-tradicional)
  - [1.2 Dockerfile para Spring AI](#12-dockerfile-para-spring-ai)
  - [1.3 Manifiestos de Kubernetes](#13-manifiestos-de-kubernetes)
  - [1.4 Configuracion de recursos](#14-configuracion-de-recursos)
  - [1.5 Consideraciones de GPU](#15-consideraciones-de-gpu)
  - [1.6 Health checks para servicios de IA](#16-health-checks-para-servicios-de-ia)
- [2. Model Serving Local](#2-model-serving-local)
  - [2.1 Por que servir modelos localmente](#21-por-que-servir-modelos-localmente)
  - [2.2 Ollama: setup y configuracion](#22-ollama-setup-y-configuracion)
  - [2.3 vLLM para produccion](#23-vllm-para-produccion)
  - [2.4 Comparativa de opciones](#24-comparativa-de-opciones)
  - [2.5 Ollama en Kubernetes](#25-ollama-en-kubernetes)
- [3. Auto-scaling para Workloads de IA](#3-auto-scaling-para-workloads-de-ia)
  - [3.1 Por que el auto-scaling tradicional no sirve](#31-por-que-el-auto-scaling-tradicional-no-sirve)
  - [3.2 Scaling basado en tokens](#32-scaling-basado-en-tokens)
  - [3.3 Scaling basado en cola](#33-scaling-basado-en-cola)
  - [3.4 KEDA para auto-scaling avanzado](#34-keda-para-auto-scaling-avanzado)
  - [3.5 Scale-to-zero](#35-scale-to-zero)
- [4. Gestion de Costes](#4-gestion-de-costes)
  - [4.1 La realidad de los costes de IA](#41-la-realidad-de-los-costes-de-ia)
  - [4.2 Seleccion de modelo por tarea](#42-seleccion-de-modelo-por-tarea)
  - [4.3 Estrategias de caching](#43-estrategias-de-caching)
  - [4.4 Batching de requests](#44-batching-de-requests)
  - [4.5 Presupuesto y alertas](#45-presupuesto-y-alertas)
- [5. Optimizacion de Latencia](#5-optimizacion-de-latencia)
  - [5.1 Anatomia de la latencia en IA](#51-anatomia-de-la-latencia-en-ia)
  - [5.2 Streaming de respuestas](#52-streaming-de-respuestas)
  - [5.3 Connection pooling](#53-connection-pooling)
  - [5.4 Configuracion de timeouts](#54-configuracion-de-timeouts)
  - [5.5 Prefetching y warming](#55-prefetching-y-warming)
- [6. Monitoring de Metricas de IA](#6-monitoring-de-metricas-de-ia)
  - [6.1 Metricas especificas de IA](#61-metricas-especificas-de-ia)
  - [6.2 Instrumentacion con Micrometer](#62-instrumentacion-con-micrometer)
  - [6.3 Dashboard de Grafana](#63-dashboard-de-grafana)
  - [6.4 Alertas criticas](#64-alertas-criticas)
  - [6.5 Trazabilidad de requests](#65-trazabilidad-de-requests)
- [7. Resumen](#7-resumen)
- [8. Ejercicios](#8-ejercicios)
- [9. Recursos](#9-recursos)

---

## 1. Desplegando Spring AI en Kubernetes

### 1.1 Diferencias con un deploy tradicional

Un servicio Spring Boot tipico consume ~256MB-512MB de RAM y responde en ~50ms. Un servicio con IA puede:

| Aspecto | CRUD tradicional | Servicio con IA (API externa) | Servicio con IA (modelo local) |
|---------|-----------------|-------------------------------|-------------------------------|
| **RAM** | 256MB - 512MB | 512MB - 1GB | 4GB - 32GB |
| **CPU** | 0.5 - 1 core | 1 - 2 cores | 4+ cores o GPU |
| **Latencia p50** | 10-50ms | 500ms - 3s | 200ms - 10s |
| **Latencia p99** | 100-200ms | 5s - 30s | 10s - 60s |
| **Coste por request** | ~$0.000001 | $0.001 - $0.10 | Coste fijo infra |
| **Escalamiento** | Horizontal facil | Horizontal facil | Limitado por GPU/RAM |

Estas diferencias afectan cada decision de infraestructura.

### 1.2 Dockerfile para Spring AI

```dockerfile
# Multi-stage build optimizado para Spring AI
FROM eclipse-temurin:21-jdk-alpine AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src

# Descargar dependencias primero (cache de Docker layer)
RUN --mount=type=cache,target=/root/.m2 \
    mvn dependency:go-offline -B

# Compilar
RUN --mount=type=cache,target=/root/.m2 \
    mvn package -DskipTests -B

# Extraer layers de Spring Boot para cache optimo
RUN java -Djarmode=layertools -jar target/*.jar extract --destination target/extracted

# Runtime image minima
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Crear usuario no-root
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring

# Copiar layers en orden de menos a mas frecuente cambio
COPY --from=builder /app/target/extracted/dependencies/ ./
COPY --from=builder /app/target/extracted/spring-boot-loader/ ./
COPY --from=builder /app/target/extracted/snapshot-dependencies/ ./
COPY --from=builder /app/target/extracted/application/ ./

# Configuracion de JVM optimizada para contenedores
ENV JAVA_OPTS="-XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -XX:+UseG1GC \
    -XX:+UseStringDeduplication"

EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

### 1.3 Manifiestos de Kubernetes

Deployment basico para un servicio Spring AI que usa APIs externas:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-service
  labels:
    app: ai-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ai-service
  template:
    metadata:
      labels:
        app: ai-service
    spec:
      containers:
        - name: ai-service
          image: registry.example.com/ai-service:1.0.0
          ports:
            - containerPort: 8080
          env:
            - name: OPENAI_API_KEY
              valueFrom:
                secretKeyRef:
                  name: ai-secrets
                  key: openai-api-key
            - name: SPRING_PROFILES_ACTIVE
              value: "production"
          resources:
            requests:
              memory: "512Mi"    # minimo garantizado
              cpu: "500m"
            limits:
              memory: "1Gi"      # maximo permitido
              cpu: "2000m"       # 2 cores (las llamadas son IO-bound)
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 5
          startupProbe:
            httpGet:
              path: /actuator/health
              port: 8080
            failureThreshold: 30   # 30 * 10s = 5 min maximo para arrancar
            periodSeconds: 10
---
apiVersion: v1
kind: Secret
metadata:
  name: ai-secrets
type: Opaque
data:
  openai-api-key: base64-encoded-key-aqui
---
apiVersion: v1
kind: Service
metadata:
  name: ai-service
spec:
  selector:
    app: ai-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

### 1.4 Configuracion de recursos

Reglas para dimensionar recursos:

```yaml
# Servicio que llama a APIs externas (OpenAI, Claude, etc.)
# CPU bajo, memoria moderada, las llamadas son IO-bound
resources:
  requests:
    memory: "512Mi"
    cpu: "250m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

# Servicio que ejecuta embeddings localmente
# CPU alto, memoria alta para el modelo
resources:
  requests:
    memory: "2Gi"
    cpu: "2000m"
  limits:
    memory: "4Gi"
    cpu: "4000m"

# Servicio que sirve un LLM local (Ollama, vLLM)
# Necesita GPU o muchisima RAM
resources:
  requests:
    memory: "8Gi"
    cpu: "4000m"
    nvidia.com/gpu: 1        # solicitar GPU
  limits:
    memory: "16Gi"
    cpu: "8000m"
    nvidia.com/gpu: 1
```

### 1.5 Consideraciones de GPU

Para modelos locales, necesitas GPUs en tu cluster:

```yaml
# Instalar el NVIDIA device plugin
# kubectl apply -f https://raw.githubusercontent.com/NVIDIA/k8s-device-plugin/v0.14.0/nvidia-device-plugin.yml

# Node selector para nodos con GPU
spec:
  template:
    spec:
      nodeSelector:
        nvidia.com/gpu.present: "true"
      tolerations:
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
      containers:
        - name: model-server
          resources:
            limits:
              nvidia.com/gpu: 1    # una GPU
```

**Realidad practica**: las GPUs en Kubernetes son caras. Un nodo con una NVIDIA A100 cuesta ~$3/hora en cloud. Considera si el volumen de requests justifica el coste frente a usar una API externa.

### 1.6 Health checks para servicios de IA

Un health check para IA debe verificar mas que "el servidor responde":

```kotlin
@Component
class AiHealthIndicator(
    private val chatModel: ChatLanguageModel
) : HealthIndicator {

    override fun health(): Health {
        return try {
            // Verificar que el LLM responde
            val start = System.currentTimeMillis()
            val response = chatModel.generate("Responde OK")
            val latency = System.currentTimeMillis() - start

            if (latency > 10000) {
                // Responde pero muy lento: degradado
                Health.status("DEGRADED")
                    .withDetail("latency_ms", latency)
                    .withDetail("message", "LLM respondiendo lento")
                    .build()
            } else {
                Health.up()
                    .withDetail("latency_ms", latency)
                    .build()
            }
        } catch (e: Exception) {
            Health.down()
                .withDetail("error", e.message)
                .build()
        }
    }
}
```

---

## 2. Model Serving Local

### 2.1 Por que servir modelos localmente

| Razon | Explicacion |
|-------|-------------|
| **Privacidad** | Los datos no salen de tu infraestructura |
| **Latencia** | Sin round-trip a internet (~100ms vs ~1-3s) |
| **Coste predecible** | Pagas infra fija, no por token |
| **Sin limites de rate** | Tu capacidad es tu hardware |
| **Disponibilidad** | No dependes de uptime de terceros |
| **Compliance** | Regulaciones como GDPR pueden requerirlo |

### 2.2 Ollama: setup y configuracion

Ollama es la forma mas sencilla de ejecutar modelos localmente:

```bash
# Instalar Ollama
curl -fsSL https://ollama.com/install.sh | sh

# Descargar modelos
ollama pull llama3.1:8b         # modelo general, 8B parametros (~5GB)
ollama pull nomic-embed-text     # modelo de embeddings (~270MB)
ollama pull codellama:13b        # especializado en codigo (~7GB)

# Verificar que funciona
ollama run llama3.1:8b "Hola, como estas?"

# Ollama expone una API REST compatible con OpenAI
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.1:8b",
  "prompt": "Hola"
}'
```

Configuracion de Spring AI con Ollama:

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434
      chat:
        model: llama3.1:8b
        options:
          temperature: 0.7
          num-ctx: 4096          # ventana de contexto
          num-predict: 2048      # maximo de tokens a generar
      embedding:
        model: nomic-embed-text
```

```kotlin
// Spring AI auto-configura los beans con Ollama
@Service
class LocalAiService(
    private val chatClient: ChatClient  // usa Ollama automaticamente
) {

    fun preguntar(query: String): String {
        return chatClient.prompt()
            .user(query)
            .call()
            .content()
    }
}
```

### 2.3 vLLM para produccion

Ollama es excelente para desarrollo. Para produccion con alto throughput, usa vLLM:

```bash
# vLLM con Docker
docker run --gpus all \
    -v /path/to/models:/models \
    -p 8000:8000 \
    vllm/vllm-openai:latest \
    --model meta-llama/Llama-3.1-8B-Instruct \
    --max-model-len 8192 \
    --tensor-parallel-size 1 \
    --gpu-memory-utilization 0.90
```

vLLM es superior a Ollama en produccion porque:

| Caracteristica | Ollama | vLLM |
|----------------|--------|------|
| **Batching** | Secuencial | Continuous batching (5-10x throughput) |
| **Memoria GPU** | Basico | PagedAttention (uso eficiente) |
| **API** | Propia + compatible OpenAI | 100% compatible OpenAI |
| **Metricas** | Basicas | Prometheus nativo |
| **Tensor parallel** | No | Si (multiples GPUs) |
| **Facilidad** | Muy facil | Requiere configuracion |

Configuracion de Spring AI con vLLM (usa la misma API que OpenAI):

```yaml
spring:
  ai:
    openai:
      base-url: http://vllm-service:8000/v1   # apuntar a vLLM
      api-key: "dummy"                          # vLLM no requiere key
      chat:
        model: meta-llama/Llama-3.1-8B-Instruct
```

### 2.4 Comparativa de opciones

| Opcion | Caso de uso | Hardware minimo | Throughput |
|--------|------------|-----------------|------------|
| **Ollama** | Desarrollo, demos, bajo volumen | 8GB RAM (CPU) / GPU 6GB | ~5-10 req/min |
| **vLLM** | Produccion alto volumen | GPU 16GB+ (A100 ideal) | ~100-500 req/min |
| **TGI** (HuggingFace) | Produccion, modelos HF | GPU 16GB+ | ~100-300 req/min |
| **llama.cpp** | Edge, dispositivos limitados | 4GB RAM | ~2-5 req/min |

### 2.5 Ollama en Kubernetes

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ollama
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ollama
  template:
    metadata:
      labels:
        app: ollama
    spec:
      containers:
        - name: ollama
          image: ollama/ollama:latest
          ports:
            - containerPort: 11434
          resources:
            requests:
              memory: "8Gi"
              cpu: "4000m"
            limits:
              memory: "16Gi"
              cpu: "8000m"
          volumeMounts:
            - name: ollama-data
              mountPath: /root/.ollama    # persistir modelos descargados
          # Comando para descargar modelos al iniciar
          lifecycle:
            postStart:
              exec:
                command:
                  - /bin/sh
                  - -c
                  - "ollama pull llama3.1:8b && ollama pull nomic-embed-text"
      volumes:
        - name: ollama-data
          persistentVolumeClaim:
            claimName: ollama-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ollama-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi    # modelos pueden ocupar mucho espacio
---
apiVersion: v1
kind: Service
metadata:
  name: ollama
spec:
  selector:
    app: ollama
  ports:
    - port: 11434
      targetPort: 11434
```

---

## 3. Auto-scaling para Workloads de IA

### 3.1 Por que el auto-scaling tradicional no sirve

El HPA (Horizontal Pod Autoscaler) tradicional escala por CPU o memoria. Pero los servicios de IA tienen un patron diferente:

```
CPU al 20%,  pero 50 requests esperando respuesta del LLM  -> necesitas escalar
CPU al 80%,  pero solo 2 requests activas                   -> no necesitas escalar
Memoria fija, el modelo ocupa siempre lo mismo               -> escalar por memoria no tiene sentido
```

### 3.2 Scaling basado en tokens

Mide el consumo de tokens y escala segun eso:

```kotlin
// Registrar metricas de tokens
@Component
class TokenMetrics(private val meterRegistry: MeterRegistry) {

    private val tokensEnCola = AtomicInteger(0)
    private val tokensProcessados = meterRegistry.counter("ai.tokens.processed")

    fun registrarRequest(estimatedTokens: Int) {
        tokensEnCola.addAndGet(estimatedTokens)
    }

    fun registrarCompletado(actualTokens: Int) {
        tokensEnCola.addAndGet(-actualTokens)
        tokensProcessados.increment(actualTokens.toDouble())
    }

    // Exponer como metrica de Prometheus
    init {
        Gauge.builder("ai.tokens.queued") { tokensEnCola.toDouble() }
            .register(meterRegistry)
    }
}
```

### 3.3 Scaling basado en cola

Si usas un patron de cola (recomendado para IA), escala segun la profundidad de la cola:

```yaml
# application.yml - publicar metricas de cola
management:
  endpoints:
    web:
      exposure:
        include: prometheus, health
  metrics:
    export:
      prometheus:
        enabled: true
    tags:
      application: ai-service
```

```kotlin
// Cola de requests de IA
@Service
class AiRequestQueue(private val meterRegistry: MeterRegistry) {

    private val queue = LinkedBlockingQueue<AiRequest>(1000)

    init {
        // Exponer tamanio de cola como metrica
        Gauge.builder("ai.queue.size") { queue.size.toDouble() }
            .register(meterRegistry)
        Gauge.builder("ai.queue.remaining_capacity") {
            queue.remainingCapacity().toDouble()
        }.register(meterRegistry)
    }

    fun enqueue(request: AiRequest): CompletableFuture<AiResponse> {
        val future = CompletableFuture<AiResponse>()
        request.responseFuture = future
        queue.put(request)
        return future
    }

    // Workers procesan la cola
    @Scheduled(fixedDelay = 100)
    fun processQueue() {
        val request = queue.poll() ?: return
        try {
            val response = llmService.process(request)
            request.responseFuture.complete(response)
        } catch (e: Exception) {
            request.responseFuture.completeExceptionally(e)
        }
    }
}
```

### 3.4 KEDA para auto-scaling avanzado

KEDA (Kubernetes Event-Driven Autoscaling) permite escalar basado en metricas custom:

```yaml
# Instalar KEDA: helm install keda kedacore/keda

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ai-service-scaler
spec:
  scaleTargetRef:
    name: ai-service
  minReplicaCount: 1
  maxReplicaCount: 10
  triggers:
    # Escalar por profundidad de cola en Prometheus
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: ai_queue_size
        query: sum(ai_queue_size{application="ai-service"})
        threshold: "20"           # escalar cuando hay >20 items en cola
    # Escalar por latencia p95
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: ai_request_latency
        query: histogram_quantile(0.95, rate(ai_request_duration_seconds_bucket[5m]))
        threshold: "5"            # escalar cuando p95 > 5 segundos
```

### 3.5 Scale-to-zero

Para ahorrar costes, escala a cero cuando no hay trafico:

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: ai-service-scaler
spec:
  scaleTargetRef:
    name: ai-service
  minReplicaCount: 0              # permite scale-to-zero
  maxReplicaCount: 5
  cooldownPeriod: 300             # espera 5 min sin trafico antes de apagar
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus:9090
        metricName: ai_requests_total
        query: sum(rate(ai_requests_total[5m]))
        threshold: "1"
```

**Advertencia**: scale-to-zero implica cold start. Un pod de Spring AI puede tardar 30-60s en arrancar. Configura un startupProbe generoso y considera mantener minimo 1 replica para endpoints criticos.

---

## 4. Gestion de Costes

### 4.1 La realidad de los costes de IA

Un calculo realista para una aplicacion con 10,000 usuarios activos:

| Concepto | Calculo | Coste mensual |
|----------|---------|---------------|
| Llamadas a GPT-4o | 50k requests * ~1000 tokens * $0.005/1k tokens | ~$250 |
| Llamadas a GPT-4o-mini | 200k requests * ~500 tokens * $0.00015/1k tokens | ~$15 |
| Embeddings | 100k documents * 500 tokens * $0.00002/1k tokens | ~$1 |
| Infra K8s (API) | 2 pods * $50/pod/mes | ~$100 |
| Infra K8s (Ollama) | 1 GPU pod * $2000/mes | ~$2000 |
| **Total con API externa** | | **~$366/mes** |
| **Total con modelo local** | | **~$2100/mes** |

### 4.2 Seleccion de modelo por tarea

No uses el modelo mas potente para todo. Asigna modelos segun la complejidad:

```kotlin
@Service
class ModelRouter(
    private val cheapModel: ChatClient,    // GPT-4o-mini, Llama 3.1 8B
    private val powerModel: ChatClient,    // GPT-4o, Claude 3.5
    private val classifier: TaskClassifier
) {

    fun procesar(request: AiRequest): String {
        val complejidad = classifier.clasificar(request.prompt)

        return when (complejidad) {
            Complejidad.SIMPLE -> {
                // FAQ, clasificacion, extraccion simple -> modelo barato
                cheapModel.prompt().user(request.prompt).call().content()
            }
            Complejidad.COMPLEJA -> {
                // Razonamiento, sintesis, analisis -> modelo potente
                powerModel.prompt().user(request.prompt).call().content()
            }
        }
    }
}

// Configuracion de multiples modelos
@Configuration
class MultiModelConfig {

    @Bean("cheapModel")
    fun cheapModel(): ChatClient {
        return ChatClient.builder(
            OpenAiChatModel(OpenAiApi("key"), OpenAiChatOptions.builder()
                .withModel("gpt-4o-mini")
                .build())
        ).build()
    }

    @Bean("powerModel")
    fun powerModel(): ChatClient {
        return ChatClient.builder(
            OpenAiChatModel(OpenAiApi("key"), OpenAiChatOptions.builder()
                .withModel("gpt-4o")
                .build())
        ).build()
    }
}
```

### 4.3 Estrategias de caching

El cache es la herramienta numero uno para reducir costes:

```kotlin
@Service
class CachedAiService(
    private val chatClient: ChatClient,
    private val cache: SemanticCache,
    private val meterRegistry: MeterRegistry
) {

    private val cacheHits = meterRegistry.counter("ai.cache.hits")
    private val cacheMisses = meterRegistry.counter("ai.cache.misses")
    private val costSaved = meterRegistry.counter("ai.cost.saved.usd")

    fun query(prompt: String): String {
        return cache.getOrCompute(prompt) {
            cacheMisses.increment()
            chatClient.prompt().user(prompt).call().content()
        }.also { resultado ->
            if (cache.wasHit()) {
                cacheHits.increment()
                // Estimar coste ahorrado (~$0.002 por llamada a GPT-4o-mini)
                costSaved.increment(0.002)
            }
        }
    }
}
```

### 4.4 Batching de requests

Agrupa multiples requests para reducir overhead:

```kotlin
@Service
class BatchEmbeddingService(private val embeddingModel: EmbeddingModel) {

    // En lugar de generar embeddings uno por uno
    // los agrupamos en batches
    fun embedBatch(textos: List<String>): List<FloatArray> {
        // Un solo request con multiples textos
        // Mucho mas eficiente que N requests individuales
        return embeddingModel.embed(textos)
            .map { it.output() }
    }

    // Collector que agrupa requests que llegan en una ventana de tiempo
    private val batchCollector = BatchCollector<String, FloatArray>(
        maxBatchSize = 100,
        maxWaitMs = 50,    // espera hasta 50ms para agrupar
        processFn = { batch -> embedBatch(batch) }
    )

    fun embedSingle(texto: String): CompletableFuture<FloatArray> {
        return batchCollector.submit(texto)
    }
}
```

### 4.5 Presupuesto y alertas

```kotlin
@Component
class CostTracker(private val meterRegistry: MeterRegistry) {

    private val dailyCost = AtomicDouble(0.0)
    private val monthlyCost = AtomicDouble(0.0)

    fun registrarCoste(tokens: Int, modelPrice: Double) {
        val cost = (tokens / 1000.0) * modelPrice
        dailyCost.addAndGet(cost)
        monthlyCost.addAndGet(cost)
        meterRegistry.counter("ai.cost.total.usd").increment(cost)
    }

    // Reset diario
    @Scheduled(cron = "0 0 0 * * *")
    fun resetDaily() {
        dailyCost.set(0.0)
    }

    // Verificar presupuesto
    @Scheduled(fixedRate = 60000)  // cada minuto
    fun checkBudget() {
        if (dailyCost.get() > 50.0) {
            logger.warn("ALERTA: coste diario de IA supera $50: ${dailyCost.get()}")
            // Enviar notificacion, activar circuit breaker, etc.
        }
    }
}
```

---

## 5. Optimizacion de Latencia

### 5.1 Anatomia de la latencia en IA

```
Total latencia de una request:

[Red ida]  [Cola servidor]  [TTFT]  [Generacion tokens]  [Red vuelta]
  ~20ms      ~0-500ms      ~200ms    ~1-10s (variable)     ~20ms

TTFT = Time To First Token
La generacion es proporcional al numero de tokens de salida (~30-50 tokens/segundo)
```

### 5.2 Streaming de respuestas

No esperes la respuesta completa. Enviale tokens al cliente conforme se generan:

```kotlin
@RestController
class StreamingController(private val chatClient: ChatClient) {

    @GetMapping("/chat/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun chatStream(@RequestParam prompt: String): Flux<String> {
        return chatClient.prompt()
            .user(prompt)
            .stream()
            .content()          // Flux<String> - cada elemento es un chunk de texto
    }
}
```

Configuracion del cliente para streaming con SSE:

```kotlin
// Con WebClient para comunicarse con otro microservicio
@Service
class AiGateway(private val webClient: WebClient) {

    fun streamFromAiService(prompt: String): Flux<String> {
        return webClient.get()
            .uri("/chat/stream?prompt={prompt}", prompt)
            .accept(MediaType.TEXT_EVENT_STREAM)
            .retrieve()
            .bodyToFlux(String::class.java)
    }
}
```

### 5.3 Connection pooling

Las conexiones HTTP a APIs de IA deben ser reutilizadas:

```kotlin
@Configuration
class HttpClientConfig {

    @Bean
    fun restClient(): RestClient {
        val connectionManager = PoolingHttpClientConnectionManagerBuilder.create()
            .setMaxConnTotal(50)            // total de conexiones
            .setMaxConnPerRoute(20)         // conexiones por host
            .setDefaultConnectionConfig(
                ConnectionConfig.custom()
                    .setConnectTimeout(Timeout.ofSeconds(5))
                    .setSocketTimeout(Timeout.ofSeconds(60))  // LLMs son lentos
                    .build()
            )
            .build()

        val httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .setKeepAliveStrategy { _, _ -> TimeValue.ofMinutes(5) }
            .build()

        return RestClient.builder()
            .requestFactory(HttpComponentsClientHttpRequestFactory(httpClient))
            .build()
    }
}
```

### 5.4 Configuracion de timeouts

Los LLMs son lentos. Configura timeouts realistas:

```yaml
spring:
  ai:
    openai:
      chat:
        options:
          # Timeout de conexion: cuanto esperar para establecer conexion
          # 5s es suficiente, si no conecta en 5s hay un problema
          connect-timeout: 5000
          # Timeout de lectura: cuanto esperar por la respuesta
          # Los LLMs pueden tardar hasta 60s para respuestas largas
          read-timeout: 60000

# Para llamadas internas entre microservicios
app:
  ai:
    timeout:
      simple-query: 15s    # clasificacion, extraccion
      complex-query: 60s   # generacion larga, resumen
      embedding: 10s       # embeddings son rapidos
```

```kotlin
@Service
class TimeoutAwareService(private val chatClient: ChatClient) {

    fun queryConTimeout(prompt: String, timeout: Duration): String {
        return try {
            CompletableFuture.supplyAsync {
                chatClient.prompt().user(prompt).call().content()
            }.get(timeout.toMillis(), TimeUnit.MILLISECONDS)
        } catch (e: TimeoutException) {
            logger.warn("Timeout de IA tras ${timeout.seconds}s para: ${prompt.take(100)}")
            "Lo siento, la respuesta esta tardando demasiado. Intenta de nuevo."
        }
    }
}
```

### 5.5 Prefetching y warming

Precalcula respuestas que sabes que se van a necesitar:

```kotlin
@Service
class PrefetchService(
    private val aiService: AiService,
    private val cache: SemanticCache
) {

    // Al arrancar, precalcular respuestas frecuentes
    @EventListener(ApplicationReadyEvent::class)
    fun warmCache() {
        val preguntasFrecuentes = listOf(
            "Cual es tu politica de devolucion?",
            "Como puedo rastrear mi pedido?",
            "Cuales son los metodos de pago aceptados?",
            "Cual es el horario de atencion?"
        )

        preguntasFrecuentes.forEach { pregunta ->
            try {
                cache.getOrCompute(pregunta) { aiService.responder(pregunta) }
                logger.info("Cache pre-calentado para: $pregunta")
            } catch (e: Exception) {
                logger.warn("No se pudo precalentar cache para: $pregunta", e)
            }
        }
    }
}
```

---

## 6. Monitoring de Metricas de IA

### 6.1 Metricas especificas de IA

Ademas de las metricas clasicas (CPU, memoria, requests/s), necesitas:

| Metrica | Que mide | Por que importa |
|---------|----------|-----------------|
| `ai_tokens_input_total` | Tokens de entrada consumidos | Coste |
| `ai_tokens_output_total` | Tokens de salida generados | Coste + latencia |
| `ai_request_duration_seconds` | Latencia total por request | UX |
| `ai_ttft_seconds` | Time To First Token | UX en streaming |
| `ai_cost_usd_total` | Coste acumulado en dolares | Presupuesto |
| `ai_cache_hit_ratio` | % de requests servidas por cache | Eficiencia |
| `ai_error_rate` | % de requests que fallan | Fiabilidad |
| `ai_hallucination_detected` | Alucinaciones detectadas | Calidad |

### 6.2 Instrumentacion con Micrometer

```kotlin
@Component
class AiMetricsInterceptor(private val meterRegistry: MeterRegistry) {

    fun <T> instrumentar(
        operacion: String,
        modelo: String,
        block: () -> AiResponse<T>
    ): T {
        val timer = Timer.builder("ai.request.duration")
            .tag("operation", operacion)
            .tag("model", modelo)
            .register(meterRegistry)

        return timer.recordCallable {
            try {
                val response = block()

                // Registrar tokens
                Counter.builder("ai.tokens.input")
                    .tag("model", modelo)
                    .register(meterRegistry)
                    .increment(response.inputTokens.toDouble())

                Counter.builder("ai.tokens.output")
                    .tag("model", modelo)
                    .register(meterRegistry)
                    .increment(response.outputTokens.toDouble())

                // Registrar coste estimado
                val cost = estimarCoste(modelo, response.inputTokens, response.outputTokens)
                Counter.builder("ai.cost.usd")
                    .tag("model", modelo)
                    .register(meterRegistry)
                    .increment(cost)

                response.data
            } catch (e: Exception) {
                Counter.builder("ai.errors")
                    .tag("model", modelo)
                    .tag("error_type", e.javaClass.simpleName)
                    .register(meterRegistry)
                    .increment()
                throw e
            }
        }!!
    }

    private fun estimarCoste(modelo: String, input: Int, output: Int): Double {
        val precios = mapOf(
            "gpt-4o" to Pair(0.005, 0.015),        // $/1k tokens (input, output)
            "gpt-4o-mini" to Pair(0.00015, 0.0006),
            "claude-3-5-sonnet" to Pair(0.003, 0.015)
        )
        val (precioInput, precioOutput) = precios[modelo] ?: Pair(0.001, 0.002)
        return (input / 1000.0 * precioInput) + (output / 1000.0 * precioOutput)
    }
}
```

### 6.3 Dashboard de Grafana

Paneles esenciales para tu dashboard de IA:

```json
{
  "panels": [
    {
      "title": "Coste acumulado hoy ($)",
      "type": "stat",
      "query": "sum(ai_cost_usd_total) - sum(ai_cost_usd_total offset 1d)"
    },
    {
      "title": "Latencia p50 / p95 / p99",
      "type": "timeseries",
      "queries": [
        "histogram_quantile(0.50, rate(ai_request_duration_seconds_bucket[5m]))",
        "histogram_quantile(0.95, rate(ai_request_duration_seconds_bucket[5m]))",
        "histogram_quantile(0.99, rate(ai_request_duration_seconds_bucket[5m]))"
      ]
    },
    {
      "title": "Tokens por minuto (input vs output)",
      "type": "timeseries",
      "queries": [
        "rate(ai_tokens_input_total[1m])",
        "rate(ai_tokens_output_total[1m])"
      ]
    },
    {
      "title": "Cache hit ratio",
      "type": "gauge",
      "query": "rate(ai_cache_hits_total[5m]) / (rate(ai_cache_hits_total[5m]) + rate(ai_cache_misses_total[5m]))"
    },
    {
      "title": "Error rate por modelo",
      "type": "timeseries",
      "query": "rate(ai_errors_total[5m]) by (model)"
    }
  ]
}
```

### 6.4 Alertas criticas

```yaml
# alerting-rules.yml para Prometheus
groups:
  - name: ai-alerts
    rules:
      # Coste diario excesivo
      - alert: AiDailyCostHigh
        expr: sum(increase(ai_cost_usd_total[24h])) > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Coste diario de IA supera $100"

      # Latencia degradada
      - alert: AiLatencyHigh
        expr: histogram_quantile(0.95, rate(ai_request_duration_seconds_bucket[5m])) > 10
        for: 10m
        labels:
          severity: critical
        annotations:
          summary: "Latencia p95 de IA supera 10 segundos"

      # Tasa de error alta
      - alert: AiErrorRateHigh
        expr: rate(ai_errors_total[5m]) / rate(ai_request_duration_seconds_count[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Mas del 5% de requests de IA estan fallando"

      # Cache hit ratio bajo
      - alert: AiCacheHitLow
        expr: >
          rate(ai_cache_hits_total[1h]) /
          (rate(ai_cache_hits_total[1h]) + rate(ai_cache_misses_total[1h])) < 0.3
        for: 30m
        labels:
          severity: warning
        annotations:
          summary: "Cache hit ratio por debajo del 30%"
```

### 6.5 Trazabilidad de requests

Cada request de IA debe ser rastreable de principio a fin:

```kotlin
@Component
class AiTracingFilter : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val traceId = request.getHeader("X-Trace-Id") ?: UUID.randomUUID().toString()
        MDC.put("traceId", traceId)
        MDC.put("userId", extractUserId(request))

        try {
            filterChain.doFilter(request, response)
        } finally {
            MDC.clear()
        }
    }
}

// Loguear cada interaccion con el LLM
@Aspect
@Component
class AiLoggingAspect {

    private val logger = LoggerFactory.getLogger(javaClass)

    @Around("execution(* org.springframework.ai.chat.client.ChatClient.CallResponseSpec.content(..))")
    fun logAiCall(joinPoint: ProceedingJoinPoint): Any? {
        val traceId = MDC.get("traceId")
        logger.info("[{}] Llamada a LLM iniciada", traceId)
        val start = System.currentTimeMillis()

        return try {
            val result = joinPoint.proceed()
            val elapsed = System.currentTimeMillis() - start
            logger.info("[{}] LLM respondio en {}ms", traceId, elapsed)
            result
        } catch (e: Exception) {
            logger.error("[{}] LLM fallo: {}", traceId, e.message)
            throw e
        }
    }
}
```

---

## 7. Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| **Deploy K8s** | Contenedorizar y desplegar servicio de IA | Dockerfile multi-stage + manifiestos K8s |
| **Ollama** | Servidor de modelos local facil de usar | `ollama pull` + configurar `spring.ai.ollama` |
| **vLLM** | Servidor de modelos de alto rendimiento | Docker con GPU + API compatible OpenAI |
| **KEDA** | Auto-scaling basado en metricas custom | ScaledObject con triggers de Prometheus |
| **Model routing** | Elegir modelo segun complejidad | Clasificador + multiples ChatClient beans |
| **Semantic cache** | Cache por similitud de significado | Embeddings + vector store + threshold |
| **Streaming** | Enviar tokens conforme se generan | `Flux<String>` + SSE |
| **Token metrics** | Medir consumo de tokens | Micrometer counters por modelo |
| **Cost tracking** | Rastrear gasto en IA | Counter de USD + alertas Prometheus |
| **Tracing** | Rastrear requests de principio a fin | MDC + traceId + logging estructurado |

---

## 8. Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | [Spring AI en K8s](ejercicio-01-spring-ai-k8s/) | Dockerizar una app Spring AI, crear manifiestos de K8s con health checks, secrets y resource limits | Dockerfile, Deployment, Service, Secret, Probes |
| 02 | [Model serving local](ejercicio-02-model-serving/) | Configurar Ollama en Docker Compose, conectar Spring AI, comparar latencia con API externa | Ollama, Docker Compose, `spring.ai.ollama`, benchmarks |
| 03 | [Auto-scaling para IA](ejercicio-03-autoscaling-ai/) | Implementar metricas custom de tokens y cola, configurar KEDA para escalar por profundidad de cola | Micrometer, KEDA, ScaledObject, Prometheus |
| 04 | [Gestion de costes](ejercicio-04-cost-management/) | Implementar model routing, semantic cache, cost tracking y alertas de presupuesto | ModelRouter, SemanticCache, CostTracker, alertas |

Haz los ejercicios en orden. Cada uno construye sobre los conceptos del anterior.

---

## 9. Recursos

- [Spring AI Reference - Ollama](https://docs.spring.io/spring-ai/reference/api/chat/ollama-chat.html)
- [Ollama Documentation](https://ollama.com/)
- [vLLM Documentation](https://docs.vllm.ai/)
- [KEDA Documentation](https://keda.sh/docs/)
- [Kubernetes GPU Scheduling](https://kubernetes.io/docs/tasks/manage-gpus/scheduling-gpus/)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/)
- [Spring Boot Docker Guide](https://spring.io/guides/topicals/spring-boot-docker)
- [OpenAI Pricing](https://openai.com/pricing)
