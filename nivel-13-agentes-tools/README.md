# Nivel 13 — Agentes y Function Calling

> Un chatbot responde preguntas. Un agente toma acciones. Los agentes combinan la capacidad
> de razonamiento de los LLMs con herramientas externas: consultar bases de datos, llamar
> APIs, hacer calculos, enviar emails. Este nivel cubre desde function calling basico hasta
> arquitecturas de agentes multi-herramienta con Spring AI. Ya dominas RAG (nivel 12).
> Ahora das al LLM la capacidad de actuar, no solo de hablar.

---

## Contenido

1. [Que son los agentes de IA](#1-que-son-los-agentes-de-ia)
2. [Function calling: como los LLMs usan herramientas](#2-function-calling-como-los-llms-usan-herramientas)
3. [Function calling en Spring AI](#3-function-calling-en-spring-ai)
4. [Construir herramientas personalizadas](#4-construir-herramientas-personalizadas)
5. [Arquitectura ReAct: razonamiento y accion](#5-arquitectura-react-razonamiento-y-accion)
6. [Agentes multi-herramienta](#6-agentes-multi-herramienta)
7. [Memoria y estado de los agentes](#7-memoria-y-estado-de-los-agentes)
8. [Manejo de errores y guardrails](#8-manejo-de-errores-y-guardrails)
9. [Cuando usar agentes vs RAG simple](#9-cuando-usar-agentes-vs-rag-simple)
10. [Resumen](#resumen)
11. [Ejercicios](#ejercicios)
12. [Recursos](#recursos)

---

## 1. Que son los agentes de IA

### 1.1 Del chat a la accion

Un chatbot clasico solo genera texto. Un agente puede:

```
Chatbot:
  Usuario: "Cual es el estado de mi pedido 12345?"
  Bot: "No tengo acceso a esa informacion." (o inventa algo)

Agente:
  Usuario: "Cual es el estado de mi pedido 12345?"
  Agente piensa: "Necesito consultar la base de datos de pedidos"
  Agente llama: consultarPedido(id=12345)
  Base de datos responde: {estado: "enviado", tracking: "ES123456789"}
  Agente responde: "Tu pedido 12345 esta enviado. El numero de tracking es ES123456789."
```

### 1.2 Componentes de un agente

```
Un agente tiene tres componentes:

1. LLM (el cerebro)
   - Razona sobre la tarea
   - Decide que herramienta usar y con que parametros
   - Interpreta los resultados

2. Herramientas (las manos)
   - Funciones que el agente puede invocar
   - Consultas a BD, llamadas a APIs, calculos, etc.
   - Cada herramienta tiene nombre, descripcion y parametros tipados

3. Orquestacion (el bucle)
   - El ciclo pensar -> actuar -> observar -> repetir
   - Decide cuando parar y dar la respuesta final
```

### 1.3 El flujo de un agente

```
                    +--------+
                    | Usuario|
                    +---+----+
                        |
                        v
                  +-----------+
            +---->|   LLM     |<---+
            |     | (razonar) |    |
            |     +-----+-----+    |
            |           |          |
            |    Decide usar       |
            |    herramienta       |
            |           |          |
            |           v          |
            |     +-----------+    |
            |     |Herramienta|    |
            |     |(ejecutar) |    |
            |     +-----+-----+    |
            |           |          |
            |     Resultado        |
            |           |          |
            +-----------+          |
                                   |
              Cuando tiene         |
              respuesta final------+
                        |
                        v
                  +-----------+
                  | Respuesta |
                  +-----------+
```

---

## 2. Function calling: como los LLMs usan herramientas

### 2.1 El mecanismo

Function calling no es magia. Funciona asi:

```
1. Tu defines una lista de funciones disponibles (nombre, descripcion, parametros)
2. Envias esta lista al LLM junto con el mensaje del usuario
3. El LLM decide si necesita llamar a alguna funcion
4. Si decide llamarla, devuelve el nombre de la funcion y los argumentos (en JSON)
5. TU CODIGO ejecuta la funcion con esos argumentos
6. Devuelves el resultado al LLM
7. El LLM genera la respuesta final incorporando el resultado
```

Importante: **el LLM NO ejecuta la funcion**. Solo decide cual llamar y con que
parametros. Tu backend es quien la ejecuta.

### 2.2 Ejemplo con la API de OpenAI (a bajo nivel)

```kotlin
// Asi se ve function calling a bajo nivel (sin Spring AI)
val request = mapOf(
    "model" to "gpt-4o",
    "messages" to listOf(
        mapOf("role" to "user", "content" to "Que tiempo hace en Madrid?")
    ),
    "tools" to listOf(
        mapOf(
            "type" to "function",
            "function" to mapOf(
                "name" to "obtenerClima",
                "description" to "Obtiene el clima actual de una ciudad",
                "parameters" to mapOf(
                    "type" to "object",
                    "properties" to mapOf(
                        "ciudad" to mapOf(
                            "type" to "string",
                            "description" to "Nombre de la ciudad"
                        )
                    ),
                    "required" to listOf("ciudad")
                )
            )
        )
    )
)

// El LLM responde con:
// {
//   "tool_calls": [{
//     "function": {
//       "name": "obtenerClima",
//       "arguments": "{\"ciudad\": \"Madrid\"}"
//     }
//   }]
// }

// Tu codigo ejecuta: obtenerClima("Madrid") -> "22C, soleado"
// Envias el resultado de vuelta al LLM
// El LLM responde: "En Madrid hace 22 grados y esta soleado."
```

Spring AI abstrae todo esto. Veamos como.

---

## 3. Function calling en Spring AI

### 3.1 Definir una funcion con @Description

La forma mas simple de crear una herramienta en Spring AI:

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Description

@Configuration
class FunctionConfig {

    @Bean
    @Description("Obtiene el clima actual de una ciudad dada")
    fun obtenerClima(): java.util.function.Function<ClimaRequest, ClimaResponse> {
        return java.util.function.Function { request ->
            // Aqui llamarias a una API real de clima
            // Para el ejemplo, datos hardcodeados
            when (request.ciudad.lowercase()) {
                "madrid" -> ClimaResponse("Madrid", 22.0, "Soleado", 45)
                "barcelona" -> ClimaResponse("Barcelona", 24.0, "Parcialmente nublado", 55)
                "bilbao" -> ClimaResponse("Bilbao", 18.0, "Lluvioso", 80)
                else -> ClimaResponse(request.ciudad, 20.0, "Datos no disponibles", 50)
            }
        }
    }
}

data class ClimaRequest(val ciudad: String)

data class ClimaResponse(
    val ciudad: String,
    val temperatura: Double,
    val condicion: String,
    val humedad: Int
)
```

### 3.2 Usar la funcion en ChatClient

```kotlin
@RestController
@RequestMapping("/api/agente")
class AgenteClimaController(
    private val chatClient: ChatClient
) {

    @GetMapping("/clima")
    fun preguntarClima(@RequestParam pregunta: String): String {
        return chatClient.prompt()
            .user(pregunta)
            .functions("obtenerClima")       // nombre del bean
            .call()
            .content() ?: "Sin respuesta"
    }
}

// Ejemplo:
// GET /api/agente/clima?pregunta=Que tiempo hace en Madrid?
// Respuesta: "En Madrid la temperatura es de 22 grados con cielo soleado
//             y una humedad del 45%."
```

### 3.3 FunctionCallback para mas control

Si necesitas mas control sobre la definicion de la funcion:

```kotlin
import org.springframework.ai.model.function.FunctionCallback
import org.springframework.ai.model.function.FunctionCallbackWrapper

@Configuration
class FunctionCallbackConfig {

    @Bean
    fun consultarPedidoCallback(): FunctionCallback {
        return FunctionCallbackWrapper.builder(::consultarPedido)
            .withName("consultarPedido")
            .withDescription("Consulta el estado de un pedido por su ID numerico")
            .withInputType(PedidoRequest::class.java)
            .build()
    }

    private fun consultarPedido(request: PedidoRequest): PedidoResponse {
        // En produccion, consultarias la base de datos real
        return PedidoResponse(
            id = request.pedidoId,
            estado = "enviado",
            tracking = "ES${request.pedidoId}789",
            fechaEstimada = "2024-12-20"
        )
    }
}

data class PedidoRequest(val pedidoId: Long)

data class PedidoResponse(
    val id: Long,
    val estado: String,
    val tracking: String,
    val fechaEstimada: String
)
```

### 3.4 Multiples funciones en una llamada

```kotlin
@GetMapping("/asistente")
fun asistente(@RequestParam pregunta: String): String {
    return chatClient.prompt()
        .system("""
            Eres un asistente de una tienda online. Puedes:
            - Consultar el estado de pedidos
            - Obtener el clima de una ciudad
            - Buscar productos en el catalogo
            Responde en espanol de forma amable.
        """.trimIndent())
        .user(pregunta)
        .functions("consultarPedido", "obtenerClima", "buscarProducto")
        .call()
        .content() ?: "Sin respuesta"
}
// El LLM decide automaticamente cual(es) funcion(es) usar segun la pregunta
```

---

## 4. Construir herramientas personalizadas

### 4.1 Herramienta de consulta a base de datos

```kotlin
@Configuration
class DatabaseToolConfig(
    private val productoRepository: ProductoRepository
) {

    @Bean
    @Description("Busca productos en el catalogo por nombre o categoria. Devuelve lista de productos con precio y stock.")
    fun buscarProducto(): java.util.function.Function<BuscarProductoRequest, BuscarProductoResponse> {
        return java.util.function.Function { request ->
            val productos = when {
                request.categoria != null ->
                    productoRepository.findByCategoriaIgnoreCase(request.categoria)
                request.nombre != null ->
                    productoRepository.findByNombreContainingIgnoreCase(request.nombre)
                else -> emptyList()
            }

            BuscarProductoResponse(
                resultados = productos.map { p ->
                    ProductoInfo(
                        id = p.id,
                        nombre = p.nombre,
                        precio = p.precio,
                        stock = p.stock,
                        categoria = p.categoria?.nombre ?: "Sin categoria"
                    )
                },
                total = productos.size
            )
        }
    }
}

data class BuscarProductoRequest(
    val nombre: String? = null,
    val categoria: String? = null
)

data class BuscarProductoResponse(
    val resultados: List<ProductoInfo>,
    val total: Int
)

data class ProductoInfo(
    val id: Long,
    val nombre: String,
    val precio: java.math.BigDecimal,
    val stock: Int,
    val categoria: String
)
```

### 4.2 Herramienta de llamada a API externa

```kotlin
@Configuration
class ExternalApiToolConfig(
    private val webClient: WebClient
) {

    @Bean
    @Description("Convierte una cantidad de una moneda a otra usando tasas de cambio actuales")
    fun convertirMoneda(): java.util.function.Function<ConvertirMonedaRequest, ConvertirMonedaResponse> {
        return java.util.function.Function { request ->
            try {
                // Llamar a una API de tasas de cambio
                val response = webClient.get()
                    .uri("https://api.exchangerate-api.com/v4/latest/${request.monedaOrigen}")
                    .retrieve()
                    .bodyToMono(ExchangeRateResponse::class.java)
                    .block()

                val tasa = response?.rates?.get(request.monedaDestino)
                    ?: throw RuntimeException("Moneda no encontrada")

                val resultado = request.cantidad * tasa

                ConvertirMonedaResponse(
                    cantidadOriginal = request.cantidad,
                    monedaOrigen = request.monedaOrigen,
                    cantidadConvertida = resultado,
                    monedaDestino = request.monedaDestino,
                    tasaCambio = tasa
                )
            } catch (e: Exception) {
                ConvertirMonedaResponse(
                    cantidadOriginal = request.cantidad,
                    monedaOrigen = request.monedaOrigen,
                    cantidadConvertida = 0.0,
                    monedaDestino = request.monedaDestino,
                    tasaCambio = 0.0,
                    error = "Error al obtener tasa de cambio: ${e.message}"
                )
            }
        }
    }
}

data class ConvertirMonedaRequest(
    val cantidad: Double,
    val monedaOrigen: String,    // "EUR", "USD", etc.
    val monedaDestino: String
)

data class ConvertirMonedaResponse(
    val cantidadOriginal: Double,
    val monedaOrigen: String,
    val cantidadConvertida: Double,
    val monedaDestino: String,
    val tasaCambio: Double,
    val error: String? = null
)
```

### 4.3 Herramienta de calculo

```kotlin
@Configuration
class CalculoToolConfig {

    @Bean
    @Description("Calcula metricas financieras: margen de beneficio, ROI, punto de equilibrio")
    fun calcularFinanzas(): java.util.function.Function<CalculoRequest, CalculoResponse> {
        return java.util.function.Function { request ->
            when (request.operacion) {
                "margen" -> {
                    val margen = ((request.ingresos - request.costes) / request.ingresos) * 100
                    CalculoResponse(
                        resultado = margen,
                        unidad = "porcentaje",
                        descripcion = "Margen de beneficio: ${String.format("%.2f", margen)}%"
                    )
                }
                "roi" -> {
                    val roi = ((request.ingresos - request.costes) / request.costes) * 100
                    CalculoResponse(
                        resultado = roi,
                        unidad = "porcentaje",
                        descripcion = "ROI: ${String.format("%.2f", roi)}%"
                    )
                }
                "punto_equilibrio" -> {
                    val pe = request.costeFijo / (request.precioUnitario - request.costeVariable)
                    CalculoResponse(
                        resultado = pe,
                        unidad = "unidades",
                        descripcion = "Punto de equilibrio: ${pe.toInt()} unidades"
                    )
                }
                else -> CalculoResponse(
                    resultado = 0.0,
                    unidad = "error",
                    descripcion = "Operacion no reconocida: ${request.operacion}"
                )
            }
        }
    }
}

data class CalculoRequest(
    val operacion: String,           // "margen", "roi", "punto_equilibrio"
    val ingresos: Double = 0.0,
    val costes: Double = 0.0,
    val costeFijo: Double = 0.0,
    val costeVariable: Double = 0.0,
    val precioUnitario: Double = 0.0
)

data class CalculoResponse(
    val resultado: Double,
    val unidad: String,
    val descripcion: String
)
```

---

## 5. Arquitectura ReAct: razonamiento y accion

### 5.1 Que es ReAct

ReAct (Reasoning + Acting) es un patron donde el LLM alterna entre razonar
sobre la tarea y ejecutar acciones:

```
Pregunta: "Cual es el producto mas caro de la categoria electronica y cuanto 
           costaria en dolares?"

Pensamiento 1: Necesito buscar productos de la categoria electronica
Accion 1: buscarProducto(categoria="electronica")
Observacion 1: [TV 4K: 899EUR, Portatil: 1299EUR, Auriculares: 79EUR]

Pensamiento 2: El mas caro es el Portatil a 1299EUR. Ahora necesito convertir a dolares
Accion 2: convertirMoneda(cantidad=1299, origen="EUR", destino="USD")
Observacion 2: {cantidadConvertida: 1414.91, tasaCambio: 1.089}

Pensamiento 3: Tengo toda la informacion necesaria
Respuesta Final: El producto mas caro de la categoria electronica es el Portatil,
                  con un precio de 1,299 EUR (aproximadamente 1,414.91 USD).
```

### 5.2 Implementacion con Spring AI

Spring AI maneja el ciclo ReAct automaticamente cuando usas function calling.
El ChatClient re-envia los resultados de las funciones al modelo hasta que el
modelo decide dar una respuesta final (sin llamar a mas funciones):

```kotlin
@RestController
@RequestMapping("/api/agente")
class ReActController(
    private val chatClient: ChatClient
) {

    @PostMapping("/react")
    fun agenteReAct(@RequestBody request: AgenteRequest): AgenteResponse {
        val systemPrompt = """
            Eres un agente inteligente de una tienda online. 
            
            Tienes acceso a las siguientes herramientas:
            - buscarProducto: buscar productos por nombre o categoria
            - consultarPedido: ver el estado de un pedido
            - convertirMoneda: convertir entre monedas
            - calcularFinanzas: calculos financieros
            
            Proceso:
            1. Analiza la pregunta del usuario
            2. Decide que herramientas necesitas
            3. Usa las herramientas en el orden necesario
            4. Combina los resultados para dar una respuesta completa
            
            Si necesitas multiples pasos, ejecutalos en orden.
            Responde en espanol de forma clara y amable.
        """.trimIndent()

        val respuesta = chatClient.prompt()
            .system(systemPrompt)
            .user(request.mensaje)
            .functions(
                "buscarProducto",
                "consultarPedido",
                "convertirMoneda",
                "calcularFinanzas"
            )
            .call()
            .content() ?: "No pude procesar tu solicitud"

        return AgenteResponse(respuesta = respuesta)
    }
}

data class AgenteRequest(val mensaje: String)
data class AgenteResponse(val respuesta: String)
```

### 5.3 Flujo interno de Spring AI con funciones

```
Lo que ocurre internamente:

1. ChatClient envia al LLM: mensaje + definiciones de funciones
2. LLM responde: "quiero llamar a buscarProducto(categoria='electronica')"
3. Spring AI ejecuta buscarProducto automaticamente
4. Spring AI re-envia al LLM: resultado de buscarProducto
5. LLM responde: "quiero llamar a convertirMoneda(cantidad=1299, ...)"
6. Spring AI ejecuta convertirMoneda
7. Spring AI re-envia al LLM: resultado de convertirMoneda
8. LLM responde: respuesta final en texto (sin mas tool calls)
9. ChatClient devuelve la respuesta final

Todo esto ocurre dentro de una sola llamada a chatClient.prompt()...call()
```

---

## 6. Agentes multi-herramienta

### 6.1 Agente de soporte completo

```kotlin
@Configuration
class SoporteAgentConfig(
    private val pedidoRepository: PedidoRepository,
    private val productoRepository: ProductoRepository,
    private val clienteRepository: ClienteRepository,
    private val emailService: EmailService
) {

    @Bean
    @Description("Consulta informacion de un cliente por su email o ID")
    fun consultarCliente(): java.util.function.Function<ClienteQuery, ClienteInfo> {
        return java.util.function.Function { query ->
            val cliente = when {
                query.email != null -> clienteRepository.findByEmail(query.email)
                query.clienteId != null -> clienteRepository.findById(query.clienteId).orElse(null)
                else -> null
            }
            cliente?.let {
                ClienteInfo(it.id, it.nombre, it.email, it.fechaRegistro.toString())
            } ?: ClienteInfo(0, "No encontrado", "", "")
        }
    }

    @Bean
    @Description("Consulta los pedidos de un cliente. Puede filtrar por estado: pendiente, enviado, entregado, cancelado")
    fun consultarPedidosCliente(): java.util.function.Function<PedidosQuery, PedidosResponse> {
        return java.util.function.Function { query ->
            val pedidos = if (query.estado != null) {
                pedidoRepository.findByClienteIdAndEstado(query.clienteId, query.estado)
            } else {
                pedidoRepository.findByClienteId(query.clienteId)
            }
            PedidosResponse(
                pedidos = pedidos.map { p ->
                    PedidoResumen(p.id, p.estado, p.total.toString(), p.fechaCreacion.toString())
                }
            )
        }
    }

    @Bean
    @Description("Envia un email al equipo de soporte con un resumen del problema del cliente")
    fun escalarASoporte(): java.util.function.Function<EscalarRequest, EscalarResponse> {
        return java.util.function.Function { request ->
            emailService.enviar(
                to = "soporte@empresa.com",
                subject = "Escalacion: ${request.asunto}",
                body = """
                    Cliente: ${request.clienteEmail}
                    Problema: ${request.descripcion}
                    Prioridad: ${request.prioridad}
                """.trimIndent()
            )
            EscalarResponse(
                exito = true,
                mensaje = "Ticket creado y asignado al equipo de soporte"
            )
        }
    }
}

// Data classes para las herramientas de soporte
data class ClienteQuery(val email: String? = null, val clienteId: Long? = null)
data class ClienteInfo(val id: Long, val nombre: String, val email: String, val fechaRegistro: String)
data class PedidosQuery(val clienteId: Long, val estado: String? = null)
data class PedidosResponse(val pedidos: List<PedidoResumen>)
data class PedidoResumen(val id: Long, val estado: String, val total: String, val fecha: String)
data class EscalarRequest(val clienteEmail: String, val asunto: String, val descripcion: String, val prioridad: String)
data class EscalarResponse(val exito: Boolean, val mensaje: String)
```

### 6.2 Controlador del agente de soporte

```kotlin
@RestController
@RequestMapping("/api/soporte")
class SoporteAgenteController(
    private val chatClient: ChatClient
) {

    @PostMapping
    fun agenteSoporte(@RequestBody request: SoporteRequest): SoporteResponse {
        val respuesta = chatClient.prompt()
            .system("""
                Eres un agente de soporte al cliente de una tienda online.
                
                Flujo recomendado:
                1. Identifica al cliente (por email o ID)
                2. Consulta sus pedidos si es relevante
                3. Intenta resolver el problema con la informacion disponible
                4. Si no puedes resolver, escala al equipo humano
                
                Se amable y profesional. Responde en espanol.
                Nunca reveles datos internos del sistema.
            """.trimIndent())
            .user(request.mensaje)
            .functions(
                "consultarCliente",
                "consultarPedidosCliente",
                "buscarProducto",
                "escalarASoporte"
            )
            .call()
            .content() ?: "Lo siento, no pude procesar tu solicitud."

        return SoporteResponse(respuesta = respuesta)
    }
}

data class SoporteRequest(val mensaje: String)
data class SoporteResponse(val respuesta: String)
```

---

## 7. Memoria y estado de los agentes

### 7.1 Agente con memoria de conversacion

```kotlin
@Configuration
class AgenteConMemoriaConfig {

    @Bean
    fun agenteChatClient(
        chatModel: ChatModel,
        chatMemory: ChatMemory
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("""
                Eres un agente de soporte con acceso a herramientas.
                Recuerdas toda la conversacion con el usuario.
                Si ya obtuviste informacion del cliente, no la vuelvas a consultar.
            """.trimIndent())
            .defaultAdvisors(
                MessageChatMemoryAdvisor(chatMemory)
            )
            .build()
    }
}

@RestController
@RequestMapping("/api/agente")
class AgenteConMemoriaController(
    private val chatClient: ChatClient
) {

    @PostMapping("/chat")
    fun chat(
        @RequestParam sessionId: String,
        @RequestBody request: AgenteRequest
    ): AgenteResponse {
        val respuesta = chatClient.prompt()
            .user(request.mensaje)
            .functions("consultarCliente", "consultarPedidosCliente", "buscarProducto")
            .advisors { advisor ->
                advisor.param(ChatMemory.CONVERSATION_ID, sessionId)
            }
            .call()
            .content() ?: "Sin respuesta"

        return AgenteResponse(respuesta = respuesta)
    }
}

// Conversacion de ejemplo:
// User: "Soy juan@email.com, que pedidos tengo?"
// Agente: [llama consultarCliente] [llama consultarPedidosCliente] "Tienes 3 pedidos..."
// User: "El pedido 456, cuando llega?"
// Agente: [ya sabe quien es el usuario, consulta solo el pedido 456]
```

### 7.2 Estado persistente entre sesiones

```kotlin
// Para agentes que necesitan mantener estado mas alla de la conversacion
@Entity
@Table(name = "agent_state")
class AgentState(
    @Id
    val sessionId: String,

    @Column(columnDefinition = "jsonb")
    var state: String = "{}",          // estado serializado como JSON

    @Column(name = "updated_at")
    var updatedAt: LocalDateTime = LocalDateTime.now()
)

@Service
class StatefulAgentService(
    private val chatClient: ChatClient,
    private val stateRepository: AgentStateRepository
) {

    fun procesarConEstado(sessionId: String, mensaje: String): String {
        // Cargar estado previo
        val estado = stateRepository.findById(sessionId)
            .map { ObjectMapper().readValue(it.state, AgentContext::class.java) }
            .orElse(AgentContext())

        // Incluir estado en el system prompt
        val systemConEstado = """
            Eres un agente de soporte. Estado actual de la sesion:
            - Cliente identificado: ${estado.clienteEmail ?: "no"}
            - Pedidos consultados: ${estado.pedidosConsultados.joinToString(", ")}
            - Problema: ${estado.problemaDescrito ?: "no definido"}
        """.trimIndent()

        val respuesta = chatClient.prompt()
            .system(systemConEstado)
            .user(mensaje)
            .functions("consultarCliente", "consultarPedidosCliente")
            .call()
            .content() ?: "Sin respuesta"

        // Actualizar estado (simplificado)
        // En produccion, extraerias el estado del flujo de la conversacion
        stateRepository.save(AgentState(
            sessionId = sessionId,
            state = ObjectMapper().writeValueAsString(estado),
            updatedAt = LocalDateTime.now()
        ))

        return respuesta
    }
}

data class AgentContext(
    var clienteEmail: String? = null,
    var pedidosConsultados: List<Long> = emptyList(),
    var problemaDescrito: String? = null
)
```

---

## 8. Manejo de errores y guardrails

### 8.1 Errores en herramientas

Las herramientas pueden fallar. Debes manejar los errores de forma que el agente
pueda continuar:

```kotlin
@Bean
@Description("Consulta informacion en tiempo real de una API externa")
fun consultarApiExterna(): java.util.function.Function<ApiExternaRequest, ApiExternaResponse> {
    return java.util.function.Function { request ->
        try {
            // Llamada a API externa que puede fallar
            val resultado = webClient.get()
                .uri(request.url)
                .retrieve()
                .bodyToMono(String::class.java)
                .timeout(Duration.ofSeconds(5))     // timeout de 5 segundos
                .block()

            ApiExternaResponse(
                exito = true,
                datos = resultado ?: "Sin datos",
                error = null
            )
        } catch (e: WebClientResponseException) {
            ApiExternaResponse(
                exito = false,
                datos = null,
                error = "Error HTTP ${e.statusCode}: ${e.statusText}"
            )
        } catch (e: Exception) {
            ApiExternaResponse(
                exito = false,
                datos = null,
                error = "Error: ${e.message}"
            )
        }
    }
}

data class ApiExternaRequest(val url: String)
data class ApiExternaResponse(val exito: Boolean, val datos: String?, val error: String?)
```

### 8.2 Limitar el numero de llamadas a herramientas

```kotlin
// Prevenir bucles infinitos de function calling
@Service
class SafeAgentService(
    private val chatClient: ChatClient
) {

    private val maxToolCalls = 10    // maximo de herramientas por interaccion

    fun ejecutarConLimite(mensaje: String): String {
        // Spring AI no expone directamente un limite de tool calls,
        // pero puedes configurarlo via propiedades del proveedor
        return chatClient.prompt()
            .system("""
                Maximo de herramientas por respuesta: $maxToolCalls.
                Si necesitas mas de $maxToolCalls pasos, resume lo que tienes
                y pide al usuario que sea mas especifico.
            """.trimIndent())
            .user(mensaje)
            .functions("consultarCliente", "consultarPedidosCliente")
            .call()
            .content() ?: "Sin respuesta"
    }
}
```

### 8.3 Guardrails: que puede y que no puede hacer el agente

```kotlin
// Definir claramente los limites del agente
val systemPromptConGuardrails = """
    Eres un agente de soporte al cliente.
    
    PUEDES:
    - Consultar informacion de clientes y pedidos
    - Buscar productos en el catalogo
    - Escalar problemas al equipo humano
    - Responder preguntas sobre politicas de la empresa
    
    NO PUEDES:
    - Modificar pedidos (cancelar, cambiar direccion)
    - Emitir reembolsos
    - Acceder a datos de pago (tarjetas, cuentas bancarias)
    - Compartir informacion de un cliente con otro
    - Ejecutar acciones destructivas
    
    Si el usuario pide algo que no puedes hacer, explica amablemente
    por que no y sugiere contactar al equipo humano.
""".trimIndent()
```

### 8.4 Logging y auditoria de acciones del agente

```kotlin
@Component
class AgentAuditLogger {

    fun registrarAccion(
        sessionId: String,
        funcion: String,
        parametros: Any,
        resultado: Any,
        duracion: Long
    ) {
        logger.info("""
            [AGENT AUDIT] session=$sessionId
            funcion=$funcion
            parametros=$parametros
            duracion=${duracion}ms
            resultado=${resultado.toString().take(500)}
        """.trimIndent())

        // En produccion: enviar a sistema de auditoria
        // auditoriaService.registrar(...)
    }
}
```

---

## 9. Cuando usar agentes vs RAG simple

### 9.1 Criterios de decision

```
Usa RAG simple cuando:
  - El usuario necesita informacion de documentos estaticos
  - No hay acciones que tomar, solo consultas de informacion
  - Los datos no cambian en tiempo real
  - La respuesta se basa en textos existentes
  Ejemplos: FAQ, documentacion, base de conocimiento

Usa agentes con function calling cuando:
  - Necesitas consultar datos en tiempo real (BD, APIs)
  - El usuario quiere ejecutar acciones (crear pedido, enviar email)
  - Se requieren multiples pasos de razonamiento
  - Necesitas combinar informacion de varias fuentes
  Ejemplos: soporte al cliente, asistente de ventas, automatizacion

Combina ambos (RAG + Agentes) cuando:
  - Necesitas conocimiento de documentos Y acciones en tiempo real
  - El agente necesita contexto de la documentacion para decidir acciones
  Ejemplo: agente de soporte que consulta la politica de devoluciones (RAG)
           y luego procesa la devolucion (function calling)
```

### 9.2 Tabla comparativa

| Aspecto | RAG simple | Agente con tools |
|---------|-----------|-----------------|
| Complejidad | Baja | Media-Alta |
| Latencia | 1-3 seg (1 llamada LLM) | 3-15 seg (multiples llamadas) |
| Coste | Bajo (1 llamada) | Medio-Alto (N llamadas) |
| Riesgo | Bajo (solo lee datos) | Medio (ejecuta acciones) |
| Trazabilidad | Facil (documentos usados) | Media (cadena de acciones) |
| Mantenimiento | Bajo | Alto (herramientas, errores, guardrails) |

### 9.3 Patron hibrido: RAG + Agente

```kotlin
@Configuration
class HybridAgentConfig(
    private val vectorStore: VectorStore
) {

    @Bean
    @Description("Busca en la base de conocimiento de la empresa: politicas, procedimientos, manuales")
    fun buscarEnDocumentacion(): java.util.function.Function<DocQuery, DocResponse> {
        return java.util.function.Function { query ->
            val docs = vectorStore.similaritySearch(
                SearchRequest.builder()
                    .query(query.consulta)
                    .topK(3)
                    .similarityThreshold(0.7)
                    .build()
            )
            DocResponse(
                documentos = docs.map { it.content },
                encontrados = docs.size
            )
        }
    }
}

data class DocQuery(val consulta: String)
data class DocResponse(val documentos: List<String>, val encontrados: Int)

// Ahora el agente puede buscar en la documentacion Y ejecutar acciones:
// User: "Quiero devolver mi pedido 789"
// Agente: [buscarEnDocumentacion("politica devoluciones")]
//         -> Lee que las devoluciones se aceptan hasta 30 dias
// Agente: [consultarPedido(789)]
//         -> Ve que el pedido fue hace 15 dias
// Agente: "Tu pedido es elegible para devolucion. Segun nuestra politica,
//          tienes 30 dias y tu pedido tiene 15. Quieres que inicie el proceso?"
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|---------|--------|-------------|
| Agente | LLM que puede razonar y ejecutar acciones via herramientas | ChatClient con `.functions(...)` |
| Function calling | Mecanismo por el que el LLM invoca funciones externas | LLM elige funcion y parametros; tu codigo la ejecuta |
| @Description | Anotacion que describe una funcion para el LLM | En el `@Bean` que define la herramienta |
| FunctionCallback | Definicion programatica de herramienta con mas control | `FunctionCallbackWrapper.builder(...)` |
| ReAct | Patron de pensar-actuar-observar-repetir | Automatico en Spring AI con function calling |
| Multi-herramienta | Agente con acceso a multiples funciones | Listar todas en `.functions("f1", "f2", "f3")` |
| Guardrails | Limites de lo que el agente puede hacer | System prompt con reglas PUEDES/NO PUEDES |
| Memoria de agente | Historial de conversacion del agente | `MessageChatMemoryAdvisor` con `ChatMemory` |
| Estado persistente | Estado del agente entre sesiones | Entidad JPA con JSON del contexto |
| RAG + Agente | Combinar busqueda documental con acciones | Tool que consulta VectorStore + tools de accion |
| Auditoria | Registrar todas las acciones del agente | Logger/servicio que captura funcion+params+resultado |
| Limite de llamadas | Prevenir bucles infinitos de tool calls | System prompt + configuracion del proveedor |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **Function calling basico** | Crear dos herramientas (clima y conversor de moneda) con `@Bean` + `@Description`, registrarlas en ChatClient y probar que el LLM las invoca correctamente segun la pregunta del usuario | `@Description`, function beans, ChatClient.functions(), request/response DTOs |
| 02 | **Agente ReAct** | Construir un agente de tienda online que pueda buscar productos, consultar pedidos y hacer calculos. Probar con preguntas que requieran multiples pasos (ej: "Producto mas caro en EUR y su precio en USD") | ReAct, multi-step reasoning, multiples tool calls |
| 03 | **Multi-herramienta con BD** | Crear un agente de soporte con herramientas que consulten repositorios JPA reales (clientes, pedidos, productos) y una herramienta que escale al equipo humano. Probar con escenarios reales de soporte | JPA repositories como tools, escalacion, guardrails |
| 04 | **Agente con memoria** | Anadir memoria de conversacion al agente de soporte para que recuerde al usuario entre mensajes. Implementar estado persistente en BD y probar conversaciones de multiples turnos | ChatMemory, conversationId, estado persistente, JPA |

---

## Recursos

- [Spring AI Function Calling](https://docs.spring.io/spring-ai/reference/api/functions.html)
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
- [ReAct Paper (Yao et al. 2022)](https://arxiv.org/abs/2210.03629)
- [Building Agents with Spring AI](https://spring.io/blog)
- [Anthropic Tool Use](https://docs.anthropic.com/en/docs/tool-use)
- [LangChain Agents Concepts](https://python.langchain.com/docs/concepts/agents/)

---

> **Consejo practico**: los agentes son poderosos pero complejos. En produccion, empieza
> con el minimo de herramientas necesarias (2-3 maximo) y ve anadiendo mas a medida que
> validas que el agente las usa correctamente. Cada herramienta adicional aumenta la
> probabilidad de que el agente se equivoque al elegir cual usar. Y siempre: guardrails
> primero, funcionalidad despues.
