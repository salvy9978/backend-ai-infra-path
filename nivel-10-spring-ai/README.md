# Nivel 10 — Spring AI

> Spring AI es el framework oficial de Spring para integrar modelos de inteligencia artificial
> en aplicaciones Spring Boot. Proporciona abstracciones uniformes sobre distintos proveedores
> (OpenAI, Anthropic, Ollama, Google) para que cambiar de modelo sea tan simple como cambiar
> una propiedad en la configuracion. Ya conoces las APIs de LLMs (nivel 09). Ahora las usamos
> con la ergonomia y las convenciones del ecosistema Spring.

---

## Contenido

1. [Que es Spring AI](#1-que-es-spring-ai)
2. [Dependencias Maven](#2-dependencias-maven)
3. [ChatClient y ChatModel](#3-chatclient-y-chatmodel)
4. [Configuracion de proveedores](#4-configuracion-de-proveedores)
5. [Prompt Templates](#5-prompt-templates)
6. [Output Parsers](#6-output-parsers)
7. [Streaming de respuestas](#7-streaming-de-respuestas)
8. [Memoria de conversacion](#8-memoria-de-conversacion)
9. [Soporte multi-modelo](#9-soporte-multi-modelo)
10. [Manejo de errores y reintentos](#10-manejo-de-errores-y-reintentos)
11. [Resumen](#resumen)
12. [Ejercicios](#ejercicios)
13. [Recursos](#recursos)

---

## 1. Que es Spring AI

### 1.1 El problema que resuelve

Sin Spring AI, integrar un LLM en tu aplicacion Spring Boot requiere:

- Configurar WebClient o RestTemplate manualmente
- Construir los JSON de request/response a mano
- Parsear las respuestas y mapearlas a objetos Kotlin
- Implementar reintentos, timeouts y manejo de errores
- Cambiar de proveedor implica reescribir todo el codigo

Spring AI abstrae todo esto con interfaces comunes:

```
Tu codigo Kotlin
      |
      v
  Spring AI (ChatClient, ChatModel, VectorStore)
      |
      +---> OpenAI
      +---> Anthropic
      +---> Ollama (modelos locales)
      +---> Google Vertex AI
      +---> Azure OpenAI
      +---> Mistral AI
```

### 1.2 Arquitectura de Spring AI

```
spring-ai
├── spring-ai-core              # Interfaces comunes (ChatModel, EmbeddingModel, VectorStore)
├── spring-ai-openai            # Implementacion para OpenAI
├── spring-ai-anthropic         # Implementacion para Anthropic
├── spring-ai-ollama            # Implementacion para Ollama
├── spring-ai-vertex-ai-gemini  # Implementacion para Google Gemini
├── spring-ai-pgvector          # VectorStore con PostgreSQL pgvector
└── spring-ai-retry             # Reintentos y circuit breaker
```

La clave es que tu codigo de negocio depende de las interfaces de `spring-ai-core`,
nunca de un proveedor especifico. Puedes cambiar de OpenAI a Anthropic cambiando
una dependencia Maven y unas propiedades.

---

## 2. Dependencias Maven

### 2.1 BOM (Bill of Materials)

Spring AI usa un BOM para gestionar versiones de forma coherente:

```xml
<!-- pom.xml -->
<properties>
    <spring-ai.version>1.0.0</spring-ai.version>
</properties>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.ai</groupId>
            <artifactId>spring-ai-bom</artifactId>
            <version>${spring-ai.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### 2.2 Starters por proveedor

Anade solo el starter del proveedor que vas a usar:

```xml
<!-- Para OpenAI -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
</dependency>

<!-- Para Anthropic (Claude) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-anthropic-spring-boot-starter</artifactId>
</dependency>

<!-- Para Ollama (modelos locales) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-ollama-spring-boot-starter</artifactId>
</dependency>

<!-- Para Google Vertex AI (Gemini) -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-vertex-ai-gemini-spring-boot-starter</artifactId>
</dependency>
```

### 2.3 Repositorio de Spring Milestones

Spring AI puede requerir el repositorio de milestones si usas una version pre-release:

```xml
<repositories>
    <repository>
        <id>spring-milestones</id>
        <name>Spring Milestones</name>
        <url>https://repo.spring.io/milestone</url>
        <snapshots>
            <enabled>false</enabled>
        </snapshots>
    </repository>
</repositories>
```

---

## 3. ChatClient y ChatModel

### 3.1 ChatModel: la interfaz base

`ChatModel` es la interfaz central de Spring AI. Representa cualquier modelo de chat
y tiene un solo metodo principal:

```kotlin
// La interfaz (simplificada)
interface ChatModel {
    fun call(prompt: Prompt): ChatResponse
    fun stream(prompt: Prompt): Flux<ChatResponse>
}
```

Cuando anade el starter de OpenAI, Spring Boot autoconfigura un bean `OpenAiChatModel`
que implementa `ChatModel`. No necesitas crearlo manualmente.

### 3.2 ChatClient: la API fluida

`ChatClient` es una API de alto nivel (builder pattern) construida sobre `ChatModel`.
Es la forma recomendada de interactuar con los modelos:

```kotlin
@RestController
@RequestMapping("/api/chat")
class ChatController(
    private val chatClient: ChatClient
) {

    // La forma mas simple: una pregunta, una respuesta
    @GetMapping("/simple")
    fun chatSimple(@RequestParam pregunta: String): String {
        return chatClient.prompt()
            .user(pregunta)                    // mensaje del usuario
            .call()                            // llamada sincrona
            .content()                         // extrae el texto de la respuesta
            ?: "Sin respuesta"
    }

    // Con system prompt
    @GetMapping("/con-sistema")
    fun chatConSistema(@RequestParam pregunta: String): String {
        return chatClient.prompt()
            .system("Eres un experto en Spring Boot. Responde en espanol, de forma concisa.")
            .user(pregunta)
            .call()
            .content() ?: "Sin respuesta"
    }
}
```

### 3.3 Configurar el bean ChatClient

```kotlin
@Configuration
class ChatClientConfig {

    @Bean
    fun chatClient(chatModel: ChatModel): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                Eres un asistente tecnico especializado en Spring Boot y Kotlin.
                Respondes en espanol. Incluyes ejemplos de codigo cuando es relevante.
            """.trimIndent())
            .build()
    }
}
```

### 3.4 ChatResponse detallada

```kotlin
@GetMapping("/detallada")
fun chatDetallada(@RequestParam pregunta: String): Map<String, Any> {
    val response = chatClient.prompt()
        .user(pregunta)
        .call()
        .chatResponse()                          // respuesta completa, no solo el texto

    val generation = response.result             // primera generacion
    val metadata = response.metadata             // metadatos de la llamada

    return mapOf(
        "respuesta" to (generation.output.content ?: ""),
        "modelo" to (metadata.model ?: "desconocido"),
        "tokensPrompt" to (metadata.usage?.promptTokens ?: 0),
        "tokensRespuesta" to (metadata.usage?.generationTokens ?: 0),
        "tokensTotal" to (metadata.usage?.totalTokens ?: 0),
        "finishReason" to (generation.metadata?.finishReason ?: "desconocido")
    )
}
```

---

## 4. Configuracion de proveedores

### 4.1 OpenAI

```yaml
# application.yml
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}          # variable de entorno
      chat:
        options:
          model: gpt-4o-mini              # modelo por defecto
          temperature: 0.7                # creatividad
          max-tokens: 1000                # limite de respuesta
          top-p: 1.0                      # nucleus sampling
      embedding:
        options:
          model: text-embedding-3-small   # modelo de embeddings
```

### 4.2 Anthropic (Claude)

```yaml
spring:
  ai:
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514
          temperature: 0.7
          max-tokens: 1024
```

### 4.3 Ollama (modelos locales)

```yaml
spring:
  ai:
    ollama:
      base-url: http://localhost:11434    # URL de Ollama local
      chat:
        options:
          model: llama3.1:8b              # modelo descargado localmente
          temperature: 0.7
      embedding:
        options:
          model: nomic-embed-text         # modelo de embeddings local
```

### 4.4 Sobreescribir opciones por llamada

Las opciones de `application.yml` son los defaults, pero puedes sobreescribirlas
en cada llamada:

```kotlin
import org.springframework.ai.openai.OpenAiChatOptions

@GetMapping("/personalizado")
fun chatPersonalizado(@RequestParam pregunta: String): String {
    return chatClient.prompt()
        .user(pregunta)
        .options(OpenAiChatOptions.builder()
            .model("gpt-4o")              // usar un modelo diferente para esta llamada
            .temperature(0.0)              // determinista
            .maxTokens(200)                // respuesta corta
            .build()
        )
        .call()
        .content() ?: "Sin respuesta"
}
```

---

## 5. Prompt Templates

### 5.1 El problema de concatenar strings

Construir prompts concatenando strings es fragil y dificil de mantener:

```kotlin
// Malo: concatenacion manual
val prompt = "Traduce el siguiente texto de $idiomaOrigen a $idiomaDestino: $texto"
```

### 5.2 PromptTemplate de Spring AI

`PromptTemplate` permite definir plantillas con variables:

```kotlin
import org.springframework.ai.chat.prompt.PromptTemplate

@Service
class TraduccionService(
    private val chatClient: ChatClient
) {

    // Template reutilizable con variables
    private val templateTraduccion = PromptTemplate("""
        Traduce el siguiente texto de {idiomaOrigen} a {idiomaDestino}.
        Manten el formato original. No anadeas explicaciones.
        
        Texto a traducir:
        {texto}
    """.trimIndent())

    fun traducir(texto: String, de: String, a: String): String {
        val prompt = templateTraduccion.create(mapOf(
            "idiomaOrigen" to de,
            "idiomaDestino" to a,
            "texto" to texto
        ))

        return chatClient.prompt(prompt)
            .call()
            .content() ?: "Error en la traduccion"
    }
}
```

### 5.3 Templates desde archivos

Para prompts largos, es mejor mantenerlos en archivos externos:

```
src/main/resources/prompts/
├── clasificador-tickets.st       # template para clasificar tickets
├── resumen-documento.st          # template para resumir documentos
└── extraccion-datos.st           # template para extraer datos
```

```
# clasificador-tickets.st
Eres un sistema de clasificacion de tickets de soporte.

Categorias disponibles: {categorias}

Clasifica el siguiente ticket y devuelve un JSON con formato:
{{"categoria": "...", "prioridad": "alta|media|baja", "resumen": "..."}}

Ticket:
{ticket}
```

```kotlin
import org.springframework.ai.chat.prompt.PromptTemplate
import org.springframework.beans.factory.annotation.Value
import org.springframework.core.io.Resource

@Service
class ClasificadorTicketsService(
    private val chatClient: ChatClient
) {
    // Cargar el template desde el archivo de recursos
    @Value("classpath:prompts/clasificador-tickets.st")
    private lateinit var templateResource: Resource

    fun clasificar(ticket: String): String {
        val template = PromptTemplate(templateResource)
        val prompt = template.create(mapOf(
            "categorias" to "autenticacion, funcionalidad, rendimiento, datos, feature_request",
            "ticket" to ticket
        ))

        return chatClient.prompt(prompt)
            .options(OpenAiChatOptions.builder()
                .temperature(0.0)          // determinista para clasificacion
                .build()
            )
            .call()
            .content() ?: """{"error": "Sin respuesta"}"""
    }
}
```

### 5.4 Multiples mensajes en un template

```kotlin
@Service
class ChatConHistorialService(
    private val chatClient: ChatClient
) {

    fun chatConContexto(
        historial: List<Pair<String, String>>,  // pares (user, assistant)
        nuevaPregunta: String
    ): String {
        // Construir la lista de mensajes con el historial
        val builder = chatClient.prompt()
            .system("Eres un asistente tecnico. Responde en espanol.")

        // El ChatClient de Spring AI maneja multiples mensajes
        // Para conversaciones multi-turno, usa memoria (seccion 8)
        return builder
            .user(nuevaPregunta)
            .call()
            .content() ?: "Sin respuesta"
    }
}
```

---

## 6. Output Parsers

### 6.1 El problema: parsear texto libre

Sin output parsers, tienes que parsear manualmente la respuesta del modelo:

```kotlin
// Malo: parsear JSON manualmente y esperar que el modelo lo devuelva bien
val respuesta = chatClient.prompt().user("Dame info sobre Kotlin en JSON").call().content()
val json = ObjectMapper().readTree(respuesta)  // puede fallar si el modelo no devuelve JSON valido
```

### 6.2 BeanOutputConverter

Spring AI incluye `BeanOutputConverter` que convierte la respuesta del modelo directamente
a un objeto Kotlin/Java:

```kotlin
// Data class que define la estructura esperada
data class InfoLenguaje(
    val nombre: String,
    val anioCreacion: Int,
    val creador: String,
    val paradigmas: List<String>,
    val tipado: String,
    val usoPrincipal: String
)

@Service
class LenguajeService(
    private val chatClient: ChatClient
) {

    fun obtenerInfo(lenguaje: String): InfoLenguaje {
        return chatClient.prompt()
            .user("Dame informacion sobre el lenguaje de programacion $lenguaje")
            .call()
            .entity(InfoLenguaje::class.java)    // conversion automatica a la data class
    }
}
```

`entity()` anade automaticamente instrucciones al prompt indicando al modelo que
devuelva JSON con el schema de la data class, y parsea la respuesta.

### 6.3 Listas y tipos complejos

```kotlin
data class Recomendacion(
    val titulo: String,
    val descripcion: String,
    val prioridad: String,   // "alta", "media", "baja"
    val complejidad: String  // "simple", "moderada", "compleja"
)

@Service
class RecomendacionService(
    private val chatClient: ChatClient
) {

    // Devuelve una lista de objetos
    fun recomendar(contexto: String): List<Recomendacion> {
        return chatClient.prompt()
            .system("Eres un arquitecto de software. Genera recomendaciones tecnicas.")
            .user("Analiza y recomienda mejoras para: $contexto")
            .call()
            .entity(object : ParameterizedTypeReference<List<Recomendacion>>() {})
    }
}
```

### 6.4 Converter personalizado

Para formatos que no son JSON directo, puedes crear tu propio converter:

```kotlin
import org.springframework.ai.converter.StructuredOutputConverter

class CsvOutputConverter(
    private val headers: List<String>
) : StructuredOutputConverter<List<Map<String, String>>> {

    override fun getFormat(): String {
        return """
            Responde en formato CSV con las siguientes columnas: ${headers.joinToString(", ")}
            La primera linea son los headers. Usa coma como separador.
            No incluyas nada mas que el CSV.
        """.trimIndent()
    }

    override fun convert(text: String): List<Map<String, String>> {
        val lineas = text.trim().lines()
        if (lineas.size < 2) return emptyList()

        val cabeceras = lineas[0].split(",").map { it.trim() }
        return lineas.drop(1).map { linea ->
            val valores = linea.split(",").map { it.trim() }
            cabeceras.zip(valores).toMap()
        }
    }
}
```

### 6.5 Enum output

```kotlin
enum class Sentimiento {
    POSITIVO, NEGATIVO, NEUTRAL
}

@Service
class AnalisisSentimientoService(
    private val chatClient: ChatClient
) {

    fun analizar(texto: String): Sentimiento {
        val respuesta = chatClient.prompt()
            .system("""
                Clasificas el sentimiento de textos.
                Responde UNICAMENTE con una de estas palabras: POSITIVO, NEGATIVO, NEUTRAL
            """.trimIndent())
            .user(texto)
            .call()
            .content()?.trim()?.uppercase() ?: "NEUTRAL"

        return try {
            Sentimiento.valueOf(respuesta)
        } catch (e: IllegalArgumentException) {
            Sentimiento.NEUTRAL  // fallback seguro
        }
    }
}
```

---

## 7. Streaming de respuestas

### 7.1 Por que streaming

En una llamada normal (`call()`), el servidor espera a que el modelo genere la respuesta
completa antes de enviarla al cliente. Esto puede tardar varios segundos.

Con streaming, el modelo envia tokens a medida que los genera, dando una experiencia
mucho mas fluida al usuario (como ChatGPT).

### 7.2 Streaming basico con Flux

```kotlin
@RestController
@RequestMapping("/api/chat")
class StreamingController(
    private val chatClient: ChatClient
) {

    // Endpoint que devuelve un stream de texto
    @GetMapping("/stream", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
    fun chatStream(@RequestParam pregunta: String): Flux<String> {
        return chatClient.prompt()
            .user(pregunta)
            .stream()                              // devuelve Flux en lugar de bloquear
            .content()                             // extrae solo el texto de cada chunk
            .filter { it != null }                 // filtrar nulls
            .map { it!! }                          // safe cast
    }
}
```

### 7.3 Server-Sent Events (SSE)

Para clientes web, SSE es el formato estandar para streaming:

```kotlin
@GetMapping("/stream/sse", produces = [MediaType.TEXT_EVENT_STREAM_VALUE])
fun chatStreamSSE(@RequestParam pregunta: String): Flux<ServerSentEvent<String>> {
    return chatClient.prompt()
        .system("Responde en espanol de forma detallada.")
        .user(pregunta)
        .stream()
        .content()
        .filter { it != null }
        .map { chunk ->
            ServerSentEvent.builder(chunk!!)
                .event("message")
                .build()
        }
        .concatWith(Flux.just(
            ServerSentEvent.builder("[DONE]")
                .event("done")
                .build()
        ))
}
```

### 7.4 Consumir streaming desde JavaScript

```javascript
// Frontend: consumir el stream
const eventSource = new EventSource('/api/chat/stream/sse?pregunta=Hola');

eventSource.onmessage = (event) => {
    if (event.data === '[DONE]') {
        eventSource.close();
        return;
    }
    document.getElementById('respuesta').textContent += event.data;
};

eventSource.onerror = () => {
    eventSource.close();
};
```

### 7.5 Streaming con acumulador

A veces necesitas procesar la respuesta completa despues del streaming:

```kotlin
@GetMapping("/stream/completo")
fun chatStreamCompleto(@RequestParam pregunta: String): Flux<String> {
    val acumulador = StringBuilder()

    return chatClient.prompt()
        .user(pregunta)
        .stream()
        .content()
        .filter { it != null }
        .map { chunk ->
            acumulador.append(chunk!!)
            chunk
        }
        .doOnComplete {
            // Cuando termina el stream, tienes la respuesta completa
            val respuestaCompleta = acumulador.toString()
            logger.info("Respuesta completa: ${respuestaCompleta.length} caracteres")
            // Guardar en BD, registrar metricas, etc.
        }
}
```

---

## 8. Memoria de conversacion

### 8.1 El problema: los LLMs no tienen memoria

Cada llamada a la API es independiente. El modelo no recuerda conversaciones previas.
Para mantener una conversacion coherente, debes enviar el historial completo en cada
llamada.

### 8.2 ChatMemory de Spring AI

Spring AI proporciona abstracciones de memoria que gestionan el historial automaticamente:

```kotlin
import org.springframework.ai.chat.memory.InMemoryChatMemory
import org.springframework.ai.chat.memory.ChatMemory

@Configuration
class MemoryConfig {

    @Bean
    fun chatMemory(): ChatMemory {
        // Memoria en RAM: se pierde al reiniciar la aplicacion
        return InMemoryChatMemory()
    }
}
```

### 8.3 MessageWindowChatMemory

`MessageWindowChatMemory` limita el numero de mensajes en el historial para no exceder
la ventana de contexto del modelo:

```kotlin
import org.springframework.ai.chat.memory.MessageWindowChatMemory

@Configuration
class MemoryConfig {

    @Bean
    fun chatMemory(): ChatMemory {
        return MessageWindowChatMemory.builder()
            .maxMessages(20)             // mantener ultimos 20 mensajes
            .build()
    }
}
```

### 8.4 ChatClient con memoria

```kotlin
@Configuration
class ChatClientConfig {

    @Bean
    fun chatClient(
        chatModel: ChatModel,
        chatMemory: ChatMemory
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("Eres un asistente tecnico. Responde en espanol.")
            .defaultAdvisors(
                MessageChatMemoryAdvisor(chatMemory)  // inyecta memoria automaticamente
            )
            .build()
    }
}
```

### 8.5 Conversaciones por usuario

Cada usuario debe tener su propia memoria de conversacion:

```kotlin
@RestController
@RequestMapping("/api/chat")
class ChatConMemoriaController(
    private val chatClient: ChatClient
) {

    @PostMapping("/conversacion")
    fun chat(
        @RequestParam conversacionId: String,    // ID unico por conversacion
        @RequestBody mensaje: String
    ): String {
        return chatClient.prompt()
            .user(mensaje)
            .advisors { advisor ->
                advisor.param(
                    ChatMemory.CONVERSATION_ID,   // clave para identificar la conversacion
                    conversacionId
                )
            }
            .call()
            .content() ?: "Sin respuesta"
    }
}
```

```kotlin
// Uso desde el cliente:
// POST /api/chat/conversacion?conversacionId=user-123-session-1
// Body: "Hola, que es Spring Boot?"

// POST /api/chat/conversacion?conversacionId=user-123-session-1
// Body: "Y como se configura?"
// -> El modelo "recuerda" que estabais hablando de Spring Boot
```

### 8.6 Persistir la memoria

Para produccion, necesitas persistir la memoria mas alla de la RAM:

```kotlin
// Implementacion basica con JDBC
@Service
class JdbcChatMemory(
    private val jdbcTemplate: JdbcTemplate
) : ChatMemory {

    override fun add(conversationId: String, messages: List<Message>) {
        for (message in messages) {
            jdbcTemplate.update(
                "INSERT INTO chat_memory (conversation_id, role, content, created_at) VALUES (?, ?, ?, ?)",
                conversationId,
                message.messageType.name,
                message.content,
                LocalDateTime.now()
            )
        }
    }

    override fun get(conversationId: String, lastN: Int): List<Message> {
        return jdbcTemplate.query(
            """SELECT role, content FROM chat_memory 
               WHERE conversation_id = ? 
               ORDER BY created_at DESC LIMIT ?""",
            arrayOf(conversationId, lastN)
        ) { rs, _ ->
            when (rs.getString("role")) {
                "USER" -> UserMessage(rs.getString("content"))
                "ASSISTANT" -> AssistantMessage(rs.getString("content"))
                "SYSTEM" -> SystemMessage(rs.getString("content"))
                else -> UserMessage(rs.getString("content"))
            }
        }.reversed()  // invertir para orden cronologico
    }

    override fun clear(conversationId: String) {
        jdbcTemplate.update(
            "DELETE FROM chat_memory WHERE conversation_id = ?",
            conversationId
        )
    }
}
```

---

## 9. Soporte multi-modelo

### 9.1 Varios proveedores en la misma aplicacion

Puedes usar distintos modelos para distintas tareas. Un modelo barato para
clasificacion y uno potente para generacion compleja:

```yaml
# application.yml con multiples proveedores
spring:
  ai:
    openai:
      api-key: ${OPENAI_API_KEY}
      chat:
        options:
          model: gpt-4o-mini          # modelo por defecto (barato)
    anthropic:
      api-key: ${ANTHROPIC_API_KEY}
      chat:
        options:
          model: claude-sonnet-4-20250514  # modelo potente
    ollama:
      base-url: http://localhost:11434
      chat:
        options:
          model: llama3.1:8b          # modelo local
```

### 9.2 Multiples beans ChatModel

```kotlin
@Configuration
class MultiModelConfig {

    @Bean("openAiChatClient")
    fun openAiChatClient(
        @Qualifier("openAiChatModel") chatModel: ChatModel
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("Responde en espanol.")
            .build()
    }

    @Bean("anthropicChatClient")
    fun anthropicChatClient(
        @Qualifier("anthropicChatModel") chatModel: ChatModel
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("Responde en espanol con detalle.")
            .build()
    }
}

@Service
class SmartRouterService(
    @Qualifier("openAiChatClient") private val clienteRapido: ChatClient,
    @Qualifier("anthropicChatClient") private val clientePotente: ChatClient
) {

    fun responder(pregunta: String, complejidad: String): String {
        val cliente = when (complejidad) {
            "simple" -> clienteRapido        // GPT-4o-mini: rapido y barato
            "compleja" -> clientePotente     // Claude Sonnet: mas capaz
            else -> clienteRapido
        }
        return cliente.prompt()
            .user(pregunta)
            .call()
            .content() ?: "Sin respuesta"
    }
}
```

### 9.3 Fallback entre proveedores

```kotlin
@Service
class FallbackLlmService(
    @Qualifier("openAiChatClient") private val primario: ChatClient,
    @Qualifier("ollamaChatClient") private val fallback: ChatClient
) {

    fun chat(pregunta: String): String {
        return try {
            primario.prompt().user(pregunta).call().content()
                ?: throw RuntimeException("Respuesta vacia")
        } catch (e: Exception) {
            logger.warn("OpenAI fallo, usando Ollama como fallback: ${e.message}")
            fallback.prompt().user(pregunta).call().content()
                ?: "Error: ningún proveedor disponible"
        }
    }
}
```

---

## 10. Manejo de errores y reintentos

### 10.1 Excepciones comunes

```kotlin
// Spring AI lanza estas excepciones comunes:
// - NonTransientAiException: error permanente (ej. API key invalida, prompt demasiado largo)
// - TransientAiException: error temporal (ej. rate limit, timeout, server error)
```

### 10.2 Configuracion de reintentos

Spring AI incluye soporte de reintentos con Spring Retry:

```yaml
spring:
  ai:
    retry:
      max-attempts: 3                    # maximo 3 intentos
      backoff:
        initial-interval: 2000           # 2 segundos de espera inicial
        multiplier: 2                    # duplicar espera en cada reintento
        max-interval: 30000              # maximo 30 segundos de espera
      on-client-errors: false            # no reintentar en errores 4xx
      on-http-codes: 429, 500, 502, 503  # reintentar en estos codigos HTTP
```

### 10.3 Manejo de errores en controladores

```kotlin
@RestController
@RequestMapping("/api/chat")
class ChatController(
    private val chatClient: ChatClient
) {

    @PostMapping
    fun chat(@RequestBody request: ChatRequest): ResponseEntity<ChatResponseDto> {
        return try {
            val respuesta = chatClient.prompt()
                .user(request.mensaje)
                .call()
                .content()

            ResponseEntity.ok(ChatResponseDto(
                mensaje = respuesta ?: "Sin respuesta",
                error = false
            ))
        } catch (e: NonTransientAiException) {
            // Error permanente: no reintentar
            logger.error("Error permanente de AI: ${e.message}")
            ResponseEntity.badRequest().body(ChatResponseDto(
                mensaje = "Error en la solicitud. Verifica tu mensaje.",
                error = true
            ))
        } catch (e: TransientAiException) {
            // Error temporal: el retry ya se agoto
            logger.error("Error temporal de AI tras reintentos: ${e.message}")
            ResponseEntity.status(503).body(ChatResponseDto(
                mensaje = "Servicio temporalmente no disponible. Intenta en unos minutos.",
                error = true
            ))
        }
    }
}

data class ChatRequest(val mensaje: String)
data class ChatResponseDto(val mensaje: String, val error: Boolean)
```

### 10.4 Circuit breaker con Resilience4j

Para mayor resiliencia en produccion:

```xml
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
```

```kotlin
@Service
class ResilientChatService(
    private val chatClient: ChatClient
) {

    @CircuitBreaker(name = "llm-service", fallbackMethod = "fallbackChat")
    @RateLimiter(name = "llm-rate-limiter")
    fun chat(pregunta: String): String {
        return chatClient.prompt()
            .user(pregunta)
            .call()
            .content() ?: "Sin respuesta"
    }

    // Fallback cuando el circuit breaker esta abierto
    fun fallbackChat(pregunta: String, throwable: Throwable): String {
        logger.warn("Circuit breaker abierto para LLM: ${throwable.message}")
        return "El servicio de AI no esta disponible en este momento. " +
               "Por favor, intenta de nuevo mas tarde."
    }
}
```

```yaml
# application.yml - configuracion de Resilience4j
resilience4j:
  circuitbreaker:
    instances:
      llm-service:
        sliding-window-size: 10          # ultimas 10 llamadas
        failure-rate-threshold: 50       # 50% de fallos abre el circuit breaker
        wait-duration-in-open-state: 30s # esperar 30s antes de probar de nuevo
        permitted-number-of-calls-in-half-open-state: 3
  ratelimiter:
    instances:
      llm-rate-limiter:
        limit-for-period: 50             # 50 llamadas por periodo
        limit-refresh-period: 1m         # periodo de 1 minuto
        timeout-duration: 5s             # esperar max 5s por un slot
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|---------|--------|-------------|
| Spring AI | Framework de Spring para integracion con LLMs | Starter + `application.yml` con API key |
| ChatModel | Interfaz base para modelos de chat | Se autoconfigura con el starter del proveedor |
| ChatClient | API fluida de alto nivel sobre ChatModel | `chatClient.prompt().user(...).call().content()` |
| PromptTemplate | Plantillas con variables para prompts | `PromptTemplate("Hola {nombre}")` con `.create(map)` |
| BeanOutputConverter | Convierte respuestas a objetos Kotlin | `.call().entity(MiDataClass::class.java)` |
| Streaming | Respuestas token a token con Flux | `.stream().content()` con SSE en el controlador |
| ChatMemory | Almacen de historial de conversacion | `MessageWindowChatMemory` + `MessageChatMemoryAdvisor` |
| InMemoryChatMemory | Memoria en RAM (desarrollo) | Bean `ChatMemory` con `InMemoryChatMemory()` |
| Multi-modelo | Varios proveedores en la misma app | `@Qualifier` para inyectar distintos ChatClient |
| Reintentos | Reintentar llamadas fallidas automaticamente | `spring.ai.retry.*` en application.yml |
| Circuit Breaker | Cortar llamadas cuando el servicio falla | Resilience4j con `@CircuitBreaker` |
| Fallback | Respuesta alternativa cuando falla el LLM | Metodo fallback o proveedor alternativo |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **ChatClient basico** | Crear una aplicacion Spring Boot con Spring AI que se conecte a OpenAI (o Ollama local), exponga un endpoint REST de chat y devuelva la respuesta con metadatos (tokens, modelo) | ChatClient, ChatModel, `application.yml`, OpenAI/Ollama starter |
| 02 | **Prompt Templates** | Construir un servicio de traduccion y un clasificador de tickets usando PromptTemplate con archivos `.st` externos. Comparar resultados entre modelos | PromptTemplate, archivos de resources, `@Value`, configuracion por llamada |
| 03 | **Output Parsers** | Crear endpoints que devuelvan objetos Kotlin tipados: info de lenguajes (entity), lista de recomendaciones (ParameterizedTypeReference), y sentimiento (enum) | BeanOutputConverter, entity(), data classes, tipado seguro |
| 04 | **Streaming y memoria** | Implementar un chatbot con streaming SSE y memoria de conversacion por usuario. Probar que el modelo recuerda mensajes previos de la misma conversacion | Flux, SSE, ChatMemory, MessageWindowChatMemory, conversationId |

---

## Recursos

- [Spring AI Reference Documentation](https://docs.spring.io/spring-ai/reference/)
- [Spring AI GitHub Repository](https://github.com/spring-projects/spring-ai)
- [Spring AI Samples](https://github.com/spring-projects/spring-ai/tree/main/spring-ai-samples)
- [Spring AI con Kotlin - Blog de Spring](https://spring.io/blog)
- [Ollama - Modelos locales](https://ollama.ai)

---

> **Consejo practico**: durante el desarrollo, usa Ollama con un modelo local (llama3.1:8b).
> Es gratis, no requiere API key y la API es compatible con el starter de Spring AI para Ollama.
> Cambia a OpenAI o Anthropic solo cuando necesites la calidad de un modelo mas grande.
