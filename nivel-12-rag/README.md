# Nivel 12 — RAG (Retrieval-Augmented Generation)

> Los LLMs son potentes pero tienen dos limitaciones criticas: no conocen tus datos privados
> y su conocimiento tiene una fecha de corte. RAG resuelve ambos problemas: busca informacion
> relevante en tus documentos y la inyecta en el prompt antes de que el modelo responda.
> Ya sabes usar embeddings (nivel 09), Spring AI (nivel 10) y bases de datos vectoriales
> (nivel 11). Ahora conectas todas las piezas en un pipeline de RAG completo.

---

## Contenido

1. [Que es RAG y por que existe](#1-que-es-rag-y-por-que-existe)
2. [Arquitectura de un sistema RAG](#2-arquitectura-de-un-sistema-rag)
3. [Carga de documentos](#3-carga-de-documentos)
4. [Estrategias de chunking](#4-estrategias-de-chunking)
5. [Generacion y almacenamiento de embeddings](#5-generacion-y-almacenamiento-de-embeddings)
6. [Recuperacion (Retrieval)](#6-recuperacion-retrieval)
7. [Augmentacion del prompt](#7-augmentacion-del-prompt)
8. [Pipeline RAG con Spring AI](#8-pipeline-rag-con-spring-ai)
9. [Evaluacion y metricas](#9-evaluacion-y-metricas)
10. [Errores comunes y soluciones](#10-errores-comunes-y-soluciones)
11. [Resumen](#resumen)
12. [Ejercicios](#ejercicios)
13. [Recursos](#recursos)

---

## 1. Que es RAG y por que existe

### 1.1 Las limitaciones de los LLMs

Los LLMs tienen tres limitaciones fundamentales que RAG resuelve:

```
1. Fecha de corte del conocimiento (Knowledge Cutoff)
   - GPT-4o fue entrenado con datos hasta cierta fecha
   - No sabe nada sobre eventos posteriores a esa fecha
   - No conoce tus datos internos de empresa

2. Alucinaciones (Hallucinations)
   - Cuando el modelo no sabe algo, puede inventarlo
   - Genera respuestas convincentes pero falsas
   - Especialmente problematico para datos especificos (precios, fechas, politicas)

3. Ventana de contexto limitada
   - No puedes meter toda tu base de conocimiento en un prompt
   - Necesitas seleccionar solo la informacion relevante
```

### 1.2 Como RAG resuelve estos problemas

RAG = **R**etrieval-**A**ugmented **G**eneration

```
Sin RAG:
  Usuario: "Cual es la politica de devolucion de mi empresa?"
  LLM: "No tengo informacion sobre tu empresa..." (o peor, inventa una politica)

Con RAG:
  1. Usuario pregunta: "Cual es la politica de devolucion?"
  2. Sistema busca en la base de conocimiento: encuentra el documento de politicas
  3. Inyecta el documento relevante en el prompt
  4. LLM responde basandose en el documento real: "Segun vuestra politica, 
     las devoluciones se aceptan hasta 30 dias despues de la compra..."
```

### 1.3 RAG vs Fine-tuning

| Aspecto | RAG | Fine-tuning |
|---------|-----|-------------|
| Actualizacion de datos | Instantanea (actualizar documentos) | Requiere re-entrenar el modelo |
| Coste | Bajo (solo embeddings + BD) | Alto (GPU, datos de entrenamiento) |
| Trazabilidad | Alta (sabes que documentos se usaron) | Baja (el conocimiento esta "dentro" del modelo) |
| Precision con datos especificos | Alta | Variable |
| Complejidad | Media (pipeline de ingestion + retrieval) | Alta (entrenamiento, evaluacion) |
| Cuando usar | Datos que cambian, documentacion, soporte | Comportamiento especifico, estilo, tareas nuevas |

**Regla practica**: empieza siempre con RAG. Solo considera fine-tuning si RAG no es
suficiente para tu caso de uso.

---

## 2. Arquitectura de un sistema RAG

### 2.1 Los dos pipelines

Un sistema RAG tiene dos pipelines que funcionan en momentos diferentes:

```
Pipeline de Ingestion (offline, cuando cargas documentos):

  Documentos (PDF, web, BD)
       |
       v
  Carga de documentos (DocumentReader)
       |
       v
  Chunking / Splitting (DocumentTransformer)
       |
       v
  Generacion de embeddings (EmbeddingModel)
       |
       v
  Almacenamiento (VectorStore)


Pipeline de Consulta (online, cuando el usuario pregunta):

  Pregunta del usuario
       |
       v
  Generacion de embedding de la pregunta
       |
       v
  Busqueda por similitud en VectorStore
       |
       v
  Recuperacion de chunks relevantes (top-K)
       |
       v
  Construccion del prompt (pregunta + contexto)
       |
       v
  LLM genera la respuesta
       |
       v
  Respuesta al usuario
```

### 2.2 Componentes en Spring AI

```
Spring AI mapea cada paso a una abstraccion:

DocumentReader     -> Carga documentos de distintas fuentes
DocumentTransformer -> Transforma documentos (chunking, limpieza)
DocumentWriter     -> Escribe documentos al VectorStore
VectorStore        -> Almacena y busca por similitud
EmbeddingModel     -> Genera embeddings
ChatClient         -> Interactua con el LLM
```

---

## 3. Carga de documentos

### 3.1 DocumentReader de Spring AI

Spring AI proporciona lectores para distintas fuentes de documentos:

```kotlin
import org.springframework.ai.reader.TextReader
import org.springframework.ai.reader.JsonReader
import org.springframework.ai.document.Document

// Leer un archivo de texto
@Service
class CargaDocumentosService {

    // Desde un archivo de texto plano
    fun cargarTexto(recurso: Resource): List<Document> {
        val reader = TextReader(recurso)
        reader.customMetadata["fuente"] = "manual-interno"  // metadatos adicionales
        return reader.get()
    }

    // Desde un archivo JSON
    fun cargarJson(recurso: Resource): List<Document> {
        val reader = JsonReader(recurso)
        return reader.get()
    }
}
```

### 3.2 Lector de PDFs

```xml
<!-- Dependencia para leer PDFs -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pdf-document-reader</artifactId>
</dependency>
```

```kotlin
import org.springframework.ai.reader.pdf.PagePdfDocumentReader
import org.springframework.ai.reader.pdf.config.PdfDocumentReaderConfig

fun cargarPdf(recurso: Resource): List<Document> {
    val config = PdfDocumentReaderConfig.builder()
        .withPageExtractedTextFormatter(
            ExtractedTextFormatter.builder()
                .withNumberOfBottomTextLinesToDelete(3)   // eliminar footer
                .withNumberOfTopPagesToSkipBeforeDelete(1) // saltar portada
                .build()
        )
        .withPagesPerDocument(1)    // un Document por pagina
        .build()

    val reader = PagePdfDocumentReader(recurso, config)
    return reader.get()
}
```

### 3.3 Lector de paginas web

```kotlin
import org.springframework.ai.reader.tika.TikaDocumentReader

// Tika puede leer HTML, PDF, Word, Excel, etc.
fun cargarWeb(url: String): List<Document> {
    val reader = TikaDocumentReader(url)
    return reader.get()
}
```

```xml
<!-- Dependencia para Tika -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-tika-document-reader</artifactId>
</dependency>
```

### 3.4 Lector personalizado desde base de datos

```kotlin
@Service
class DatabaseDocumentReader(
    private val jdbcTemplate: JdbcTemplate
) {

    fun cargarArticulos(): List<Document> {
        return jdbcTemplate.query(
            "SELECT id, titulo, contenido, categoria, fecha_publicacion FROM articulos WHERE publicado = true"
        ) { rs, _ ->
            Document(
                rs.getString("contenido"),               // texto del documento
                mapOf(
                    "id" to rs.getLong("id"),
                    "titulo" to rs.getString("titulo"),
                    "categoria" to rs.getString("categoria"),
                    "fecha" to rs.getString("fecha_publicacion"),
                    "fuente" to "base_de_datos"
                )
            )
        }
    }
}
```

---

## 4. Estrategias de chunking

### 4.1 Por que hacer chunking

Los documentos completos suelen ser demasiado grandes para:
- Generar un embedding unico que capture todo el significado
- Caber en la ventana de contexto del LLM junto con la pregunta
- Ser relevantes en su totalidad (solo partes son relevantes)

El **chunking** divide documentos en fragmentos mas pequenos y manejables.

### 4.2 Chunking de tamano fijo

La estrategia mas simple: cortar cada N caracteres con solapamiento (overlap):

```kotlin
import org.springframework.ai.transformer.splitter.TokenTextSplitter

@Service
class ChunkingService {

    // Chunking por tokens (recomendado)
    fun chunkPorTokens(documentos: List<Document>): List<Document> {
        val splitter = TokenTextSplitter.builder()
            .withChunkSize(800)            // tokens por chunk
            .withMinChunkSizeChars(350)    // tamano minimo para evitar chunks diminutos
            .withMinChunkLengthToEmbed(5)  // minimo para generar embedding
            .withMaxNumChunks(10000)       // limite total de chunks
            .withKeepSeparator(true)       // mantener separadores
            .build()

        return splitter.apply(documentos)
    }
}
```

### 4.3 Chunking recursivo

Intenta cortar por limites naturales del texto (parrafos, frases, palabras):

```kotlin
// Chunking recursivo: respeta la estructura del texto
fun chunkRecursivo(texto: String, maxSize: Int = 1000, overlap: Int = 200): List<String> {
    val separadores = listOf("\n\n", "\n", ". ", " ", "")
    return dividirRecursivo(texto, separadores, maxSize, overlap)
}

private fun dividirRecursivo(
    texto: String,
    separadores: List<String>,
    maxSize: Int,
    overlap: Int
): List<String> {
    if (texto.length <= maxSize) return listOf(texto)

    val separador = separadores.firstOrNull { texto.contains(it) } ?: ""
    val partes = texto.split(separador)

    val chunks = mutableListOf<String>()
    var actual = StringBuilder()

    for (parte in partes) {
        if (actual.length + parte.length + separador.length > maxSize) {
            if (actual.isNotEmpty()) {
                chunks.add(actual.toString().trim())
                // Overlap: mantener el final del chunk anterior
                val overlapTexto = actual.toString().takeLast(overlap)
                actual = StringBuilder(overlapTexto)
            }
        }
        if (actual.isNotEmpty()) actual.append(separador)
        actual.append(parte)
    }

    if (actual.isNotEmpty()) {
        chunks.add(actual.toString().trim())
    }

    return chunks
}
```

### 4.4 Chunking semantico

Agrupa frases que hablan del mismo tema, usando embeddings para detectar cambios de tema:

```kotlin
// Chunking semantico: divide cuando cambia el tema
fun chunkSemantico(
    frases: List<String>,
    embeddingModel: EmbeddingModel,
    umbralSimilitud: Double = 0.75
): List<String> {
    val embeddings = frases.map { embeddingModel.embed(it) }
    val chunks = mutableListOf<String>()
    var chunkActual = mutableListOf(frases[0])

    for (i in 1 until frases.size) {
        val similitud = cosineSimilarity(embeddings[i - 1], embeddings[i])

        if (similitud >= umbralSimilitud) {
            // Frases similares: mismo chunk
            chunkActual.add(frases[i])
        } else {
            // Cambio de tema: nuevo chunk
            chunks.add(chunkActual.joinToString(" "))
            chunkActual = mutableListOf(frases[i])
        }
    }

    if (chunkActual.isNotEmpty()) {
        chunks.add(chunkActual.joinToString(" "))
    }

    return chunks
}
```

### 4.5 Tamano optimo de chunks

| Tamano de chunk | Ventajas | Desventajas | Uso tipico |
|----------------|----------|-------------|------------|
| 100-200 tokens | Alta precision en busqueda | Pierde contexto | FAQs, datos puntuales |
| 300-500 tokens | Buen balance precision/contexto | Puede cortar ideas | Documentacion tecnica |
| 500-1000 tokens | Preserva contexto completo | Menos preciso en busqueda | Articulos, manuales |
| 1000+ tokens | Maximo contexto | Demasiado generico, costoso | Resumenes de documentos |

**Regla practica**: empieza con 500 tokens y overlap de 100-200 tokens. Ajusta segun
los resultados de tus evaluaciones.

---

## 5. Generacion y almacenamiento de embeddings

### 5.1 Pipeline de ingestion completo

```kotlin
@Service
class IngestionService(
    private val embeddingModel: EmbeddingModel,
    private val vectorStore: VectorStore
) {

    // Pipeline completo: cargar -> chunk -> almacenar (embeddings se generan automaticamente)
    fun ingestar(recurso: Resource) {
        // 1. Cargar el documento
        val reader = TikaDocumentReader(recurso)
        val documentos = reader.get()

        // 2. Chunking
        val splitter = TokenTextSplitter.builder()
            .withChunkSize(500)
            .withMinChunkSizeChars(200)
            .build()
        val chunks = splitter.apply(documentos)

        logger.info("Documento cargado: ${documentos.size} docs -> ${chunks.size} chunks")

        // 3. Almacenar (Spring AI genera los embeddings automaticamente)
        vectorStore.add(chunks)

        logger.info("${chunks.size} chunks almacenados en el VectorStore")
    }

    // Ingestion en batch para multiples documentos
    fun ingestarBatch(recursos: List<Resource>) {
        for (recurso in recursos) {
            try {
                ingestar(recurso)
            } catch (e: Exception) {
                logger.error("Error al ingestar ${recurso.filename}: ${e.message}")
            }
        }
    }
}
```

### 5.2 Endpoint de ingestion

```kotlin
@RestController
@RequestMapping("/api/ingestion")
class IngestionController(
    private val ingestionService: IngestionService
) {

    @PostMapping("/texto")
    fun ingestarTexto(@RequestBody request: IngestionRequest): ResponseEntity<String> {
        val documentos = request.textos.mapIndexed { index, texto ->
            Document(
                texto,
                mapOf(
                    "fuente" to request.fuente,
                    "indice" to index,
                    "fecha" to LocalDate.now().toString()
                )
            )
        }

        val splitter = TokenTextSplitter.builder()
            .withChunkSize(500)
            .build()
        val chunks = splitter.apply(documentos)

        ingestionService.almacenarChunks(chunks)

        return ResponseEntity.ok("${chunks.size} chunks almacenados correctamente")
    }
}

data class IngestionRequest(
    val textos: List<String>,
    val fuente: String = "api"
)
```

### 5.3 Enriquecer metadatos durante la ingestion

```kotlin
// Anadir metadatos utiles para filtrado posterior
fun enriquecerDocumento(doc: Document, fuente: String): Document {
    val metadatos = doc.metadata.toMutableMap()
    metadatos["fuente"] = fuente
    metadatos["fecha_ingestion"] = LocalDateTime.now().toString()
    metadatos["longitud_caracteres"] = doc.content.length
    metadatos["longitud_palabras"] = doc.content.split("\\s+".toRegex()).size

    // Extraer titulo si es posible
    val primeraLinea = doc.content.lines().firstOrNull()?.take(100) ?: ""
    metadatos["titulo_inferido"] = primeraLinea

    return Document(doc.content, metadatos)
}
```

---

## 6. Recuperacion (Retrieval)

### 6.1 Busqueda por similitud basica

La forma mas simple de recuperacion:

```kotlin
@Service
class RetrievalService(
    private val vectorStore: VectorStore
) {

    // Top-K por similitud coseno
    fun recuperar(consulta: String, topK: Int = 5): List<Document> {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)
                .topK(topK)
                .similarityThreshold(0.7)    // solo resultados relevantes
                .build()
        )
    }
}
```

### 6.2 Maximum Marginal Relevance (MMR)

El problema de la busqueda top-K pura: los K resultados pueden ser muy similares
entre si, repitiendo la misma informacion.

MMR equilibra **relevancia** y **diversidad**:

```kotlin
// Busqueda con MMR: documentos relevantes pero diversos
fun recuperarConMMR(consulta: String, topK: Int = 5): List<Document> {
    return vectorStore.similaritySearch(
        SearchRequest.builder()
            .query(consulta)
            .topK(topK)
            .similarityThreshold(0.6)
            // MMR: lambda controla balance entre relevancia y diversidad
            // 1.0 = solo relevancia (igual que top-K)
            // 0.0 = solo diversidad
            // 0.5 = balance (recomendado)
            .build()
    )
}
```

### 6.3 Busqueda hibrida (vector + texto)

Combinar busqueda vectorial con busqueda por palabras clave para mejores resultados:

```kotlin
@Service
class HybridRetrievalService(
    private val vectorStore: VectorStore,
    private val jdbcTemplate: JdbcTemplate,
    private val embeddingModel: EmbeddingModel
) {

    fun busquedaHibrida(
        consulta: String,
        topK: Int = 10,
        pesoVector: Double = 0.7,    // peso de la busqueda vectorial
        pesoTexto: Double = 0.3      // peso de la busqueda por texto
    ): List<Document> {
        // 1. Busqueda vectorial
        val resultadosVector = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)
                .topK(topK * 2)    // buscar mas para tener margen
                .build()
        )

        // 2. Busqueda por texto (full-text search de PostgreSQL)
        val resultadosTexto = busquedaPorTexto(consulta, topK * 2)

        // 3. Fusion de resultados (Reciprocal Rank Fusion)
        val scoresFinales = mutableMapOf<String, Double>()
        val documentosPorId = mutableMapOf<String, Document>()

        resultadosVector.forEachIndexed { rank, doc ->
            val rrf = pesoVector * (1.0 / (60 + rank))   // RRF con k=60
            scoresFinales.merge(doc.id, rrf) { a, b -> a + b }
            documentosPorId[doc.id] = doc
        }

        resultadosTexto.forEachIndexed { rank, doc ->
            val rrf = pesoTexto * (1.0 / (60 + rank))
            scoresFinales.merge(doc.id, rrf) { a, b -> a + b }
            documentosPorId[doc.id] = doc
        }

        // 4. Ordenar por score fusionado y devolver top-K
        return scoresFinales.entries
            .sortedByDescending { it.value }
            .take(topK)
            .mapNotNull { documentosPorId[it.key] }
    }

    private fun busquedaPorTexto(consulta: String, limite: Int): List<Document> {
        return jdbcTemplate.query(
            """SELECT id, content, metadata 
               FROM vector_store 
               WHERE to_tsvector('spanish', content) @@ plainto_tsquery('spanish', ?)
               LIMIT ?""",
            arrayOf(consulta, limite)
        ) { rs, _ ->
            Document(rs.getString("content"), mapOf("id" to rs.getString("id")))
        }
    }
}
```

### 6.4 Re-ranking

Despues de la recuperacion inicial, un modelo de re-ranking puede reordenar los
resultados por relevancia mas precisa:

```kotlin
// Re-ranking simple basado en el LLM
fun reRanking(consulta: String, documentos: List<Document>, topK: Int = 3): List<Document> {
    if (documentos.size <= topK) return documentos

    val prompt = """
        Dada la siguiente pregunta y lista de documentos, ordena los documentos 
        del mas relevante al menos relevante. Devuelve solo los indices en orden.
        
        Pregunta: $consulta
        
        Documentos:
        ${documentos.mapIndexed { i, d -> "$i: ${d.content.take(200)}" }.joinToString("\n")}
        
        Responde solo con los indices separados por comas (ej: 2,0,4,1,3)
    """.trimIndent()

    val respuesta = chatClient.prompt().user(prompt).call().content() ?: return documentos.take(topK)

    return try {
        respuesta.trim().split(",")
            .map { it.trim().toInt() }
            .filter { it in documentos.indices }
            .take(topK)
            .map { documentos[it] }
    } catch (e: Exception) {
        documentos.take(topK)    // fallback: devolver los primeros
    }
}
```

---

## 7. Augmentacion del prompt

### 7.1 Prompt stuffing: inyectar contexto

La tecnica mas comun: insertar los documentos recuperados directamente en el prompt:

```kotlin
@Service
class RagService(
    private val chatClient: ChatClient,
    private val retrievalService: RetrievalService
) {

    fun responder(pregunta: String): RagResponse {
        // 1. Recuperar documentos relevantes
        val documentos = retrievalService.recuperar(pregunta, topK = 5)

        if (documentos.isEmpty()) {
            return RagResponse(
                respuesta = "No encontre informacion relevante para responder tu pregunta.",
                documentosUsados = emptyList()
            )
        }

        // 2. Construir el contexto
        val contexto = documentos.mapIndexed { i, doc ->
            """[Documento ${i + 1}]
               Fuente: ${doc.metadata["fuente"] ?: "desconocida"}
               Contenido: ${doc.content}""".trimIndent()
        }.joinToString("\n\n---\n\n")

        // 3. Construir el prompt con contexto
        val systemPrompt = """
            Eres un asistente que responde preguntas basandose UNICAMENTE en el 
            contexto proporcionado. Si la informacion no esta en el contexto, 
            di "No tengo suficiente informacion para responder eso."
            
            No inventes datos. Cita el numero de documento cuando sea posible.
        """.trimIndent()

        val userPrompt = """
            Contexto:
            $contexto
            
            ---
            
            Pregunta: $pregunta
        """.trimIndent()

        // 4. Generar respuesta
        val respuesta = chatClient.prompt()
            .system(systemPrompt)
            .user(userPrompt)
            .call()
            .content() ?: "Error al generar la respuesta"

        return RagResponse(
            respuesta = respuesta,
            documentosUsados = documentos.map { doc ->
                DocumentoRef(
                    fuente = doc.metadata["fuente"]?.toString() ?: "desconocida",
                    fragmento = doc.content.take(200)
                )
            }
        )
    }
}

data class RagResponse(
    val respuesta: String,
    val documentosUsados: List<DocumentoRef>
)

data class DocumentoRef(
    val fuente: String,
    val fragmento: String
)
```

### 7.2 Template para RAG

```
# src/main/resources/prompts/rag-template.st
Eres un asistente que responde preguntas basandose en la documentacion proporcionada.

Reglas:
- Responde SOLO con informacion del contexto
- Si no hay suficiente informacion, dilo claramente
- Cita las fuentes cuando sea posible
- Responde en espanol
- Se conciso pero completo

Contexto:
{contexto}

Pregunta del usuario:
{pregunta}
```

```kotlin
@Value("classpath:prompts/rag-template.st")
private lateinit var ragTemplate: Resource

fun responderConTemplate(pregunta: String, contexto: String): String {
    val template = PromptTemplate(ragTemplate)
    val prompt = template.create(mapOf(
        "contexto" to contexto,
        "pregunta" to pregunta
    ))

    return chatClient.prompt(prompt)
        .call()
        .content() ?: "Sin respuesta"
}
```

---

## 8. Pipeline RAG con Spring AI

### 8.1 QuestionAnswerAdvisor

Spring AI proporciona `QuestionAnswerAdvisor` que automatiza todo el pipeline RAG:

```kotlin
import org.springframework.ai.chat.client.advisor.QuestionAnswerAdvisor
import org.springframework.ai.vectorstore.SearchRequest

@Configuration
class RagConfig {

    @Bean
    fun chatClient(
        chatModel: ChatModel,
        vectorStore: VectorStore
    ): ChatClient {
        return ChatClient.builder(chatModel)
            .defaultSystem("Responde en espanol basandote en el contexto proporcionado.")
            .defaultAdvisors(
                QuestionAnswerAdvisor(
                    vectorStore,
                    SearchRequest.builder()
                        .topK(5)
                        .similarityThreshold(0.7)
                        .build()
                )
            )
            .build()
    }
}
```

Con esta configuracion, cada llamada al ChatClient automaticamente:
1. Busca documentos relevantes en el VectorStore
2. Los inyecta en el prompt
3. El LLM responde con el contexto

```kotlin
@RestController
@RequestMapping("/api/rag")
class RagController(
    private val chatClient: ChatClient
) {

    // Asi de simple: RAG automatico con QuestionAnswerAdvisor
    @GetMapping
    fun preguntar(@RequestParam q: String): String {
        return chatClient.prompt()
            .user(q)
            .call()
            .content() ?: "Sin respuesta"
    }
}
```

### 8.2 Pipeline manual con DocumentReader/Transformer/Writer

```kotlin
@Service
class RagPipelineService(
    private val vectorStore: VectorStore,
    private val chatClient: ChatClient
) {

    // Paso 1: Ingestar documentos
    fun ingestar(recursos: List<Resource>) {
        for (recurso in recursos) {
            // Leer
            val docs = TikaDocumentReader(recurso).get()

            // Transformar (chunking)
            val splitter = TokenTextSplitter.builder()
                .withChunkSize(500)
                .build()
            val chunks = splitter.apply(docs)

            // Escribir al vector store
            vectorStore.add(chunks)

            logger.info("Ingestado: ${recurso.filename} -> ${chunks.size} chunks")
        }
    }

    // Paso 2: Consultar
    fun consultar(pregunta: String): RagResponse {
        // Recuperar
        val documentos = vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(pregunta)
                .topK(5)
                .similarityThreshold(0.7)
                .build()
        )

        // Augmentar
        val contexto = documentos.joinToString("\n\n") { it.content }

        // Generar
        val respuesta = chatClient.prompt()
            .system("""
                Responde basandote unicamente en el siguiente contexto.
                Si no encuentras la respuesta en el contexto, dilo.
                
                Contexto:
                $contexto
            """.trimIndent())
            .user(pregunta)
            .call()
            .content() ?: "Sin respuesta"

        return RagResponse(
            respuesta = respuesta,
            documentosUsados = documentos.map {
                DocumentoRef(
                    fuente = it.metadata["fuente"]?.toString() ?: "desconocida",
                    fragmento = it.content.take(200)
                )
            }
        )
    }
}
```

### 8.3 RAG con filtrado por metadatos

```kotlin
// RAG que filtra documentos por categoria
fun consultarPorCategoria(
    pregunta: String,
    categoria: String
): RagResponse {
    val b = FilterExpressionBuilder()

    val documentos = vectorStore.similaritySearch(
        SearchRequest.builder()
            .query(pregunta)
            .topK(5)
            .filterExpression(b.eq("categoria", categoria).build())
            .build()
    )

    // ... resto del pipeline igual
}
```

---

## 9. Evaluacion y metricas

### 9.1 Las tres metricas clave

Evaluar un sistema RAG requiere medir tres aspectos:

```
1. Faithfulness (Fidelidad)
   ¿La respuesta es fiel al contexto recuperado?
   ¿Inventa informacion que no esta en los documentos?
   
2. Relevance (Relevancia del contexto)
   ¿Los documentos recuperados son relevantes para la pregunta?
   ¿Se recuperaron los documentos correctos?
   
3. Answer Correctness (Correccion de la respuesta)
   ¿La respuesta es facticamente correcta?
   ¿Responde realmente a la pregunta del usuario?
```

### 9.2 Evaluacion con LLM como juez

Puedes usar un LLM para evaluar las respuestas de tu sistema RAG:

```kotlin
@Service
class RagEvaluator(
    private val chatClient: ChatClient
) {

    data class Evaluacion(
        val fidelidad: Int,        // 1-5
        val relevancia: Int,       // 1-5
        val correccion: Int,       // 1-5
        val explicacion: String
    )

    fun evaluar(
        pregunta: String,
        respuesta: String,
        contexto: String,
        respuestaEsperada: String? = null
    ): Evaluacion {
        val prompt = """
            Evalua la siguiente respuesta de un sistema RAG.
            
            Pregunta: $pregunta
            Contexto proporcionado: $contexto
            Respuesta del sistema: $respuesta
            ${respuestaEsperada?.let { "Respuesta esperada: $it" } ?: ""}
            
            Evalua de 1 a 5:
            - Fidelidad: la respuesta se basa en el contexto sin inventar
            - Relevancia: el contexto es relevante para la pregunta
            - Correccion: la respuesta es correcta y util
            
            Responde en JSON: {"fidelidad": N, "relevancia": N, "correccion": N, "explicacion": "..."}
        """.trimIndent()

        return chatClient.prompt()
            .user(prompt)
            .call()
            .entity(Evaluacion::class.java)
    }
}
```

### 9.3 Dataset de evaluacion

```kotlin
// Crear un dataset de preguntas y respuestas esperadas para evaluar tu RAG
data class TestCase(
    val pregunta: String,
    val respuestaEsperada: String,
    val documentosEsperados: List<String>    // IDs de documentos que deberian recuperarse
)

fun evaluarPipeline(testCases: List<TestCase>): Map<String, Double> {
    var totalFidelidad = 0.0
    var totalRelevancia = 0.0
    var totalCorreccion = 0.0
    var retrievalHits = 0

    for (testCase in testCases) {
        val resultado = ragService.consultar(testCase.pregunta)

        // Evaluar la respuesta
        val evaluacion = evaluator.evaluar(
            pregunta = testCase.pregunta,
            respuesta = resultado.respuesta,
            contexto = resultado.documentosUsados.joinToString("\n") { it.fragmento },
            respuestaEsperada = testCase.respuestaEsperada
        )

        totalFidelidad += evaluacion.fidelidad
        totalRelevancia += evaluacion.relevancia
        totalCorreccion += evaluacion.correccion

        // Retrieval precision: documentos correctos recuperados
        val docsRecuperados = resultado.documentosUsados.map { it.fuente }
        if (testCase.documentosEsperados.any { it in docsRecuperados }) {
            retrievalHits++
        }
    }

    val n = testCases.size
    return mapOf(
        "fidelidad_promedio" to totalFidelidad / n,
        "relevancia_promedio" to totalRelevancia / n,
        "correccion_promedio" to totalCorreccion / n,
        "retrieval_precision" to retrievalHits.toDouble() / n
    )
}
```

---

## 10. Errores comunes y soluciones

### 10.1 Los chunks son demasiado pequenos o grandes

```
Sintoma: respuestas vagas o irrelevantes
Causa: chunks de 50 tokens pierden contexto; chunks de 2000 tokens son poco precisos

Solucion:
- Probar con 300-500 tokens como punto de partida
- Usar overlap de 20-30% del tamano del chunk
- Evaluar con metricas antes y despues de cambiar el tamano
```

### 10.2 Los documentos recuperados no son relevantes

```
Sintoma: el contexto no tiene relacion con la pregunta
Causas posibles:
  a) Modelo de embeddings inadecuado
  b) Falta de datos o datos de mala calidad
  c) Umbral de similitud demasiado bajo

Soluciones:
  a) Probar text-embedding-3-large en lugar de text-embedding-3-small
  b) Limpiar y preprocesar los documentos antes de la ingestion
  c) Subir el umbral de similitud a 0.75-0.8
  d) Usar busqueda hibrida (vector + texto)
```

### 10.3 El modelo alucina a pesar del contexto

```
Sintoma: la respuesta contiene informacion que no esta en los documentos
Causas:
  a) System prompt no es suficientemente restrictivo
  b) Temperatura demasiado alta
  c) El modelo ignora el contexto en preguntas muy generales

Soluciones:
  a) Reforzar el system prompt: "SOLO usa informacion del contexto"
  b) Bajar temperature a 0.0-0.3 para RAG
  c) Anadir instruccion: "Si la respuesta no esta en el contexto, di 'No lo se'"
  d) Validar la respuesta comparando con el contexto
```

### 10.4 El pipeline es demasiado lento

```
Sintoma: respuestas tardan mas de 5 segundos
Causas:
  a) Sin indice en la BD vectorial
  b) Demasiados documentos recuperados
  c) Re-ranking con LLM es costoso

Soluciones:
  a) Crear indice HNSW en pgvector
  b) Reducir topK de 10 a 3-5
  c) Cache de embeddings de consultas frecuentes
  d) Usar streaming para que el usuario vea la respuesta progresivamente
```

### 10.5 La ventana de contexto se llena

```
Sintoma: error de "context length exceeded" o respuestas truncadas
Causa: demasiados chunks inyectados + historial de conversacion largo

Soluciones:
  a) Reducir topK
  b) Truncar chunks largos antes de inyectar
  c) Resumir los chunks recuperados antes de inyectarlos
  d) Usar un modelo con ventana mayor (128K o 200K tokens)
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|---------|--------|-------------|
| RAG | Recuperar documentos relevantes antes de generar respuesta | Pipeline: ingestion + retrieval + generation |
| Pipeline de ingestion | Cargar, chunk, generar embeddings y almacenar | DocumentReader -> Splitter -> VectorStore.add() |
| Pipeline de consulta | Buscar, augmentar prompt y generar respuesta | VectorStore.search() -> prompt stuffing -> ChatClient |
| Chunking | Dividir documentos en fragmentos | TokenTextSplitter con 300-500 tokens y overlap |
| Chunking recursivo | Dividir respetando estructura del texto | Cortar por parrafos, luego frases, luego palabras |
| Retrieval top-K | Recuperar los K documentos mas similares | SearchRequest con topK y similarityThreshold |
| MMR | Balancear relevancia y diversidad en resultados | Evita resultados redundantes |
| Busqueda hibrida | Combinar busqueda vectorial y textual | Reciprocal Rank Fusion de ambos resultados |
| Prompt stuffing | Inyectar documentos recuperados en el prompt | Contexto en el system/user prompt antes de la pregunta |
| QuestionAnswerAdvisor | RAG automatico de Spring AI | Advisor que busca y augmenta automaticamente |
| Evaluacion RAG | Medir fidelidad, relevancia y correccion | LLM como juez + dataset de test |
| Fidelidad | La respuesta se basa en el contexto, no inventa | System prompt restrictivo + temperature baja |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **Carga de documentos** | Crear un servicio que cargue documentos desde texto plano, JSON y PDF, anadiendo metadatos a cada uno. Implementar un endpoint REST de ingestion | DocumentReader, TikaDocumentReader, metadatos, Resource |
| 02 | **Chunking y embeddings** | Implementar chunking de tamano fijo y recursivo. Comparar la calidad de busqueda con diferentes tamanos de chunk (200, 500, 1000 tokens) midiendo tiempos y relevancia | TokenTextSplitter, chunking recursivo, overlap, comparativa |
| 03 | **Pipeline de retrieval** | Construir un pipeline RAG completo con Spring AI: ingestion de 50+ documentos, busqueda por similitud con filtrado, y generacion de respuestas con QuestionAnswerAdvisor | VectorStore, SearchRequest, QuestionAnswerAdvisor, FilterExpression |
| 04 | **Evaluacion de RAG** | Crear un dataset de 20 preguntas con respuestas esperadas, evaluar el pipeline con metricas de fidelidad/relevancia/correccion, e iterar mejorando la configuracion | Evaluacion, LLM como juez, metricas, tuning de parametros |

---

## Recursos

- [Spring AI RAG Documentation](https://docs.spring.io/spring-ai/reference/api/rag.html)
- [RAG Paper original (Lewis et al. 2020)](https://arxiv.org/abs/2005.11401)
- [Chunking Strategies for RAG](https://www.pinecone.io/learn/chunking-strategies/)
- [RAG Evaluation Best Practices](https://www.ragas.io/)
- [Spring AI Advisors](https://docs.spring.io/spring-ai/reference/api/advisors.html)
- [Reciprocal Rank Fusion](https://plg.uwaterloo.ca/~gvcormac/cormacksigir09-rrf.pdf)

---

> **Consejo practico**: no optimices prematuramente. Empieza con el pipeline mas simple
> posible (chunking fijo de 500 tokens, top-5, sin re-ranking) y mide con tu dataset de
> evaluacion. Solo anade complejidad (chunking semantico, busqueda hibrida, re-ranking)
> cuando las metricas lo justifiquen. La mayoria de mejoras en RAG vienen de mejorar la
> calidad de los documentos de entrada, no del algoritmo de retrieval.
