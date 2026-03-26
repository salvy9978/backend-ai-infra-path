# Nivel 14 — Orquestacion Avanzada de IA

> Ya sabes conectar un LLM a tu aplicacion Spring. Ahora necesitas ir mas alla: encadenar llamadas, estructurar respuestas, coordinar multiples agentes y, sobre todo, probar que todo funciona. Este nivel te convierte en alguien que construye sistemas de IA, no solo demos.

---

## Contenido

- [1. LangChain4j: por que otro framework](#1-langchain4j-por-que-otro-framework)
  - [1.1 Spring AI vs LangChain4j](#11-spring-ai-vs-langchain4j)
  - [1.2 Configuracion basica](#12-configuracion-basica)
  - [1.3 AI Services: la abstraccion central](#13-ai-services-la-abstraccion-central)
  - [1.4 Memoria de conversacion](#14-memoria-de-conversacion)
  - [1.5 Cuando usar cual](#15-cuando-usar-cual)
- [2. Chains y Workflows](#2-chains-y-workflows)
  - [2.1 Que es un chain](#21-que-es-un-chain)
  - [2.2 Chains secuenciales](#22-chains-secuenciales)
  - [2.3 Chains con branching](#23-chains-con-branching)
  - [2.4 Ejecucion paralela](#24-ejecucion-paralela)
  - [2.5 Patrones de orquestacion avanzados](#25-patrones-de-orquestacion-avanzados)
- [3. Structured Outputs](#3-structured-outputs)
  - [3.1 El problema de las respuestas libres](#31-el-problema-de-las-respuestas-libres)
  - [3.2 JSON Mode](#32-json-mode)
  - [3.3 Schema enforcement con LangChain4j](#33-schema-enforcement-con-langchain4j)
  - [3.4 Schema enforcement con Spring AI](#34-schema-enforcement-con-spring-ai)
  - [3.5 Validacion y fallback](#35-validacion-y-fallback)
- [4. Sistemas Multi-Agente](#4-sistemas-multi-agente)
  - [4.1 Que es un agente](#41-que-es-un-agente)
  - [4.2 Comunicacion entre agentes](#42-comunicacion-entre-agentes)
  - [4.3 Patron de delegacion](#43-patron-de-delegacion)
  - [4.4 Orquestador central vs agentes autonomos](#44-orquestador-central-vs-agentes-autonomos)
- [5. Evaluacion de Sistemas de IA](#5-evaluacion-de-sistemas-de-ia)
  - [5.1 Por que evaluar es critico](#51-por-que-evaluar-es-critico)
  - [5.2 Tipos de evaluacion](#52-tipos-de-evaluacion)
  - [5.3 Metricas de calidad](#53-metricas-de-calidad)
  - [5.4 Evaluacion humana](#54-evaluacion-humana)
  - [5.5 Evaluacion automatizada (LLM-as-judge)](#55-evaluacion-automatizada-llm-as-judge)
- [6. Testing de Aplicaciones de IA](#6-testing-de-aplicaciones-de-ia)
  - [6.1 El problema fundamental](#61-el-problema-fundamental)
  - [6.2 Tests deterministicos](#62-tests-deterministicos)
  - [6.3 Snapshot testing](#63-snapshot-testing)
  - [6.4 LLM-as-judge en tests](#64-llm-as-judge-en-tests)
  - [6.5 Tests de integracion con Testcontainers](#65-tests-de-integracion-con-testcontainers)
- [7. Caching y Optimizacion](#7-caching-y-optimizacion)
  - [7.1 Semantic caching](#71-semantic-caching)
  - [7.2 Response deduplication](#72-response-deduplication)
  - [7.3 Estrategias de cache por tipo de consulta](#73-estrategias-de-cache-por-tipo-de-consulta)
- [8. Fine-tuning: cuando tiene sentido](#8-fine-tuning-cuando-tiene-sentido)
  - [8.1 Fine-tuning vs prompting vs RAG](#81-fine-tuning-vs-prompting-vs-rag)
  - [8.2 Preparacion de datos](#82-preparacion-de-datos)
  - [8.3 El flujo completo](#83-el-flujo-completo)
- [9. Resumen](#9-resumen)
- [10. Ejercicios](#10-ejercicios)
- [11. Recursos](#11-recursos)

---

## 1. LangChain4j: por que otro framework

### 1.1 Spring AI vs LangChain4j

Spring AI y LangChain4j resuelven problemas similares, pero desde filosofias distintas:

| Aspecto | Spring AI | LangChain4j |
|---------|-----------|-------------|
| **Filosofia** | Spring-native, configuracion por convencion | Framework independiente, mas explicitio |
| **Integracion Spring** | Nativa (auto-configuration, starters) | Buena (tiene spring-boot-starter propio) |
| **Abstracciones** | ChatClient, Advisors, funciones | AI Services, Chains, Tools |
| **Madurez RAG** | Buena (ETL pipeline, VectorStore) | Muy buena (EmbeddingStore, Retrievers) |
| **Structured Output** | BeanOutputConverter | AI Services con tipos de retorno |
| **Multi-agente** | Basico (manual) | Mejor soporte nativo |
| **Comunidad** | Creciendo rapido (respaldada por Pivotal) | Madura (port de LangChain Python) |

En la practica, muchos proyectos usan **ambos**. Spring AI para la integracion nativa con Spring Boot y LangChain4j cuando necesitan orquestacion mas compleja.

### 1.2 Configuracion basica

Dependencia Maven:

```xml
<!-- LangChain4j core -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-spring-boot-starter</artifactId>
    <version>0.36.2</version>
</dependency>

<!-- Proveedor OpenAI -->
<dependency>
    <groupId>dev.langchain4j</groupId>
    <artifactId>langchain4j-open-ai-spring-boot-starter</artifactId>
    <version>0.36.2</version>
</dependency>
```

Configuracion en `application.yml`:

```yaml
langchain4j:
  open-ai:
    chat-model:
      api-key: ${OPENAI_API_KEY}
      model-name: gpt-4o-mini
      temperature: 0.7
      max-tokens: 2000
      log-requests: true    # util en desarrollo
      log-responses: true
```

### 1.3 AI Services: la abstraccion central

La magia de LangChain4j esta en los **AI Services**. Defines una interfaz y LangChain4j genera la implementacion:

```kotlin
// Solo defines la interfaz, LangChain4j implementa todo
interface AsistenteDeViajes {

    @SystemMessage("Eres un asistente de viajes experto. Responde en español.")
    fun recomendar(
        @UserMessage("Recomienda destinos para: {{it}}")
        preferencias: String
    ): String
}

// Configuracion como bean de Spring
@Configuration
class AiConfig {

    @Bean
    fun asistenteDeViajes(chatModel: ChatLanguageModel): AsistenteDeViajes {
        return AiServices.builder(AsistenteDeViajes::class.java)
            .chatLanguageModel(chatModel)
            .build()
    }
}

// Uso en un controller: inyectas como cualquier bean
@RestController
class ViajesController(private val asistente: AsistenteDeViajes) {

    @GetMapping("/recomendar")
    fun recomendar(@RequestParam preferencias: String): String {
        return asistente.recomendar(preferencias)
    }
}
```

### 1.4 Memoria de conversacion

LangChain4j soporta memoria conversacional de forma nativa:

```kotlin
@Bean
fun asistenteConMemoria(chatModel: ChatLanguageModel): AsistenteDeViajes {
    return AiServices.builder(AsistenteDeViajes::class.java)
        .chatLanguageModel(chatModel)
        .chatMemory(MessageWindowChatMemory.withMaxMessages(20))  // ultimos 20 mensajes
        .build()
}
```

Para memoria por usuario (cada sesion independiente):

```kotlin
interface AsistentePersonal {

    fun chat(@MemoryId userId: String, @UserMessage mensaje: String): String
}

@Bean
fun asistentePersonal(chatModel: ChatLanguageModel): AsistentePersonal {
    return AiServices.builder(AsistentePersonal::class.java)
        .chatLanguageModel(chatModel)
        .chatMemoryProvider { memoryId ->
            // Cada usuario tiene su propia memoria
            MessageWindowChatMemory.builder()
                .id(memoryId)
                .maxMessages(30)
                .build()
        }
        .build()
}
```

### 1.5 Cuando usar cual

| Situacion | Recomendacion |
|-----------|--------------|
| Proyecto 100% Spring, llamadas simples | Spring AI |
| Necesitas chains complejos | LangChain4j |
| RAG basico con pgvector | Spring AI |
| RAG avanzado con re-ranking | LangChain4j |
| Multi-agente | LangChain4j |
| Ambas cosas | Ambos (no son mutuamente excluyentes) |

---

## 2. Chains y Workflows

### 2.1 Que es un chain

Un chain es una secuencia de operaciones donde la salida de una se convierte en la entrada de la siguiente. En el contexto de IA, un chain tipico seria:

```
Input usuario -> Reformular pregunta -> Buscar en BD vectorial -> Generar respuesta -> Formatear salida
```

Cada paso puede involucrar un LLM, una busqueda en base de datos, una llamada a API externa, o logica de negocio pura.

### 2.2 Chains secuenciales

El patron mas simple: ejecutar pasos uno tras otro.

```kotlin
@Service
class ResearchChain(
    private val chatModel: ChatLanguageModel,
    private val searchService: WebSearchService
) {

    // Paso 1: el LLM genera las queries de busqueda
    private val queryGenerator = AiServices.builder(QueryGenerator::class.java)
        .chatLanguageModel(chatModel)
        .build()

    // Paso 2: otro LLM sintetiza los resultados
    private val synthesizer = AiServices.builder(Synthesizer::class.java)
        .chatLanguageModel(chatModel)
        .build()

    fun investigar(tema: String): ResearchResult {
        // Paso 1: generar queries de busqueda
        val queries = queryGenerator.generarQueries(tema)

        // Paso 2: ejecutar busquedas (no es LLM, es logica)
        val resultados = queries.map { query ->
            searchService.buscar(query)
        }.flatten()

        // Paso 3: sintetizar con LLM
        val sintesis = synthesizer.sintetizar(
            tema = tema,
            fuentes = resultados.joinToString("\n---\n")
        )

        return ResearchResult(tema, queries, resultados, sintesis)
    }
}

interface QueryGenerator {
    @UserMessage("Genera 3 queries de busqueda para investigar: {{it}}")
    fun generarQueries(tema: String): List<String>
}

interface Synthesizer {
    @SystemMessage("Sintetiza informacion de multiples fuentes en un resumen coherente.")
    @UserMessage("Tema: {{tema}}\n\nFuentes:\n{{fuentes}}")
    fun sintetizar(@V("tema") tema: String, @V("fuentes") fuentes: String): String
}
```

### 2.3 Chains con branching

A veces necesitas ejecutar caminos diferentes segun el input:

```kotlin
@Service
class RoutingChain(
    private val classifier: IntentClassifier,
    private val faqHandler: FaqHandler,
    private val ticketCreator: TicketCreator,
    private val escalator: HumanEscalator
) {

    fun procesar(mensaje: String): String {
        // Paso 1: clasificar la intencion
        val intent = classifier.clasificar(mensaje)

        // Paso 2: rutear segun la clasificacion
        return when (intent) {
            Intent.FAQ -> faqHandler.responder(mensaje)
            Intent.QUEJA -> ticketCreator.crearTicket(mensaje)
            Intent.URGENTE -> escalator.escalar(mensaje)
            Intent.OTRO -> "No pude entender tu solicitud. Intenta reformularla."
        }
    }
}

interface IntentClassifier {
    @SystemMessage("""
        Clasifica el mensaje del usuario en una de estas categorias:
        - FAQ: pregunta frecuente sobre productos o servicios
        - QUEJA: el usuario tiene un problema o queja
        - URGENTE: situacion critica que necesita atencion inmediata
        - OTRO: no encaja en ninguna categoria
    """)
    fun clasificar(@UserMessage mensaje: String): Intent
}

enum class Intent { FAQ, QUEJA, URGENTE, OTRO }
```

### 2.4 Ejecucion paralela

Cuando los pasos son independientes, ejecutalos en paralelo para reducir latencia:

```kotlin
@Service
class ParallelAnalysisChain(
    private val sentimentAnalyzer: SentimentAnalyzer,
    private val topicExtractor: TopicExtractor,
    private val summaryGenerator: SummaryGenerator
) {

    // Tres analisis independientes en paralelo
    fun analizar(texto: String): AnalisisCompleto = runBlocking {
        val sentimiento = async { sentimentAnalyzer.analizar(texto) }
        val temas = async { topicExtractor.extraer(texto) }
        val resumen = async { summaryGenerator.resumir(texto) }

        AnalisisCompleto(
            sentimiento = sentimiento.await(),
            temas = temas.await(),
            resumen = resumen.await()
        )
    }
}

data class AnalisisCompleto(
    val sentimiento: Sentiment,
    val temas: List<String>,
    val resumen: String
)
```

### 2.5 Patrones de orquestacion avanzados

**Map-Reduce**: procesar documentos largos dividiendolos en chunks.

```kotlin
@Service
class MapReduceSummarizer(private val chatModel: ChatLanguageModel) {

    private val chunkSummarizer = AiServices.builder(ChunkSummarizer::class.java)
        .chatLanguageModel(chatModel)
        .build()

    private val finalSummarizer = AiServices.builder(FinalSummarizer::class.java)
        .chatLanguageModel(chatModel)
        .build()

    fun resumirDocumentoLargo(documento: String): String {
        // Map: dividir y resumir cada chunk
        val chunks = documento.chunked(4000)  // respetar limite de tokens
        val resumenesParciales = chunks.map { chunk ->
            chunkSummarizer.resumir(chunk)
        }

        // Reduce: combinar resumenes parciales en uno final
        return finalSummarizer.combinar(
            resumenesParciales.joinToString("\n\n---\n\n")
        )
    }
}

interface ChunkSummarizer {
    @UserMessage("Resume este fragmento en 2-3 oraciones:\n\n{{it}}")
    fun resumir(chunk: String): String
}

interface FinalSummarizer {
    @UserMessage("Combina estos resumenes parciales en un resumen final coherente:\n\n{{it}}")
    fun combinar(resumenes: String): String
}
```

**Refinamiento iterativo**: mejorar la salida en multiples pasadas.

```kotlin
fun refinar(borrador: String, criterios: List<String>, maxIteraciones: Int = 3): String {
    var actual = borrador
    for (i in 1..maxIteraciones) {
        val evaluacion = evaluador.evaluar(actual, criterios)
        if (evaluacion.cumpleTodos) break        // ya es suficiente
        actual = refinador.mejorar(actual, evaluacion.feedback)
    }
    return actual
}
```

---

## 3. Structured Outputs

### 3.1 El problema de las respuestas libres

Los LLMs generan texto libre. Pero tu aplicacion necesita datos estructurados para procesarlos. Si le pides a un LLM "extrae el nombre y email del siguiente texto", puede responder:

```
El nombre es Juan y su email es juan@example.com
```

O puede responder:

```
Nombre: Juan
Email: juan@example.com
```

O incluso:

```
{"nombre": "Juan", "email": "juan@example.com"}
```

Ninguno de estos formatos es fiable para parsear programaticamente. Necesitas **garantias** de estructura.

### 3.2 JSON Mode

La forma mas directa: pedirle al modelo que responda en JSON.

```kotlin
// Con Spring AI
@Bean
fun chatClient(builder: ChatClient.Builder): ChatClient {
    return builder
        .defaultOptions(
            OpenAiChatOptions.builder()
                .withResponseFormat(ResponseFormat.JSON)  // fuerza JSON
                .build()
        )
        .build()
}

// En el prompt debes especificar la estructura esperada
val respuesta = chatClient.prompt()
    .user("""
        Extrae la informacion del siguiente texto y responde en JSON
        con el formato: {"nombre": "...", "email": "...", "telefono": "..."}
        
        Texto: "Soy Maria Lopez, me puedes contactar en maria@test.com o al 555-1234"
    """)
    .call()
    .content()

// respuesta sera algo como: {"nombre": "Maria Lopez", "email": "maria@test.com", "telefono": "555-1234"}
```

**Problema**: JSON Mode solo garantiza que la respuesta sea JSON valido, pero NO garantiza que siga tu esquema. El modelo puede devolver campos diferentes a los que esperas.

### 3.3 Schema enforcement con LangChain4j

LangChain4j resuelve esto con AI Services que retornan tipos fuertes:

```kotlin
// Defines exactamente la estructura que quieres
data class ContactoExtraido(
    val nombre: String,
    val email: String?,
    val telefono: String?,
    val confianza: Double        // 0.0 a 1.0
)

interface ExtractorDeContactos {
    @UserMessage("Extrae la informacion de contacto del siguiente texto:\n\n{{it}}")
    fun extraer(texto: String): ContactoExtraido  // retorno tipado
}

// LangChain4j automaticamente:
// 1. Genera el prompt con el schema JSON del data class
// 2. Pide al LLM que responda en ese formato
// 3. Parsea la respuesta al data class
// 4. Lanza excepcion si no puede parsear

val contacto = extractor.extraer("Soy Maria Lopez, maria@test.com, 555-1234")
// contacto.nombre = "Maria Lopez"
// contacto.email = "maria@test.com"
// contacto.telefono = "555-1234"
// contacto.confianza = 0.95
```

Tambien funciona con listas y tipos complejos:

```kotlin
data class Producto(
    val nombre: String,
    val precio: Double,
    val categoria: Categoria
)

enum class Categoria { ELECTRONICA, ROPA, ALIMENTOS, OTROS }

interface CatalogExtractor {
    @UserMessage("Extrae los productos mencionados en:\n\n{{it}}")
    fun extraer(texto: String): List<Producto>  // lista de objetos tipados
}
```

### 3.4 Schema enforcement con Spring AI

Spring AI tiene su propio mecanismo con `BeanOutputConverter`:

```kotlin
@Service
class ExtractionService(private val chatClient: ChatClient) {

    // Spring AI convierte automaticamente la respuesta al tipo indicado
    fun extraerContacto(texto: String): ContactoExtraido {
        return chatClient.prompt()
            .user("Extrae la informacion de contacto de: $texto")
            .call()
            .entity(ContactoExtraido::class.java)  // conversion automatica
    }

    // Tambien listas
    fun extraerProductos(texto: String): List<Producto> {
        return chatClient.prompt()
            .user("Extrae los productos de: $texto")
            .call()
            .entity(object : ParameterizedTypeReference<List<Producto>>() {})
    }
}
```

### 3.5 Validacion y fallback

Nunca confies ciegamente en la salida del LLM, ni siquiera con schema enforcement:

```kotlin
@Service
class SafeExtractor(
    private val extractor: ExtractorDeContactos,
    private val chatModel: ChatLanguageModel
) {

    fun extraerConValidacion(texto: String): ContactoExtraido? {
        return try {
            val resultado = extractor.extraer(texto)

            // Validacion adicional de negocio
            require(resultado.nombre.isNotBlank()) { "Nombre vacio" }
            resultado.email?.let {
                require(it.contains("@")) { "Email invalido: $it" }
            }

            resultado
        } catch (e: Exception) {
            logger.warn("Extraccion fallida, reintentando con prompt mas explicito", e)
            // Fallback: reintentar con un prompt mas detallado
            retryConPromptExplicito(texto)
        }
    }

    private fun retryConPromptExplicito(texto: String): ContactoExtraido? {
        // Segundo intento con instrucciones mas claras
        return try {
            extractor.extraer("IMPORTANTE: extrae SOLO datos que aparezcan explicitamente. $texto")
        } catch (e: Exception) {
            logger.error("Extraccion fallida tras retry", e)
            null  // devuelve null si no puede
        }
    }
}
```

---

## 4. Sistemas Multi-Agente

### 4.1 Que es un agente

Un agente es un LLM con acceso a **herramientas** (tools) y capacidad de **decidir** cuales usar. A diferencia de un chain fijo, un agente es dinamico: evalua la situacion y elige su propio camino.

```kotlin
// Definir herramientas que el agente puede usar
@Component
class HerramientasDeInvestigacion {

    @Tool("Busca informacion en la base de datos de productos")
    fun buscarProducto(nombre: String): String {
        val productos = productoRepo.findByNombreContaining(nombre)
        return productos.joinToString("\n") { "${it.nombre}: ${it.precio}€" }
    }

    @Tool("Busca el historial de pedidos de un cliente")
    fun historialCliente(clienteId: Long): String {
        val pedidos = pedidoRepo.findByClienteId(clienteId)
        return pedidos.joinToString("\n") { "Pedido #${it.id}: ${it.total}€ - ${it.estado}" }
    }

    @Tool("Calcula descuento segun el tipo de cliente")
    fun calcularDescuento(clienteId: Long, montoBase: Double): Double {
        val cliente = clienteRepo.findById(clienteId).orElseThrow()
        return when (cliente.tipo) {
            TipoCliente.VIP -> montoBase * 0.20
            TipoCliente.FRECUENTE -> montoBase * 0.10
            TipoCliente.NUEVO -> montoBase * 0.05
        }
    }
}

// El agente decide que herramientas usar segun la pregunta
@Bean
fun agenteDeVentas(
    chatModel: ChatLanguageModel,
    herramientas: HerramientasDeInvestigacion
): AgenteDeVentas {
    return AiServices.builder(AgenteDeVentas::class.java)
        .chatLanguageModel(chatModel)
        .tools(herramientas)  // el agente puede usar estas herramientas
        .chatMemory(MessageWindowChatMemory.withMaxMessages(20))
        .build()
}

interface AgenteDeVentas {
    @SystemMessage("""
        Eres un agente de ventas. Tienes acceso a herramientas para buscar productos,
        consultar historiales y calcular descuentos. Usa las herramientas que necesites
        para responder al cliente de forma completa.
    """)
    fun atender(@UserMessage consulta: String): String
}
```

### 4.2 Comunicacion entre agentes

En sistemas complejos, diferentes agentes se especializan en diferentes tareas y se comunican:

```kotlin
// Agente 1: analiza la consulta del cliente
interface AgenteClasificador {
    @SystemMessage("Clasifica la consulta y decide que departamento debe atenderla.")
    fun clasificar(@UserMessage consulta: String): ClasificacionConsulta
}

data class ClasificacionConsulta(
    val departamento: Departamento,
    val prioridad: Prioridad,
    val resumenParaAgente: String  // contexto para el siguiente agente
)

// Agente 2: resuelve consultas tecnicas
interface AgenteTecnico {
    @SystemMessage("Eres un experto tecnico. Resuelve problemas tecnicos de los clientes.")
    fun resolver(@UserMessage contexto: String): String
}

// Agente 3: gestiona quejas y compensaciones
interface AgenteCompensaciones {
    @SystemMessage("Gestionas quejas. Puedes ofrecer descuentos hasta 20% o reembolsos parciales.")
    fun gestionar(@UserMessage contexto: String): RespuestaCompensacion
}
```

### 4.3 Patron de delegacion

Un agente orquestador coordina a los agentes especializados:

```kotlin
@Service
class OrquestadorMultiAgente(
    private val clasificador: AgenteClasificador,
    private val tecnico: AgenteTecnico,
    private val compensaciones: AgenteCompensaciones,
    private val ventas: AgenteDeVentas
) {

    fun procesarConsulta(consulta: String): RespuestaFinal {
        // Paso 1: clasificar
        val clasificacion = clasificador.clasificar(consulta)

        // Paso 2: delegar al agente adecuado
        val respuesta = when (clasificacion.departamento) {
            Departamento.TECNICO -> tecnico.resolver(clasificacion.resumenParaAgente)
            Departamento.QUEJAS -> compensaciones.gestionar(clasificacion.resumenParaAgente).mensaje
            Departamento.VENTAS -> ventas.atender(clasificacion.resumenParaAgente)
            Departamento.GENERAL -> "Un representante se pondra en contacto contigo."
        }

        return RespuestaFinal(
            departamento = clasificacion.departamento,
            prioridad = clasificacion.prioridad,
            respuesta = respuesta
        )
    }
}
```

### 4.4 Orquestador central vs agentes autonomos

Dos arquitecturas principales:

```
ORQUESTADOR CENTRAL:                AGENTES AUTONOMOS:

    [Orquestador]                   [Agente A] <---> [Agente B]
     /    |    \                        |                 |
    v     v     v                      v                 v
[Ag.A] [Ag.B] [Ag.C]              [Agente C] <---> [Agente D]

- Control centralizado              - Descentralizado
- Facil de debuggear                - Mas resiliente
- Un punto de fallo                 - Dificil de razonar
- Bueno para flujos predecibles     - Bueno para tareas exploratorias
```

Para la mayoria de aplicaciones empresariales, **el orquestador central es la mejor opcion**. Es mas predecible, mas facil de monitorear y mas facil de explicar a los stakeholders.

---

## 5. Evaluacion de Sistemas de IA

### 5.1 Por que evaluar es critico

Un sistema de IA sin evaluacion es como un servicio sin tests: funciona hasta que no funciona, y no tienes forma de saber cuando se rompio.

Los LLMs pueden:
- Alucinar datos que parecen correctos pero son inventados
- Degradarse silenciosamente al cambiar de modelo o version
- Comportarse diferente con distintos tipos de input
- Responder correctamente el 95% de las veces (el 5% restante te explota en produccion)

### 5.2 Tipos de evaluacion

| Tipo | Que mide | Cuando usarlo |
|------|----------|---------------|
| **Benchmarks** | Rendimiento en datasets estandar | Al elegir modelo |
| **Evaluacion humana** | Calidad subjetiva percibida | Antes de lanzar a produccion |
| **Evaluacion automatizada** | Metricas computables (BLEU, ROUGE, cosine sim) | En CI/CD, tras cada cambio |
| **LLM-as-judge** | Un LLM evalua a otro LLM | Cuando la evaluacion humana no escala |
| **A/B testing** | Preferencia de usuarios reales | En produccion con trafico real |

### 5.3 Metricas de calidad

Para RAG (Retrieval Augmented Generation):

| Metrica | Que mide | Como se calcula |
|---------|----------|-----------------|
| **Faithfulness** | La respuesta se basa en los documentos recuperados | % de claims verificables con el contexto |
| **Relevance** | Los documentos recuperados son relevantes a la pregunta | Cosine similarity entre query y docs |
| **Answer correctness** | La respuesta es factualmente correcta | Comparacion con ground truth |
| **Context precision** | Los docs relevantes estan al principio | Precision@K sobre los docs recuperados |
| **Context recall** | Se recuperaron todos los docs relevantes | Recall sobre docs relevantes conocidos |

### 5.4 Evaluacion humana

Crea un dataset de evaluacion con pares (input, respuesta_esperada):

```kotlin
data class EvaluationCase(
    val id: String,
    val input: String,
    val expectedOutput: String,         // respuesta "gold standard"
    val acceptableCriteria: List<String> // criterios que debe cumplir
)

// Ejemplo de dataset de evaluacion
val evaluationDataset = listOf(
    EvaluationCase(
        id = "FAQ-001",
        input = "Cual es la politica de devolucion?",
        expectedOutput = "Las devoluciones se aceptan dentro de los 30 dias...",
        acceptableCriteria = listOf(
            "Menciona el plazo de 30 dias",
            "Indica que el producto debe estar sin uso",
            "No inventa politicas que no existen"
        )
    )
)
```

### 5.5 Evaluacion automatizada (LLM-as-judge)

Usa un LLM mas potente para evaluar las respuestas de un LLM mas economico:

```kotlin
interface EvaluadorDeCalidad {
    @SystemMessage("""
        Eres un evaluador de calidad de respuestas de IA.
        Evalua la respuesta segun estos criterios:
        1. Precision factual (0-10)
        2. Completitud (0-10)
        3. Relevancia (0-10)
        4. Claridad (0-10)
        Incluye justificacion para cada puntuacion.
    """)
    @UserMessage("""
        Pregunta: {{pregunta}}
        Respuesta esperada: {{esperada}}
        Respuesta del sistema: {{actual}}
    """)
    fun evaluar(
        @V("pregunta") pregunta: String,
        @V("esperada") esperada: String,
        @V("actual") actual: String
    ): EvaluacionCalidad
}

data class EvaluacionCalidad(
    val precisionFactual: Int,
    val completitud: Int,
    val relevancia: Int,
    val claridad: Int,
    val justificacion: String
) {
    val promedioGeneral: Double
        get() = (precisionFactual + completitud + relevancia + claridad) / 4.0
}
```

---

## 6. Testing de Aplicaciones de IA

### 6.1 El problema fundamental

Los tests tradicionales verifican resultados exactos: `assertEquals("hola", resultado)`. Pero un LLM puede responder "Hola", "hola!", "Hola, como estas?" y todas son correctas.

**No puedes testear salidas de LLM con igualdad exacta.** Necesitas estrategias diferentes.

### 6.2 Tests deterministicos

Lo que SI puedes testear de forma deterministica:

```kotlin
@SpringBootTest
class ChainServiceTest {

    @MockBean
    lateinit var chatModel: ChatLanguageModel

    @Autowired
    lateinit var chainService: ResearchChain

    @Test
    fun `el chain llama al LLM con el prompt correcto`() {
        // Mock del LLM: respuesta fija y predecible
        whenever(chatModel.generate(any())).thenReturn(
            AiMessage.from("""["query 1", "query 2"]""")
        )

        val resultado = chainService.investigar("Kotlin coroutines")

        // Verificas que el chain se comporta correctamente,
        // NO verificas la calidad de la respuesta del LLM
        verify(chatModel, times(2)).generate(any())  // se llamo 2 veces (queries + sintesis)
    }

    @Test
    fun `maneja error del LLM gracefully`() {
        whenever(chatModel.generate(any())).thenThrow(RuntimeException("API caida"))

        assertThrows<ServiceUnavailableException> {
            chainService.investigar("cualquier tema")
        }
    }
}
```

Testea:
- La logica de orquestacion (mocking el LLM)
- El manejo de errores
- La transformacion de datos
- Los prompts que se envian (no la respuesta)

### 6.3 Snapshot testing

Guarda respuestas "aprobadas" y compara futuras ejecuciones:

```kotlin
@Test
fun `respuesta de FAQ sobre devoluciones es consistente`() {
    val respuesta = faqService.responder("Cual es la politica de devolucion?")

    // Verifica que la respuesta contiene los elementos clave
    // No verifica el texto exacto
    assertThat(respuesta).contains("30 dias")
    assertThat(respuesta).contains("devolucion")
    assertThat(respuesta).doesNotContain("inventado")  // no alucina politicas

    // Opcionalmente, guarda como snapshot para revision humana
    snapshotStore.save("faq-devolucion", respuesta)
}
```

### 6.4 LLM-as-judge en tests

Usa un LLM para evaluar la calidad de las respuestas en tus tests:

```kotlin
@Test
fun `respuesta del agente es relevante y correcta`() {
    val respuesta = agenteVentas.atender("Quiero comprar un laptop para programar")

    // Un LLM evalua si la respuesta es buena
    val evaluacion = evaluador.evaluar(
        pregunta = "Quiero comprar un laptop para programar",
        esperada = "Debe recomendar laptops con buen procesador y RAM suficiente",
        actual = respuesta
    )

    assertThat(evaluacion.relevancia).isGreaterThanOrEqualTo(7)
    assertThat(evaluacion.precisionFactual).isGreaterThanOrEqualTo(7)
}
```

**Advertencia**: estos tests son mas lentos y costosos (llaman a un LLM real). Usarlos como tests de integracion, no como tests unitarios. No ejecutar en cada commit.

### 6.5 Tests de integracion con Testcontainers

Para testear RAG con una base de datos vectorial real:

```kotlin
@SpringBootTest
@Testcontainers
class RagIntegrationTest {

    companion object {
        @Container
        val pgvector = PostgreSQLContainer("pgvector/pgvector:pg16")
            .withDatabaseName("testdb")

        @Container
        val ollama = GenericContainer("ollama/ollama:latest")
            .withExposedPorts(11434)

        @DynamicPropertySource
        @JvmStatic
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url") { pgvector.jdbcUrl }
            registry.add("spring.ai.ollama.base-url") {
                "http://${ollama.host}:${ollama.getMappedPort(11434)}"
            }
        }
    }

    @Autowired
    lateinit var ragService: RagService

    @Test
    fun `RAG recupera documentos relevantes y genera respuesta coherente`() {
        // Preparar: insertar documentos conocidos
        ragService.indexar(listOf(
            Documento("La politica de devolucion es de 30 dias."),
            Documento("Los envios gratuitos aplican a pedidos mayores a 50 euros.")
        ))

        // Actuar
        val respuesta = ragService.consultar("Puedo devolver un producto?")

        // Verificar
        assertThat(respuesta).containsIgnoringCase("30 dias")
    }
}
```

---

## 7. Caching y Optimizacion

### 7.1 Semantic caching

El cache tradicional es exacto: misma query = mismo resultado. Pero los usuarios pueden preguntar lo mismo de formas diferentes:

- "Cual es tu politica de devolucion?"
- "Como puedo devolver un producto?"
- "Devolucion de articulos"

Todas deberian dar la misma respuesta cacheada. El **semantic caching** usa embeddings para detectar similitud:

```kotlin
@Service
class SemanticCache(
    private val embeddingModel: EmbeddingModel,
    private val vectorStore: VectorStore,
    private val cacheRepo: CacheEntryRepository
) {

    private val SIMILARITY_THRESHOLD = 0.92  // umbral de similitud

    fun getOrCompute(query: String, computeFn: () -> String): String {
        // Paso 1: generar embedding de la query
        val queryEmbedding = embeddingModel.embed(query).content()

        // Paso 2: buscar en cache por similitud semantica
        val similar = vectorStore.similaritySearch(
            SearchRequest.query(query)
                .withTopK(1)
                .withSimilarityThreshold(SIMILARITY_THRESHOLD)
        )

        if (similar.isNotEmpty()) {
            // Cache HIT semantico
            val cacheKey = similar.first().metadata["cacheKey"] as String
            val cached = cacheRepo.findByKey(cacheKey)
            if (cached != null && !cached.isExpired()) {
                return cached.response
            }
        }

        // Cache MISS: computar y guardar
        val response = computeFn()
        val cacheKey = UUID.randomUUID().toString()
        cacheRepo.save(CacheEntry(cacheKey, query, response, Instant.now()))
        vectorStore.add(listOf(
            Document(query, mapOf("cacheKey" to cacheKey))
        ))

        return response
    }
}
```

### 7.2 Response deduplication

Evita llamadas duplicadas al LLM cuando multiples usuarios hacen la misma pregunta simultaneamente:

```kotlin
@Service
class DeduplicatingAiService(private val chatModel: ChatLanguageModel) {

    // ConcurrentHashMap de queries en vuelo
    private val inFlight = ConcurrentHashMap<String, CompletableFuture<String>>()

    fun query(prompt: String): String {
        val key = prompt.hashCode().toString()

        // Si ya hay una peticion identica en vuelo, espera su resultado
        val future = inFlight.computeIfAbsent(key) {
            CompletableFuture.supplyAsync {
                try {
                    chatModel.generate(prompt).content().text()
                } finally {
                    inFlight.remove(key)  // limpiar al terminar
                }
            }
        }

        return future.get(30, TimeUnit.SECONDS)
    }
}
```

### 7.3 Estrategias de cache por tipo de consulta

| Tipo de consulta | Estrategia | TTL sugerido |
|------------------|------------|--------------|
| FAQ / informacion estatica | Cache agresivo, semantic cache | 24h - 7 dias |
| Resumenes de documentos | Cache por hash del documento | Indefinido (inmutable) |
| Conversacion libre | No cachear (contexto unico) | N/A |
| Clasificacion / extraccion | Cache exacto por input | 1h - 24h |
| Generacion creativa | No cachear (se espera variacion) | N/A |

---

## 8. Fine-tuning: cuando tiene sentido

### 8.1 Fine-tuning vs prompting vs RAG

| Tecnica | Que hace | Cuando usarla |
|---------|----------|---------------|
| **Prompting** | Instrucciones en el prompt | Siempre el primer intento |
| **Few-shot** | Ejemplos en el prompt | Cuando el modelo no entiende el formato |
| **RAG** | Inyectar conocimiento externo | Datos que cambian, informacion especifica |
| **Fine-tuning** | Reentrenar el modelo con tus datos | Patron de comportamiento muy especifico, formato consistente |

**Regla practica**: intenta prompting -> few-shot -> RAG antes de fine-tuning. El fine-tuning es caro, lento y dificil de mantener.

Fine-tuning tiene sentido cuando:
- Necesitas un estilo/tono muy especifico y consistente
- El volumen de llamadas justifica el costo de entrenamiento
- El comportamiento deseado es dificil de describir en un prompt
- Necesitas reducir latencia (modelo mas pequeno fine-tuneado vs modelo grande con prompt largo)

### 8.2 Preparacion de datos

El dataset de fine-tuning es una coleccion de pares (input, output) en formato JSONL:

```json
{"messages": [{"role": "system", "content": "Eres un asistente de soporte tecnico."}, {"role": "user", "content": "Mi laptop no enciende"}, {"role": "assistant", "content": "Entiendo tu frustracion. Vamos paso a paso: 1) Verifica que el cable de alimentacion esta bien conectado..."}]}
{"messages": [{"role": "system", "content": "Eres un asistente de soporte tecnico."}, {"role": "user", "content": "La pantalla esta en negro"}, {"role": "assistant", "content": "Revisemos juntos: 1) Intenta presionar las teclas Fn + F5 para alternar la pantalla..."}]}
```

Consideraciones criticas:
- **Cantidad**: minimo 50-100 ejemplos para fine-tuning basico, 500+ para resultados solidos
- **Calidad**: cada ejemplo debe ser perfecto. Basura entra, basura sale
- **Diversidad**: cubre los distintos escenarios que tu sistema encontrara
- **Consistencia**: mantener el mismo formato y tono en todos los ejemplos

### 8.3 El flujo completo

```
1. Recolectar datos     -> logs de conversaciones exitosas
2. Limpiar y anotar     -> humanos revisan y corrigen
3. Formatear            -> JSONL con el formato del proveedor
4. Entrenar             -> API del proveedor (OpenAI, Mistral, etc.)
5. Evaluar              -> comparar fine-tuned vs base en tu test set
6. Desplegar            -> apuntar tu aplicacion al modelo fine-tuned
7. Monitorear           -> el modelo puede degradarse con el tiempo
8. Re-entrenar          -> periodicamente con datos nuevos
```

---

## 9. Resumen

| Concepto | Que es | Como usarlo |
|----------|--------|-------------|
| **LangChain4j** | Framework de orquestacion de IA para JVM | AI Services, interfaces con `@UserMessage` |
| **Chains** | Secuencia de pasos (LLM + logica) | Funciones que encadenan resultados |
| **Branching** | Elegir camino segun clasificacion | `when` + agente clasificador |
| **Ejecucion paralela** | Pasos independientes en paralelo | `async/await` con coroutines |
| **Structured Output** | Respuestas del LLM como objetos tipados | Retorno tipado en AI Services |
| **Multi-agente** | Multiples LLMs especializados coordinados | Orquestador + agentes con tools |
| **LLM-as-judge** | LLM que evalua respuestas de otro LLM | Evaluador con criterios definidos |
| **Semantic cache** | Cache basado en similitud semantica | Embeddings + vector store |
| **Fine-tuning** | Reentrenar modelo con datos propios | Dataset JSONL + API del proveedor |
| **Snapshot testing** | Comparar respuestas contra versiones aprobadas | `assertThat(resp).contains(...)` |

---

## 10. Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | [LangChain4j basico](ejercicio-01-langchain4j/) | Crear un AI Service con LangChain4j, configurar memoria por usuario, definir tools para acceso a BD | AI Services, `@Tool`, `@MemoryId`, `ChatMemory` |
| 02 | [Chains y workflows](ejercicio-02-chains-workflows/) | Implementar un chain de investigacion con clasificacion, branching y ejecucion paralela | Chains secuenciales, branching por intent, `async/await` |
| 03 | [Structured outputs](ejercicio-03-structured-outputs/) | Extraer datos estructurados de texto libre, validar con schema enforcement y manejar fallbacks | `BeanOutputConverter`, retorno tipado, validacion |
| 04 | [Testing AI](ejercicio-04-testing-ai/) | Escribir tests para un sistema de IA: mock del LLM, snapshot testing y LLM-as-judge | `@MockBean`, semantic assertions, evaluacion automatizada |

Haz los ejercicios en orden. Cada uno construye sobre los conceptos del anterior.

---

## 11. Recursos

- [LangChain4j Documentation](https://docs.langchain4j.dev/)
- [LangChain4j Spring Boot Starter](https://docs.langchain4j.dev/tutorials/spring-boot-integration)
- [Spring AI Reference](https://docs.spring.io/spring-ai/reference/)
- [Spring AI Structured Output](https://docs.spring.io/spring-ai/reference/api/structured-output-converter.html)
- [OpenAI Fine-tuning Guide](https://platform.openai.com/docs/guides/fine-tuning)
- [RAGAS: Evaluation Framework for RAG](https://docs.ragas.io/)
- [DeepEval: LLM Evaluation Framework](https://docs.confident-ai.com/)
- [Testcontainers for Java](https://testcontainers.com/guides/getting-started-with-testcontainers-for-java/)

---

## Track Python

El track Python de este nivel cubre orquestacion avanzada usando el ecosistema Python:
LCEL (LangChain Expression Language) con pipes y Runnables, LangGraph para workflows
complejos con `StateGraph`, structured outputs con `with_structured_output` y Pydantic,
orquestacion multi-agente, LangSmith para tracing, evaluacion con LLM-as-judge,
testing con pytest y caching semantico.

Consulta el archivo [README-python.md](README-python.md) para la teoria completa con ejemplos de codigo.

### Ejercicios Python

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 05 | [LangChain Chains](ejercicio-05-python-langchain-chains/) | Construir chains con LCEL: chain RAG con `RunnablePassthrough`, router con `RunnableBranch` y analisis paralelo con `RunnableParallel` | LCEL, `RunnablePassthrough`, `RunnableLambda`, `RunnableParallel`, `RunnableBranch` |
| 06 | [LangGraph](ejercicio-06-python-langgraph/) | Implementar un workflow de soporte con `StateGraph`: clasificacion, routing condicional, resolucion automatica y escalacion | `StateGraph`, nodos, edges condicionales, `MemorySaver`, ciclos |
| 07 | [Structured Outputs](ejercicio-07-python-structured-outputs/) | Extraer datos estructurados con `with_structured_output`. Validar con Pydantic y manejar errores | `with_structured_output`, Pydantic models, `field_validator`, enums |
| 08 | [Evaluacion y Testing](ejercicio-08-python-eval-testing/) | Tests con pytest: mocks del LLM, snapshot testing y LLM-as-judge. Crear dataset de evaluacion y evaluar con metricas | `pytest`, `MagicMock`, snapshot testing, `@pytest.mark.slow`, LLM-as-judge |
