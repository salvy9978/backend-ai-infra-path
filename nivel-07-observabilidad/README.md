# Nivel 07 — Observabilidad: Metricas, Logs y Traces

> Tu aplicacion esta desplegada. Pero si no puedes ver que pasa dentro, estas volando a ciegas. La observabilidad te da visibilidad sobre el comportamiento real del sistema en produccion: que tan rapido responde, donde falla y por que.

---

## Contenido

- [1. Los tres pilares de la observabilidad](#1-los-tres-pilares-de-la-observabilidad)
  - [1.1 Metricas, logs y traces](#11-metricas-logs-y-traces)
  - [1.2 Diferencias y complementariedad](#12-diferencias-y-complementariedad)
- [2. Metricas con Prometheus](#2-metricas-con-prometheus)
  - [2.1 Arquitectura de Prometheus](#21-arquitectura-de-prometheus)
  - [2.2 Tipos de metricas](#22-tipos-de-metricas)
  - [2.3 PromQL basico](#23-promql-basico)
  - [2.4 Scraping y configuracion](#24-scraping-y-configuracion)
  - [2.5 Reglas de alerta](#25-reglas-de-alerta)
- [3. Dashboards con Grafana](#3-dashboards-con-grafana)
  - [3.1 Conceptos de Grafana](#31-conceptos-de-grafana)
  - [3.2 Crear un dashboard](#32-crear-un-dashboard)
  - [3.3 Variables de template](#33-variables-de-template)
- [4. Spring Boot Actuator y Micrometer](#4-spring-boot-actuator-y-micrometer)
  - [4.1 Actuator endpoints](#41-actuator-endpoints)
  - [4.2 Micrometer: la fachada de metricas](#42-micrometer-la-fachada-de-metricas)
  - [4.3 Tipos de metros en Micrometer](#43-tipos-de-metros-en-micrometer)
  - [4.4 Metricas personalizadas en Kotlin](#44-metricas-personalizadas-en-kotlin)
  - [4.5 Configuracion para Prometheus](#45-configuracion-para-prometheus)
- [5. Logging con Loki](#5-logging-con-loki)
  - [5.1 Arquitectura de Loki](#51-arquitectura-de-loki)
  - [5.2 Labels y mejores practicas](#52-labels-y-mejores-practicas)
  - [5.3 LogQL: consultando logs](#53-logql-consultando-logs)
  - [5.4 Configurar logging en Spring Boot](#54-configurar-logging-en-spring-boot)
- [6. Tracing distribuido](#6-tracing-distribuido)
  - [6.1 Conceptos fundamentales](#61-conceptos-fundamentales)
  - [6.2 OpenTelemetry](#62-opentelemetry)
  - [6.3 Auto-instrumentacion para Spring](#63-auto-instrumentacion-para-spring)
  - [6.4 Instrumentacion manual](#64-instrumentacion-manual)
  - [6.5 Jaeger y Zipkin](#65-jaeger-y-zipkin)
- [7. Alerting](#7-alerting)
  - [7.1 Alertmanager](#71-alertmanager)
  - [7.2 Integracion con Slack y PagerDuty](#72-integracion-con-slack-y-pagerduty)
  - [7.3 Buenas practicas de alerting](#73-buenas-practicas-de-alerting)
- [8. Stack completo con Docker Compose](#8-stack-completo-con-docker-compose)
- [9. Resumen](#9-resumen)
- [10. Ejercicios](#10-ejercicios)
- [11. Recursos](#11-recursos)

---

## 1. Los tres pilares de la observabilidad

### 1.1 Metricas, logs y traces

La observabilidad se construye sobre tres tipos de senales:

```
METRICAS                    LOGS                        TRACES
Numeros agregados           Eventos textuales           Camino de una request
en el tiempo                con timestamp               a traves de servicios

"99p latency = 120ms"       "ERROR: DB timeout"         "API -> Auth -> DB -> Cache"
"Requests/sec = 1500"       "User 42 logged in"         "Total: 250ms (DB: 180ms)"
"Error rate = 0.5%"         "Payment failed: ..."       "Span: payment-service 80ms"
```

| Pilar | Pregunta que responde | Ejemplo |
|-------|----------------------|---------|
| **Metricas** | Que tan bien funciona el sistema? | Latencia p99, tasa de errores, CPU |
| **Logs** | Que paso exactamente? | Stack trace de un error, acciones del usuario |
| **Traces** | Donde se paso el tiempo en una request? | Request tardo 500ms: 300ms en BD, 150ms en API externa |

### 1.2 Diferencias y complementariedad

```
Alerta: "Error rate subio al 5%"          <- METRICA (detecta el problema)
    |
    v
Investiga: "Que errores estan ocurriendo?" <- LOGS (entiende el problema)
    |
    v
Profundiza: "Por que tarda tanto?"         <- TRACE (localiza el cuello de botella)
```

Las metricas te dicen QUE pasa. Los logs te dicen POR QUE. Los traces te dicen DONDE.

---

## 2. Metricas con Prometheus

### 2.1 Arquitectura de Prometheus

Prometheus usa un modelo **pull**: el servidor de Prometheus hace scraping periodico a los endpoints de metricas de tus aplicaciones.

```
Tu App (Spring Boot)                    Prometheus Server
/actuator/prometheus  <----scrape----   almacena en TSDB
  (metricas en texto)     cada 15s      |
                                        |----> Grafana (visualizacion)
                                        |----> Alertmanager (alertas)
```

Componentes clave:
- **Prometheus Server**: scraper + almacenamiento time-series (TSDB)
- **Exporters**: procesos que exponen metricas de sistemas que no las tienen nativamente (node_exporter, postgres_exporter)
- **Alertmanager**: recibe alertas de Prometheus y las enruta (email, Slack, PagerDuty)
- **Pushgateway**: para jobs efimeros que no pueden ser scrapeados (batch jobs)

### 2.2 Tipos de metricas

| Tipo | Que mide | Ejemplo | Operacion tipica |
|------|----------|---------|-----------------|
| **Counter** | Total acumulado (solo sube) | Requests totales, errores totales | `rate()` para obtener tasa por segundo |
| **Gauge** | Valor actual (sube y baja) | Temperatura, memoria usada, conexiones activas | Valor directo |
| **Histogram** | Distribucion de valores en buckets | Latencia de requests, tamano de respuestas | `histogram_quantile()` para percentiles |
| **Summary** | Percentiles pre-calculados | Similar a histogram pero en el cliente | Valor directo de quantiles |

Ejemplo de como se ven en texto (formato exposition):

```
# HELP http_requests_total Total de requests HTTP
# TYPE http_requests_total counter
http_requests_total{method="GET",status="200"} 1542
http_requests_total{method="POST",status="201"} 324
http_requests_total{method="GET",status="500"} 12

# HELP http_request_duration_seconds Latencia de requests
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.01"} 500
http_request_duration_seconds_bucket{le="0.05"} 1200
http_request_duration_seconds_bucket{le="0.1"} 1800
http_request_duration_seconds_bucket{le="+Inf"} 1878
http_request_duration_seconds_sum 152.4
http_request_duration_seconds_count 1878
```

### 2.3 PromQL basico

PromQL es el lenguaje de consultas de Prometheus:

```promql
# Valor actual de una metrica
http_requests_total

# Filtrar por labels
http_requests_total{method="GET", status="200"}

# Regex en labels
http_requests_total{status=~"5.."}                 # todos los 5xx

# Tasa de cambio por segundo (ultimos 5 minutos)
rate(http_requests_total[5m])

# Tasa de errores como porcentaje
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100

# Percentil 99 de latencia
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))

# Percentil 50 (mediana)
histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))

# Memoria usada por instancia
process_resident_memory_bytes / 1024 / 1024         # convertir a MB

# Top 5 endpoints mas lentos
topk(5, rate(http_request_duration_seconds_sum[5m])
       / rate(http_request_duration_seconds_count[5m]))

# Agregar por label
sum by (method) (rate(http_requests_total[5m]))     # requests/s por metodo
sum by (instance) (rate(http_requests_total[5m]))   # requests/s por instancia

# Aumento total en la ultima hora
increase(http_requests_total[1h])
```

Operadores utiles:

| Operador | Que hace | Ejemplo |
|----------|----------|---------|
| `rate()` | Tasa por segundo de un counter | `rate(requests_total[5m])` |
| `increase()` | Incremento total en ventana | `increase(requests_total[1h])` |
| `sum()` | Sumar series | `sum(rate(requests_total[5m]))` |
| `avg()` | Promedio | `avg(rate(requests_total[5m]))` |
| `max()`, `min()` | Maximo/minimo | `max(cpu_usage)` |
| `histogram_quantile()` | Percentil de histogram | `histogram_quantile(0.99, ...)` |
| `topk()` | Top K series | `topk(5, rate(...))` |
| `by` | Agrupar | `sum by (method) (rate(...))` |
| `without` | Excluir labels de agrupacion | `sum without (instance) (rate(...))` |

### 2.4 Scraping y configuracion

```yaml
# prometheus.yml
global:
  scrape_interval: 15s                 # cada cuanto scrapear (default)
  evaluation_interval: 15s             # cada cuanto evaluar alertas

rule_files:
  - "alerts/*.yml"                     # archivos de reglas de alerta

scrape_configs:
  # Scrapear la propia instancia de Prometheus
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Scrapear aplicacion Spring Boot
  - job_name: 'spring-app'
    metrics_path: '/actuator/prometheus'   # endpoint de Actuator
    scrape_interval: 10s                   # override del intervalo global
    static_configs:
      - targets: ['app:8080']
        labels:
          environment: 'production'
          team: 'backend'

  # Scrapear multiples instancias
  - job_name: 'microservices'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets:
          - 'user-service:8080'
          - 'order-service:8080'
          - 'payment-service:8080'

  # Service discovery en Kubernetes (automatico)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### 2.5 Reglas de alerta

```yaml
# alerts/app-alerts.yml
groups:
  - name: aplicacion
    rules:
      # Tasa de errores alta
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          /
          sum(rate(http_server_requests_seconds_count[5m]))
          > 0.05
        for: 5m                        # debe cumplirse durante 5 minutos
        labels:
          severity: critical
        annotations:
          summary: "Tasa de errores HTTP superior al 5%"
          description: "Error rate actual: {{ $value | humanizePercentage }}"

      # Latencia alta
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(http_server_requests_seconds_bucket[5m])
          ) > 1.0
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Latencia p99 superior a 1 segundo"

      # Instancia caida
      - alert: InstanceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Instancia {{ $labels.instance }} caida"
```

---

## 3. Dashboards con Grafana

### 3.1 Conceptos de Grafana

Grafana es la herramienta de visualizacion. Se conecta a Prometheus (y otros datasources) para crear dashboards interactivos.

| Concepto | Que es |
|----------|--------|
| **Dashboard** | Coleccion de paneles organizados en una pagina |
| **Panel** | Un grafico, tabla o indicador individual |
| **Data source** | Conexion a Prometheus, Loki, Jaeger, etc. |
| **Variable** | Parametro dinamico para filtrar (ej: seleccionar servicio) |
| **Alert** | Alerta basada en una query de un panel |
| **Row** | Agrupacion horizontal de paneles |

### 3.2 Crear un dashboard

Un dashboard se puede definir como JSON (Infrastructure as Code):

```json
{
  "dashboard": {
    "title": "Spring Boot App",
    "panels": [
      {
        "title": "Requests por segundo",
        "type": "timeseries",
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count[5m])) by (method)",
            "legendFormat": "{{ method }}"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
      },
      {
        "title": "Latencia p99",
        "type": "timeseries",
        "targets": [
          {
            "expr": "histogram_quantile(0.99, rate(http_server_requests_seconds_bucket[5m]))",
            "legendFormat": "p99"
          }
        ],
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
      },
      {
        "title": "Tasa de errores",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_server_requests_seconds_count{status=~\"5..\"}[5m])) / sum(rate(http_server_requests_seconds_count[5m])) * 100"
          }
        ],
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 8 }
      }
    ]
  }
}
```

En produccion, provisiona dashboards automaticamente:

```yaml
# grafana/provisioning/dashboards/dashboards.yml
apiVersion: 1
providers:
  - name: 'default'
    orgId: 1
    folder: 'Spring Boot'
    type: file
    options:
      path: /var/lib/grafana/dashboards    # directorio con JSONs
```

### 3.3 Variables de template

Las variables permiten crear dashboards dinamicos donde el usuario selecciona el servicio o entorno:

```
Variable: $service
Query: label_values(http_server_requests_seconds_count, application)
```

Usar en las queries de los paneles:

```promql
rate(http_server_requests_seconds_count{application="$service"}[5m])
```

---

## 4. Spring Boot Actuator y Micrometer

### 4.1 Actuator endpoints

Spring Boot Actuator expone endpoints de operacion:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus   # endpoints expuestos
  endpoint:
    health:
      show-details: when-authorized               # detalles solo autenticados
```

Endpoints mas usados:

| Endpoint | Que muestra |
|----------|-------------|
| `/actuator/health` | Estado de la app y dependencias (DB, Redis, Disk) |
| `/actuator/info` | Informacion de la aplicacion (version, git commit) |
| `/actuator/metrics` | Lista de metricas disponibles |
| `/actuator/metrics/{nombre}` | Valor de una metrica especifica |
| `/actuator/prometheus` | Todas las metricas en formato Prometheus |
| `/actuator/env` | Variables de entorno y configuracion |
| `/actuator/loggers` | Niveles de log (se pueden cambiar en runtime) |

**Seguridad**: en produccion, protege los endpoints de Actuator. No expongas `/actuator/env` ni `/actuator/configprops` publicamente (pueden filtrar secretos).

### 4.2 Micrometer: la fachada de metricas

Micrometer es a las metricas lo que SLF4J es al logging: una fachada que abstrae el sistema de metricas subyacente. Escribes codigo con Micrometer y funciona con Prometheus, Datadog, CloudWatch, etc.

```xml
<!-- Dependencia para exportar a Prometheus -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

### 4.3 Tipos de metros en Micrometer

```kotlin
@Component
class MetricasEjemplo(private val meterRegistry: MeterRegistry) {

    // COUNTER: cuenta eventos (solo sube)
    private val pedidosCounter = Counter.builder("pedidos.creados")
        .description("Total de pedidos creados")
        .tag("tipo", "web")                    // label para filtrar
        .register(meterRegistry)

    fun crearPedido() {
        // ... logica de negocio
        pedidosCounter.increment()             // +1 al counter
    }

    // GAUGE: valor actual (sube y baja)
    private val colaSize = AtomicInteger(0)

    init {
        Gauge.builder("cola.tamano", colaSize) { it.get().toDouble() }
            .description("Tamano actual de la cola de procesamiento")
            .register(meterRegistry)
    }

    fun agregarACola() { colaSize.incrementAndGet() }
    fun procesarDeCola() { colaSize.decrementAndGet() }

    // TIMER: mide duracion y cuenta invocaciones
    private val procesamientoTimer = Timer.builder("pedido.procesamiento")
        .description("Tiempo de procesamiento de pedidos")
        .publishPercentiles(0.5, 0.95, 0.99)  # percentiles en cliente
        .publishPercentileHistogram()           # buckets para histogram_quantile()
        .register(meterRegistry)

    fun procesarPedido(pedido: Pedido) {
        procesamientoTimer.record {
            // ... logica que quieres medir
            Thread.sleep(100)
        }
    }

    // DISTRIBUTION SUMMARY: similar a Timer pero para valores no temporales
    private val tamanoRespuesta = DistributionSummary.builder("respuesta.tamano")
        .description("Tamano de las respuestas en bytes")
        .baseUnit("bytes")
        .publishPercentileHistogram()
        .register(meterRegistry)

    fun enviarRespuesta(bytes: ByteArray) {
        tamanoRespuesta.record(bytes.size.toDouble())
    }
}
```

### 4.4 Metricas personalizadas en Kotlin

Patron comun: medir operaciones de negocio:

```kotlin
@Service
class PagoService(
    private val meterRegistry: MeterRegistry,
    private val pagoGateway: PagoGateway
) {
    // Metricas por resultado
    private fun pagoCounter(resultado: String) = Counter.builder("pagos.total")
        .tag("resultado", resultado)
        .register(meterRegistry)

    // Timer con tags dinamicos
    fun procesarPago(monto: BigDecimal, metodo: String): ResultadoPago {
        val sample = Timer.start(meterRegistry)    // iniciar cronometro

        return try {
            val resultado = pagoGateway.cobrar(monto, metodo)
            pagoCounter("exitoso").increment()

            // Registrar monto procesado
            DistributionSummary.builder("pagos.monto")
                .tag("metodo", metodo)
                .baseUnit("USD")
                .register(meterRegistry)
                .record(monto.toDouble())

            resultado
        } catch (ex: PagoException) {
            pagoCounter("fallido").increment()
            throw ex
        } finally {
            sample.stop(Timer.builder("pagos.duracion")  // parar cronometro
                .tag("metodo", metodo)
                .register(meterRegistry))
        }
    }
}
```

Usar la anotacion `@Timed` para medir metodos sin codigo:

```kotlin
@Service
class ProductoService(private val repository: ProductoRepository) {

    @Timed(
        value = "producto.busqueda",           # nombre de la metrica
        percentiles = [0.5, 0.95, 0.99],       # percentiles
        histogram = true                        # habilitar histogram
    )
    fun buscarPorId(id: Long): Producto? {
        return repository.findById(id).orElse(null)
    }
}
```

Para que `@Timed` funcione, registra el `TimedAspect`:

```kotlin
@Configuration
class MetricsConfig {
    @Bean
    fun timedAspect(registry: MeterRegistry) = TimedAspect(registry)
}
```

### 4.5 Configuracion para Prometheus

```yaml
# application.yml
management:
  metrics:
    tags:
      application: ${spring.application.name}    # tag global en todas las metricas
    distribution:
      percentiles-histogram:
        http.server.requests: true               # histograma para latencia HTTP
      sla:
        http.server.requests: 10ms,50ms,100ms,500ms,1s   # SLA buckets
  prometheus:
    metrics:
      export:
        enabled: true
```

---

## 5. Logging con Loki

### 5.1 Arquitectura de Loki

Loki es un sistema de agregacion de logs creado por Grafana Labs. A diferencia de Elasticsearch, Loki solo indexa los **labels** (no el contenido completo), lo que lo hace mas ligero y barato.

```
Tu App --> stdout/file --> Promtail (agente) --> Loki (almacenamiento) --> Grafana (consulta)
```

Componentes:
- **Promtail**: agente que recoge logs y los envia a Loki
- **Loki**: almacena y permite consultar logs
- **Grafana**: interfaz para buscar logs con LogQL

### 5.2 Labels y mejores practicas

Loki organiza logs por **streams**, y cada stream se identifica por un conjunto unico de labels:

```
{app="user-service", env="production", level="ERROR"}  <- un stream
{app="user-service", env="production", level="INFO"}   <- otro stream
{app="order-service", env="production", level="ERROR"} <- otro stream
```

**Buenas practicas con labels**:
- Usar pocos labels con baja cardinalidad (app, env, level, namespace)
- NO usar como label: user_id, request_id, timestamps (cardinalidad altisima)
- La cardinalidad alta mata el rendimiento de Loki

Configuracion de Promtail:

```yaml
# promtail-config.yml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml        # tracking de donde se quedo leyendo

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: spring-app
    static_configs:
      - targets: [localhost]
        labels:
          app: mi-app
          env: production
          __path__: /var/log/app/*.log  # archivos a leer

    # Parsear logs de Spring Boot
    pipeline_stages:
      - multiline:
          firstline: '^\d{4}-\d{2}-\d{2}'   # nueva entrada empieza con fecha
      - regex:
          expression: '^(?P<timestamp>\S+)\s+(?P<level>\S+)\s+.*'
      - labels:
          level:                              # extraer level como label
```

### 5.3 LogQL: consultando logs

LogQL es el lenguaje de consultas de Loki (similar a PromQL):

```logql
# Todos los logs de un servicio
{app="user-service"}

# Filtrar por nivel
{app="user-service", level="ERROR"}

# Buscar texto en el contenido
{app="user-service"} |= "NullPointerException"

# Regex en el contenido
{app="user-service"} |~ "timeout|connection refused"

# Excluir texto
{app="user-service"} != "health check"

# Pipeline: parsear y filtrar campos
{app="user-service"} | json | status >= 500

# Contar errores por minuto (metricas desde logs)
count_over_time({app="user-service", level="ERROR"}[5m])

# Tasa de logs por segundo
rate({app="user-service"}[1m])

# Top 5 mensajes de error
topk(5, count_over_time({app="user-service", level="ERROR"}[1h]))
```

### 5.4 Configurar logging en Spring Boot

Formato de logs estructurado (JSON) para que Loki/Promtail puedan parsear facilmente:

```yaml
# application.yml
spring:
  application:
    name: user-service

logging:
  pattern:
    console: '{"timestamp":"%d{yyyy-MM-dd HH:mm:ss.SSS}","level":"%level","logger":"%logger","message":"%msg","thread":"%thread","traceId":"%X{traceId:-}","spanId":"%X{spanId:-}"}%n'
  level:
    root: INFO
    com.miapp: DEBUG
    org.springframework.web: INFO
```

O usar Logback con encoder JSON:

```xml
<!-- logback-spring.xml -->
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
</configuration>
```

---

## 6. Tracing distribuido

### 6.1 Conceptos fundamentales

En un sistema de microservicios, una request del usuario puede atravesar 5 o 10 servicios. El tracing distribuido te permite seguir esa request completa:

```
Request del usuario
    |
    v
[API Gateway]  ----trace-id: abc123---->  [User Service]
    |                                          |
    |                                          v
    |                                     [Database]
    |
    +----trace-id: abc123---->  [Order Service]
                                     |
                                     v
                               [Payment Service]
                                     |
                                     v
                               [External API]
```

| Concepto | Que es | Ejemplo |
|----------|--------|---------|
| **Trace** | La request completa de extremo a extremo | Todo el camino del usuario |
| **Span** | Una operacion individual dentro del trace | "Consulta a BD", "Llamada HTTP" |
| **Trace ID** | Identificador unico del trace completo | `abc123` (se propaga entre servicios) |
| **Span ID** | Identificador unico de cada span | `def456` |
| **Parent Span** | El span que invoco al actual | El span del API Gateway |
| **Context Propagation** | Mecanismo para pasar trace/span IDs entre servicios | Header `traceparent` en HTTP |

Estructura de un trace:

```
Trace: abc123
├── Span: API Gateway (250ms)
│   ├── Span: User Service (120ms)
│   │   └── Span: PostgreSQL query (80ms)
│   └── Span: Order Service (100ms)
│       └── Span: Payment Service (60ms)
│           └── Span: Stripe API call (40ms)
```

### 6.2 OpenTelemetry

OpenTelemetry (OTel) es el estandar abierto para observabilidad. Unifica metricas, logs y traces en una sola especificacion.

```
Tu App (con OTel SDK)
    |
    | OTLP (protocolo)
    v
OTel Collector (opcional, recomendado)
    |
    +---> Jaeger (traces)
    +---> Prometheus (metricas)
    +---> Loki (logs)
```

### 6.3 Auto-instrumentacion para Spring

Spring Boot 3+ con Micrometer Tracing:

```xml
<!-- pom.xml -->
<!-- Micrometer Tracing con bridge a OpenTelemetry -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>

<!-- Exportador OTLP -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
# application.yml
management:
  tracing:
    sampling:
      probability: 1.0                # 1.0 = 100% de traces (en dev)
                                       # 0.1 = 10% en produccion

  otlp:
    tracing:
      endpoint: http://otel-collector:4318/v1/traces

spring:
  application:
    name: user-service                 # aparece como service.name en traces
```

Con esta configuracion, Spring instrumenta automaticamente:
- Requests HTTP entrantes (Spring MVC/WebFlux)
- Requests HTTP salientes (RestTemplate, WebClient, RestClient)
- Consultas JDBC
- Mensajes Kafka/RabbitMQ
- Metodos anotados con `@Observed`

### 6.4 Instrumentacion manual

Para spans personalizados en operaciones de negocio:

```kotlin
@Service
class PedidoService(
    private val observationRegistry: ObservationRegistry,
    private val pagoService: PagoService
) {
    fun crearPedido(dto: PedidoDto): Pedido {
        // Crear un span personalizado
        return Observation.createNotStarted("pedido.crear", observationRegistry)
            .lowCardinalityKeyValue("tipo", dto.tipo)        # tag de baja cardinalidad
            .highCardinalityKeyValue("pedido.id", dto.id)    # tag de alta cardinalidad
            .observe {
                val pedido = validarPedido(dto)
                pagoService.cobrar(pedido)       # genera sub-span automatico
                guardarPedido(pedido)
                pedido
            }
    }
}
```

Usando la anotacion `@Observed`:

```kotlin
@Service
class InventarioService {

    @Observed(
        name = "inventario.verificar",
        contextualName = "verificar-stock",
        lowCardinalityKeyValues = ["operacion", "verificacion"]
    )
    fun verificarStock(productoId: Long, cantidad: Int): Boolean {
        // Este metodo genera un span automaticamente
        return true
    }
}
```

Registrar el aspecto para que `@Observed` funcione:

```kotlin
@Configuration
class ObservabilityConfig {
    @Bean
    fun observedAspect(registry: ObservationRegistry) = ObservedAspect(registry)
}
```

### 6.5 Jaeger y Zipkin

**Jaeger** es el backend mas popular para traces. Desarrollado por Uber:

```yaml
# docker-compose.yml (Jaeger all-in-one para desarrollo)
services:
  jaeger:
    image: jaegertracing/all-in-one:1.52
    ports:
      - "16686:16686"                  # UI web
      - "4317:4317"                    # OTLP gRPC
      - "4318:4318"                    # OTLP HTTP
    environment:
      COLLECTOR_OTLP_ENABLED: "true"
```

**Zipkin** es una alternativa mas simple:

```yaml
services:
  zipkin:
    image: openzipkin/zipkin:3
    ports:
      - "9411:9411"                    # UI + API
```

Configuracion de Spring para Zipkin:

```yaml
management:
  zipkin:
    tracing:
      endpoint: http://zipkin:9411/api/v2/spans
```

---

## 7. Alerting

### 7.1 Alertmanager

Alertmanager recibe alertas de Prometheus y las enruta al destino correcto:

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m

route:
  receiver: 'slack-default'            # receptor por defecto
  group_by: ['alertname', 'severity']  # agrupar alertas similares
  group_wait: 30s                      # esperar 30s para agrupar
  group_interval: 5m                   # intervalo entre grupos
  repeat_interval: 4h                  # repetir cada 4h si no se resuelve

  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'   # criticas van a PagerDuty
      repeat_interval: 1h

    - match:
        severity: warning
      receiver: 'slack-warnings'       # warnings van a Slack

receivers:
  - name: 'slack-default'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#alertas'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ .CommonAnnotations.description }}'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'tu-service-key'
        severity: critical

  - name: 'slack-warnings'
    slack_configs:
      - api_url: 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
        channel: '#alertas-warning'
```

### 7.2 Integracion con Slack y PagerDuty

Template de Slack personalizado:

```yaml
slack_configs:
  - api_url: 'https://hooks.slack.com/services/XXX'
    channel: '#alertas'
    title: '{{ if eq .Status "firing" }}ALERTA{{ else }}RESUELTA{{ end }}: {{ .GroupLabels.alertname }}'
    text: |
      *Severidad*: {{ .CommonLabels.severity }}
      *Descripcion*: {{ .CommonAnnotations.description }}
      *Desde*: {{ .StartsAt.Format "2006-01-02 15:04:05" }}
      *Instancias afectadas*:
      {{ range .Alerts }}
        - {{ .Labels.instance }}
      {{ end }}
    send_resolved: true                # notificar cuando se resuelve
```

### 7.3 Buenas practicas de alerting

| Practica | Por que |
|----------|---------|
| Alertar sobre sintomas, no causas | "Error rate > 5%" es mejor que "CPU > 80%" |
| Usar `for` para evitar falsos positivos | `for: 5m` evita picos momentaneos |
| Clasificar por severidad | critical = despierta a alguien, warning = revisar manana |
| Incluir runbooks en las anotaciones | Link a documentacion de como resolver |
| No alertar sobre todo | Demasiadas alertas = nadie las mira (alert fatigue) |
| Alertar sobre SLOs | "SLO de disponibilidad < 99.9% en 1h" |

---

## 8. Stack completo con Docker Compose

```yaml
# docker-compose.yml
services:
  # Tu aplicacion Spring Boot
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      MANAGEMENT_OTLP_TRACING_ENDPOINT: http://otel-collector:4318/v1/traces

  # Prometheus: metricas
  prometheus:
    image: prom/prometheus:v2.48.0
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts:/etc/prometheus/alerts
    ports:
      - "9090:9090"

  # Grafana: visualizacion
  grafana:
    image: grafana/grafana:10.2.0
    volumes:
      - ./grafana/provisioning:/etc/grafana/provisioning
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin

  # Loki: logs
  loki:
    image: grafana/loki:2.9.0
    ports:
      - "3100:3100"

  # Promtail: agente de logs
  promtail:
    image: grafana/promtail:2.9.0
    volumes:
      - ./promtail/config.yml:/etc/promtail/config.yml
      - /var/log:/var/log

  # Jaeger: traces
  jaeger:
    image: jaegertracing/all-in-one:1.52
    ports:
      - "16686:16686"
      - "4317:4317"
      - "4318:4318"
    environment:
      COLLECTOR_OTLP_ENABLED: "true"

  # OTel Collector (hub central de telemetria)
  otel-collector:
    image: otel/opentelemetry-collector-contrib:0.91.0
    volumes:
      - ./otel/otel-collector-config.yml:/etc/otelcol-contrib/config.yaml
    ports:
      - "4317:4317"                    # OTLP gRPC
      - "4318:4318"                    # OTLP HTTP

  # Alertmanager
  alertmanager:
    image: prom/alertmanager:v0.26.0
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    ports:
      - "9093:9093"
```

Configuracion del OTel Collector:

```yaml
# otel/otel-collector-config.yml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

exporters:
  jaeger:
    endpoint: jaeger:14250
    tls:
      insecure: true
  prometheus:
    endpoint: 0.0.0.0:8889

service:
  pipelines:
    traces:
      receivers: [otlp]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      exporters: [prometheus]
```

---

## 9. Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| **Metricas** | Numeros agregados en el tiempo | Prometheus scrapea `/actuator/prometheus` |
| **Logs** | Eventos textuales con timestamp | Promtail envia a Loki, consulta con LogQL |
| **Traces** | Camino de una request entre servicios | OpenTelemetry auto-instrumenta, Jaeger visualiza |
| **Prometheus** | Base de datos time-series + scraper | `prometheus.yml` con scrape_configs |
| **PromQL** | Lenguaje de consultas de Prometheus | `rate()`, `histogram_quantile()`, `sum by()` |
| **Grafana** | Dashboards y visualizacion | Datasources + paneles con queries |
| **Actuator** | Endpoints operacionales de Spring Boot | `/actuator/health`, `/actuator/prometheus` |
| **Micrometer** | Fachada de metricas para Java/Kotlin | Counter, Gauge, Timer, DistributionSummary |
| **Loki** | Agregacion de logs (como Prometheus pero para logs) | Labels de baja cardinalidad + LogQL |
| **OpenTelemetry** | Estandar abierto de observabilidad | SDK + auto-instrumentacion + exportadores |
| **Alertmanager** | Enrutamiento de alertas | Routes por severidad a Slack/PagerDuty |

---

## 10. Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | [Prometheus y Grafana](ejercicio-01-prometheus-grafana/) | Levantar Prometheus y Grafana con Docker Compose, configurar scraping de una app Spring Boot y crear un dashboard con metricas HTTP | prometheus.yml, Grafana datasource, PromQL, dashboards |
| 02 | [Spring Actuator y metricas custom](ejercicio-02-spring-actuator-metrics/) | Configurar Actuator, crear metricas de negocio con Micrometer (counters, timers, gauges) y exponerlas en formato Prometheus | Actuator, Micrometer, @Timed, Counter, Timer, Gauge |
| 03 | [Logging con Loki](ejercicio-03-logging-loki/) | Configurar logging estructurado en JSON, levantar Loki + Promtail, y consultar logs desde Grafana con LogQL | logback JSON, Promtail, Loki, LogQL |
| 04 | [Tracing distribuido](ejercicio-04-tracing-distribuido/) | Instrumentar dos microservicios con OpenTelemetry, propagar contexto entre ellos y visualizar traces en Jaeger | Micrometer Tracing, OTLP, Jaeger, spans, context propagation |

Haz los ejercicios en orden. Cada uno construye sobre los conceptos del anterior.

---

## 11. Recursos

- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [Spring Boot Actuator Reference](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html)
- [Micrometer Documentation](https://micrometer.io/docs)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [LogQL Reference](https://grafana.com/docs/loki/latest/logql/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [Spring Boot Observability Guide](https://spring.io/blog/2022/10/12/observability-with-spring-boot-3)
