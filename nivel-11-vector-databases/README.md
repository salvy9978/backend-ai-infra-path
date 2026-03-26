# Nivel 11 — Bases de datos vectoriales

> Los LLMs no pueden acceder a tus datos privados ni recordar mas alla de su ventana de contexto.
> Las bases de datos vectoriales resuelven esto: almacenan representaciones numericas (embeddings)
> de tus documentos y permiten buscar por significado, no por palabras clave. Son la pieza
> fundamental de RAG y la busqueda semantica. Ya conoces embeddings (nivel 09) y Spring AI
> (nivel 10). Ahora aprendes donde y como almacenarlos de forma eficiente.

---

## Contenido

1. [Que son las bases de datos vectoriales](#1-que-son-las-bases-de-datos-vectoriales)
2. [Embeddings y vectores](#2-embeddings-y-vectores)
3. [Busqueda por similitud](#3-busqueda-por-similitud)
4. [pgvector: vectores en PostgreSQL](#4-pgvector-vectores-en-postgresql)
5. [Bases de datos vectoriales dedicadas](#5-bases-de-datos-vectoriales-dedicadas)
6. [Estrategias de indexacion](#6-estrategias-de-indexacion)
7. [Spring AI VectorStore](#7-spring-ai-vectorstore)
8. [Filtrado por metadatos](#8-filtrado-por-metadatos)
9. [Rendimiento y tuning en produccion](#9-rendimiento-y-tuning-en-produccion)
10. [Resumen](#resumen)
11. [Ejercicios](#ejercicios)
12. [Recursos](#recursos)

---

## 1. Que son las bases de datos vectoriales

### 1.1 El problema

Las bases de datos tradicionales buscan por coincidencia exacta o patrones de texto:

```sql
-- Busqueda tradicional: palabras clave exactas
SELECT * FROM documentos WHERE contenido LIKE '%Spring Boot%';

-- Esto NO encuentra documentos que hablen de "framework de Spring para microservicios"
-- aunque el significado sea practicamente el mismo
```

Las bases de datos vectoriales permiten **busqueda semantica**: encontrar contenido
por significado, no por palabras exactas.

```
Consulta: "como arrancar una aplicacion Spring"

Busqueda tradicional (LIKE):
  - Encuentra: "como arrancar una aplicacion Spring" (match exacto)
  - NO encuentra: "iniciar un proyecto con Spring Boot" (misma idea, palabras distintas)

Busqueda vectorial (semantica):
  - Encuentra ambos, porque el SIGNIFICADO es similar
  - Tambien encuentra: "ejecutar una app Spring Boot desde la terminal"
```

### 1.2 Como funciona

El flujo es:

```
1. Tomas un documento de texto
2. Lo conviertes a un vector de numeros (embedding) usando un modelo de embeddings
3. Almacenas el vector junto con el texto original y metadatos
4. Para buscar: conviertes la consulta a vector y buscas los vectores mas cercanos
```

```
Documento: "Spring Boot simplifica la configuracion de Spring"
    |
    v [modelo de embeddings]
    |
Vector: [0.023, -0.156, 0.891, ..., 0.045]  (1536 dimensiones)
    |
    v [almacenar]
    |
Base de datos vectorial: vector + texto original + metadatos
```

### 1.3 Casos de uso

| Caso de uso | Descripcion |
|-------------|-------------|
| RAG | Dar contexto relevante a un LLM con datos privados |
| Busqueda semantica | Buscar documentos por significado |
| Recomendaciones | Recomendar contenido similar |
| Deteccion de duplicados | Encontrar textos semanticamente identicos |
| Clasificacion | Clasificar texto por proximidad a categorias predefinidas |
| Preguntas frecuentes | Encontrar la FAQ mas relevante para una pregunta |

---

## 2. Embeddings y vectores

### 2.1 Repaso rapido

Un embedding es un vector de numeros reales que representa el significado semantico
de un fragmento de texto. Textos con significado similar tienen vectores cercanos
en el espacio vectorial.

```kotlin
// Generar embeddings con Spring AI
@Service
class EmbeddingService(
    private val embeddingModel: EmbeddingModel  // autoconfigured por Spring AI
) {

    fun generarEmbedding(texto: String): List<Double> {
        val response = embeddingModel.embed(texto)
        return response.toList()               // vector de 1536 dimensiones (OpenAI)
    }

    fun generarEmbeddings(textos: List<String>): List<List<Double>> {
        val response = embeddingModel.embed(textos)
        return response.map { it.toList() }
    }
}
```

### 2.2 Dimensiones y modelos

| Modelo de embeddings | Dimensiones | Proveedor |
|---------------------|-------------|-----------|
| text-embedding-3-small | 1536 | OpenAI |
| text-embedding-3-large | 3072 | OpenAI |
| voyage-3 | 1024 | Anthropic (via Voyage AI) |
| nomic-embed-text | 768 | Ollama (local) |
| all-MiniLM-L6-v2 | 384 | Sentence Transformers (local) |

Mas dimensiones generalmente capturan mas matices semanticos, pero consumen mas
almacenamiento y la busqueda es mas lenta.

### 2.3 Almacenamiento por vector

Cada vector ocupa espacio proporcional a sus dimensiones:

```
1 vector de 1536 dimensiones (float32) = 1536 * 4 bytes = 6,144 bytes (~6 KB)

Para 1 millon de documentos:
  - Vectores: 1M * 6 KB = ~6 GB solo de vectores
  - Mas indices, metadatos y texto original

Para 10 millones: ~60 GB solo de vectores
```

---

## 3. Busqueda por similitud

### 3.1 Metricas de distancia

Hay tres metricas principales para medir la similitud entre vectores:

```
1. Similitud del coseno (Cosine Similarity)
   - Mide el angulo entre dos vectores
   - Rango: -1 a 1 (normalmente 0 a 1 para embeddings)
   - 1 = identicos, 0 = ortogonales (sin relacion)
   - La mas usada para embeddings de texto

2. Distancia euclidiana (L2 Distance)
   - Mide la distancia "recta" entre dos puntos
   - Rango: 0 a infinito
   - 0 = identicos, mayor = mas distantes
   - Sensible a la magnitud de los vectores

3. Producto punto (Dot Product)
   - Mide la proyeccion de un vector sobre otro
   - Rango: -infinito a infinito
   - Mas rapido de calcular
   - Requiere vectores normalizados para comparar magnitudes
```

### 3.2 Implementacion en Kotlin

```kotlin
import kotlin.math.sqrt

object VectorSimilarity {

    // Similitud del coseno: la metrica mas usada para embeddings
    fun cosine(a: FloatArray, b: FloatArray): Double {
        require(a.size == b.size) { "Vectores deben tener misma dimension" }

        var dot = 0.0      // producto punto
        var normA = 0.0    // norma de A
        var normB = 0.0    // norma de B

        for (i in a.indices) {
            dot += a[i] * b[i]
            normA += a[i] * a[i]
            normB += b[i] * b[i]
        }

        val denominador = sqrt(normA) * sqrt(normB)
        return if (denominador == 0.0) 0.0 else dot / denominador
    }

    // Distancia euclidiana
    fun euclidean(a: FloatArray, b: FloatArray): Double {
        require(a.size == b.size)
        var sum = 0.0
        for (i in a.indices) {
            val diff = a[i] - b[i]
            sum += diff * diff
        }
        return sqrt(sum)
    }

    // Producto punto
    fun dotProduct(a: FloatArray, b: FloatArray): Double {
        require(a.size == b.size)
        var sum = 0.0
        for (i in a.indices) {
            sum += a[i] * b[i]
        }
        return sum
    }
}
```

### 3.3 Cual usar

| Metrica | Cuando usarla | Operador pgvector |
|---------|--------------|-------------------|
| Cosine | Embeddings de texto (la mas comun) | `<=>` |
| Euclidean | Cuando la magnitud importa | `<->` |
| Dot Product | Vectores normalizados, maxima velocidad | `<#>` |

Para embeddings de texto de OpenAI o similares, usa **cosine**. Los vectores ya vienen
normalizados, asi que cosine y dot product dan resultados equivalentes.

---

## 4. pgvector: vectores en PostgreSQL

### 4.1 Que es pgvector

pgvector es una extension de PostgreSQL que anade soporte nativo para vectores.
La ventaja principal: usas la misma base de datos que ya tienes para datos relacionales
y vectores. No necesitas infraestructura adicional.

### 4.2 Instalacion

```sql
-- Habilitar la extension (requiere que este instalada en el servidor)
CREATE EXTENSION IF NOT EXISTS vector;
```

```yaml
# Docker Compose con pgvector
services:
  postgres:
    image: pgvector/pgvector:pg16      # imagen oficial con pgvector preinstalado
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: usuario
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

### 4.3 Crear tablas con columnas vectoriales

```sql
-- Tabla de documentos con embedding
CREATE TABLE documentos (
    id          BIGSERIAL PRIMARY KEY,
    contenido   TEXT NOT NULL,                     -- texto original
    embedding   vector(1536) NOT NULL,             -- vector de 1536 dimensiones
    metadata    JSONB DEFAULT '{}',                -- metadatos flexibles
    creado_en   TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Tabla de productos con embedding para busqueda semantica
CREATE TABLE productos_embeddings (
    id              BIGSERIAL PRIMARY KEY,
    producto_id     BIGINT REFERENCES productos(id),
    descripcion     TEXT NOT NULL,
    embedding       vector(1536) NOT NULL,
    categoria       VARCHAR(100),
    activo          BOOLEAN DEFAULT true
);
```

### 4.4 Insertar vectores

```sql
-- Insertar un documento con su embedding
INSERT INTO documentos (contenido, embedding, metadata) VALUES (
    'Spring Boot simplifica la configuracion de aplicaciones Spring',
    '[0.023, -0.156, 0.891, ...]',       -- vector como string
    '{"fuente": "documentacion", "version": "3.2"}'
);
```

```kotlin
// Desde Kotlin con JdbcTemplate
@Repository
class DocumentoVectorRepository(
    private val jdbcTemplate: JdbcTemplate
) {

    fun insertar(contenido: String, embedding: FloatArray, metadata: Map<String, Any>) {
        val vectorStr = embedding.joinToString(",", "[", "]")
        val metadataJson = ObjectMapper().writeValueAsString(metadata)

        jdbcTemplate.update(
            """INSERT INTO documentos (contenido, embedding, metadata) 
               VALUES (?, ?::vector, ?::jsonb)""",
            contenido,
            vectorStr,
            metadataJson
        )
    }
}
```

### 4.5 Busqueda por similitud

```sql
-- Buscar los 5 documentos mas similares a un vector dado
-- Operador <=> : distancia coseno (menor = mas similar)
SELECT id, contenido, 
       1 - (embedding <=> '[0.023, -0.156, ...]') AS similitud
FROM documentos
ORDER BY embedding <=> '[0.023, -0.156, ...]'
LIMIT 5;

-- Operador <-> : distancia euclidiana
SELECT id, contenido,
       embedding <-> '[0.023, -0.156, ...]' AS distancia
FROM documentos
ORDER BY embedding <-> '[0.023, -0.156, ...]'
LIMIT 5;

-- Operador <#> : producto punto negativo (inner product)
SELECT id, contenido,
       (embedding <#> '[0.023, -0.156, ...]') * -1 AS similitud
FROM documentos
ORDER BY embedding <#> '[0.023, -0.156, ...]'
LIMIT 5;
```

```kotlin
// Busqueda semantica desde Kotlin
@Repository
class BusquedaSemanticaRepository(
    private val jdbcTemplate: JdbcTemplate
) {

    fun buscarSimilares(
        queryEmbedding: FloatArray,
        limite: Int = 5,
        similitudMinima: Double = 0.7
    ): List<DocumentoSimilar> {
        val vectorStr = queryEmbedding.joinToString(",", "[", "]")

        return jdbcTemplate.query(
            """SELECT id, contenido, metadata,
                      1 - (embedding <=> ?::vector) AS similitud
               FROM documentos
               WHERE 1 - (embedding <=> ?::vector) >= ?
               ORDER BY embedding <=> ?::vector
               LIMIT ?""",
            arrayOf(vectorStr, vectorStr, similitudMinima, vectorStr, limite)
        ) { rs, _ ->
            DocumentoSimilar(
                id = rs.getLong("id"),
                contenido = rs.getString("contenido"),
                similitud = rs.getDouble("similitud"),
                metadata = rs.getString("metadata")
            )
        }
    }
}

data class DocumentoSimilar(
    val id: Long,
    val contenido: String,
    val similitud: Double,
    val metadata: String
)
```

### 4.6 Indices en pgvector

Sin indice, pgvector hace una busqueda exhaustiva (escanea todos los vectores).
Para tablas grandes necesitas indices:

```sql
-- Indice HNSW (recomendado): rapido en busqueda, buen recall
CREATE INDEX idx_documentos_embedding_hnsw ON documentos 
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Indice IVFFlat: mas rapido de construir, recall inferior
CREATE INDEX idx_documentos_embedding_ivfflat ON documentos
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);
```

Operadores de distancia para el indice:

| Operador | Nombre | Ops class |
|----------|--------|-----------|
| `<=>` | Cosine | `vector_cosine_ops` |
| `<->` | L2 (Euclidean) | `vector_l2_ops` |
| `<#>` | Inner product | `vector_ip_ops` |

---

## 5. Bases de datos vectoriales dedicadas

### 5.1 Cuando PostgreSQL no es suficiente

pgvector es excelente para empezar y funciona bien hasta millones de vectores.
Pero si necesitas:

- Decenas de millones de vectores o mas
- Busqueda distribuida en multiples nodos
- Filtrado avanzado en tiempo real
- Actualizaciones frecuentes a gran escala

Entonces una base de datos vectorial dedicada puede ser mejor opcion.

### 5.2 Comparativa

| Caracteristica | pgvector | Qdrant | Pinecone | Weaviate |
|---------------|----------|--------|----------|----------|
| Tipo | Extension PostgreSQL | DB dedicada | Servicio gestionado | DB dedicada |
| Hosting | Self-hosted / Cloud | Self-hosted / Cloud | Solo Cloud | Self-hosted / Cloud |
| Escalabilidad | Hasta ~10M vectores | Hasta billones | Hasta billones | Hasta billones |
| Filtrado | SQL completo | Filtros nativos | Filtros basicos | GraphQL + filtros |
| Coste | Gratis (con tu PostgreSQL) | Gratis (self-hosted) | Pago por uso | Gratis (self-hosted) |
| Complejidad | Minima (ya tienes PostgreSQL) | Media | Baja (managed) | Media |
| Spring AI | Si (PgVectorStore) | Si (QdrantVectorStore) | Si (PineconeVectorStore) | Si (WeaviateVectorStore) |

### 5.3 Qdrant

Base de datos vectorial open-source escrita en Rust. Muy rapida y con buen soporte
de filtrado por metadatos:

```yaml
# Docker Compose para Qdrant
services:
  qdrant:
    image: qdrant/qdrant:latest
    ports:
      - "6333:6333"     # API REST
      - "6334:6334"     # gRPC
    volumes:
      - qdrant_data:/qdrant/storage

volumes:
  qdrant_data:
```

```kotlin
// Spring AI con Qdrant
// Dependencia: spring-ai-qdrant-store-spring-boot-starter
@Configuration
class QdrantConfig {
    // Spring AI autoconfigura QdrantVectorStore con las propiedades
}
```

```yaml
spring:
  ai:
    vectorstore:
      qdrant:
        host: localhost
        port: 6334                  # puerto gRPC
        collection-name: documentos
```

### 5.4 Pinecone

Servicio completamente gestionado. No necesitas infraestructura:

```yaml
spring:
  ai:
    vectorstore:
      pinecone:
        api-key: ${PINECONE_API_KEY}
        environment: us-east-1
        index-name: mi-indice
        namespace: produccion
```

### 5.5 Cuando usar cual

```
¿Tienes menos de 5 millones de documentos?
  -> pgvector (ya tienes PostgreSQL, sin infraestructura extra)

¿Necesitas escalabilidad extrema y control total?
  -> Qdrant (self-hosted, open-source, muy rapido)

¿No quieres gestionar infraestructura?
  -> Pinecone (managed, facil de usar, pago por uso)

¿Necesitas busqueda hibrida (vectorial + texto + filtros)?
  -> Weaviate o Qdrant (mejores capacidades de filtrado)
```

---

## 6. Estrategias de indexacion

### 6.1 Busqueda exhaustiva (Flat / brute force)

Sin indice, se compara el vector de consulta con todos los vectores de la tabla.

```
Ventajas:
  - 100% precision (recall perfecto)
  - Sin coste de construccion de indice

Desventajas:
  - O(n) por consulta: lento con muchos vectores
  - Inaceptable para mas de ~100K vectores
```

### 6.2 IVF (Inverted File Index)

Divide los vectores en clusters. Al buscar, solo se miran los clusters mas cercanos:

```
Algoritmo:
1. Divide los vectores en K clusters (usando k-means)
2. Cada vector se asigna a su cluster mas cercano
3. Al buscar: primero encuentra los clusters mas cercanos al query
4. Luego busca solo dentro de esos clusters

Parametros:
  - lists: numero de clusters (mas = busqueda mas rapida, menor recall)
  - probes: cuantos clusters explorar al buscar (mas = mayor recall, mas lento)
```

```sql
-- Crear indice IVF en pgvector
CREATE INDEX idx_docs_ivf ON documentos
USING ivfflat (embedding vector_cosine_ops)
WITH (lists = 100);              -- 100 clusters

-- Configurar probes para la sesion (cuantos clusters explorar)
SET ivfflat.probes = 10;         -- explorar 10 de los 100 clusters
```

### 6.3 HNSW (Hierarchical Navigable Small World)

El algoritmo mas usado actualmente. Construye un grafo navegable en capas:

```
Algoritmo:
1. Construye un grafo donde cada vector es un nodo
2. Los nodos cercanos estan conectados por aristas
3. Varias capas: las superiores conectan nodos distantes (atajos)
4. Al buscar: empieza por arriba, va bajando y acercandose

Analogia:
  - Capa superior: mapa del pais (saltas entre ciudades)
  - Capa media: mapa de la ciudad (vas por barrios)
  - Capa inferior: mapa del barrio (caminas por calles)
```

```sql
-- Crear indice HNSW en pgvector (recomendado)
CREATE INDEX idx_docs_hnsw ON documentos
USING hnsw (embedding vector_cosine_ops)
WITH (
    m = 16,                      -- conexiones por nodo (16 es buen default)
    ef_construction = 64         -- precision durante construccion (mayor = mejor indice, mas lento de construir)
);

-- Configurar precision de busqueda
SET hnsw.ef_search = 40;         -- mayor = mejor recall, mas lento (default: 40)
```

### 6.4 Comparativa de indices

| Indice | Velocidad busqueda | Recall | Coste de construccion | Memoria |
|--------|-------------------|--------|----------------------|---------|
| Flat (sin indice) | Lento (O(n)) | 100% | Ninguno | Solo vectores |
| IVFFlat | Rapido | 85-95% | Rapido | Bajo |
| HNSW | Muy rapido | 95-99% | Lento | Alto (grafo en memoria) |

Recomendacion general:
- **Menos de 100K vectores**: flat (sin indice) es aceptable
- **100K a 1M vectores**: IVFFlat o HNSW
- **Mas de 1M vectores**: HNSW (preferido) con parametros ajustados

---

## 7. Spring AI VectorStore

### 7.1 La abstraccion VectorStore

Spring AI proporciona una interfaz comun `VectorStore` que funciona con cualquier
base de datos vectorial:

```kotlin
// La interfaz (simplificada)
interface VectorStore {
    fun add(documents: List<Document>)
    fun delete(idList: List<String>)
    fun similaritySearch(request: SearchRequest): List<Document>
}
```

### 7.2 Configuracion con pgvector

```xml
<!-- Dependencia Maven -->
<dependency>
    <groupId>org.springframework.ai</groupId>
    <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  ai:
    vectorstore:
      pgvector:
        dimensions: 1536                # dimension de los embeddings
        distance-type: cosine_distance  # metrica de distancia
        index-type: hnsw                # tipo de indice
        schema-name: public             # schema de PostgreSQL
        table-name: vector_store        # nombre de la tabla
        initialize-schema: true         # crear tabla automaticamente
  datasource:
    url: jdbc:postgresql://localhost:5432/miapp
    username: usuario
    password: password
```

### 7.3 Almacenar documentos

```kotlin
import org.springframework.ai.document.Document
import org.springframework.ai.vectorstore.VectorStore

@Service
class DocumentoService(
    private val vectorStore: VectorStore
) {

    // Almacenar documentos (Spring AI genera los embeddings automaticamente)
    fun almacenar(documentos: List<String>) {
        val docs = documentos.mapIndexed { index, contenido ->
            Document(
                contenido,                              // texto del documento
                mapOf(                                  // metadatos
                    "fuente" to "manual",
                    "indice" to index,
                    "fecha" to LocalDate.now().toString()
                )
            )
        }
        vectorStore.add(docs)  // genera embeddings y almacena automaticamente
    }

    // Almacenar un documento individual
    fun almacenarUno(contenido: String, metadatos: Map<String, Any>): String {
        val doc = Document(contenido, metadatos)
        vectorStore.add(listOf(doc))
        return doc.id    // Spring AI genera un ID unico
    }
}
```

### 7.4 Busqueda por similitud

```kotlin
import org.springframework.ai.vectorstore.SearchRequest

@Service
class BusquedaService(
    private val vectorStore: VectorStore
) {

    // Busqueda simple: los K documentos mas similares
    fun buscar(consulta: String, topK: Int = 5): List<Document> {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)                      // texto de busqueda
                .topK(topK)                           // numero de resultados
                .build()
        )
    }

    // Busqueda con umbral de similitud
    fun buscarConUmbral(
        consulta: String,
        topK: Int = 5,
        similitudMinima: Double = 0.7
    ): List<Document> {
        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)
                .topK(topK)
                .similarityThreshold(similitudMinima)  // solo resultados con similitud >= 0.7
                .build()
        )
    }
}
```

### 7.5 Endpoint REST de busqueda

```kotlin
@RestController
@RequestMapping("/api/busqueda")
class BusquedaController(
    private val busquedaService: BusquedaService
) {

    @GetMapping
    fun buscar(
        @RequestParam q: String,
        @RequestParam(defaultValue = "5") topK: Int,
        @RequestParam(defaultValue = "0.7") umbral: Double
    ): List<ResultadoBusqueda> {
        return busquedaService.buscarConUmbral(q, topK, umbral)
            .map { doc ->
                ResultadoBusqueda(
                    contenido = doc.content,
                    metadatos = doc.metadata,
                    // La similitud esta en los metadatos del documento
                    similitud = doc.metadata["distance"] as? Double ?: 0.0
                )
            }
    }
}

data class ResultadoBusqueda(
    val contenido: String,
    val metadatos: Map<String, Any>,
    val similitud: Double
)
```

---

## 8. Filtrado por metadatos

### 8.1 Por que filtrar

La busqueda vectorial pura devuelve los documentos mas similares semanticamente,
pero a veces necesitas restringir los resultados:

- Solo documentos de una categoria especifica
- Solo documentos creados despues de cierta fecha
- Solo documentos de un usuario particular
- Solo documentos activos

### 8.2 FilterExpression de Spring AI

```kotlin
import org.springframework.ai.vectorstore.filter.FilterExpressionBuilder

@Service
class BusquedaFiltradaService(
    private val vectorStore: VectorStore
) {

    fun buscarPorCategoria(consulta: String, categoria: String): List<Document> {
        val filterBuilder = FilterExpressionBuilder()

        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)
                .topK(5)
                .filterExpression(
                    filterBuilder.eq("categoria", categoria).build()
                )
                .build()
        )
    }

    fun buscarConFiltrosComplejos(
        consulta: String,
        categoria: String,
        fechaDesde: String,
        activo: Boolean
    ): List<Document> {
        val b = FilterExpressionBuilder()

        val filtro = b.and(
            b.eq("categoria", categoria),
            b.and(
                b.gte("fecha", fechaDesde),
                b.eq("activo", activo)
            )
        ).build()

        return vectorStore.similaritySearch(
            SearchRequest.builder()
                .query(consulta)
                .topK(10)
                .filterExpression(filtro)
                .build()
        )
    }
}
```

### 8.3 Operadores disponibles

| Operador | Metodo | Ejemplo |
|----------|--------|---------|
| Igual | `eq(campo, valor)` | `b.eq("tipo", "tutorial")` |
| Distinto | `ne(campo, valor)` | `b.ne("estado", "borrador")` |
| Mayor que | `gt(campo, valor)` | `b.gt("score", 0.8)` |
| Mayor o igual | `gte(campo, valor)` | `b.gte("fecha", "2024-01-01")` |
| Menor que | `lt(campo, valor)` | `b.lt("precio", 100)` |
| Menor o igual | `lte(campo, valor)` | `b.lte("longitud", 500)` |
| Contiene | `in(campo, valores)` | `b.in("tag", "kotlin", "spring")` |
| No contiene | `nin(campo, valores)` | `b.nin("tag", "deprecated")` |
| Y logico | `and(expr1, expr2)` | `b.and(expr1, expr2)` |
| O logico | `or(expr1, expr2)` | `b.or(expr1, expr2)` |
| Negacion | `not(expr)` | `b.not(expr)` |

### 8.4 Filtrado con pgvector nativo

Si usas pgvector directamente (sin Spring AI VectorStore), el filtrado es SQL estandar:

```sql
-- Busqueda vectorial con filtro por categoria y fecha
SELECT id, contenido, 
       1 - (embedding <=> '[0.023, ...]'::vector) AS similitud
FROM documentos
WHERE metadata->>'categoria' = 'tutorial'
  AND (metadata->>'fecha')::date >= '2024-01-01'
  AND (metadata->>'activo')::boolean = true
ORDER BY embedding <=> '[0.023, ...]'::vector
LIMIT 5;
```

```sql
-- Indice parcial para mejorar rendimiento de filtros frecuentes
CREATE INDEX idx_docs_activos ON documentos (id)
WHERE (metadata->>'activo')::boolean = true;

-- Indice en campo JSONB frecuente
CREATE INDEX idx_docs_categoria ON documentos ((metadata->>'categoria'));
```

---

## 9. Rendimiento y tuning en produccion

### 9.1 Dimensionamiento

```
Factores que afectan el rendimiento:
1. Numero de vectores: mas vectores = busqueda mas lenta
2. Dimensiones: mas dimensiones = mas memoria y calculo
3. Tipo de indice: HNSW > IVFFlat > Flat
4. Parametros del indice: ef_search/probes afectan recall vs velocidad
5. Hardware: RAM suficiente para mantener el indice en memoria
```

### 9.2 Reglas practicas para pgvector

```sql
-- Para tablas pequenas (< 100K vectores): sin indice es aceptable
-- El scan secuencial puede ser mas rapido que un indice para tablas pequenas

-- Para tablas medianas (100K - 1M vectores): HNSW con defaults
CREATE INDEX idx_embedding ON documentos
USING hnsw (embedding vector_cosine_ops)
WITH (m = 16, ef_construction = 64);

-- Para tablas grandes (> 1M vectores): HNSW con parametros ajustados
CREATE INDEX idx_embedding ON documentos
USING hnsw (embedding vector_cosine_ops)
WITH (m = 24, ef_construction = 200);   -- mas conexiones, mejor indice

-- Ajustar precision de busqueda
SET hnsw.ef_search = 100;               -- mayor recall (default: 40)
```

### 9.3 Monitoreo

```kotlin
@Component
class VectorStoreMetrics(
    private val jdbcTemplate: JdbcTemplate,
    private val meterRegistry: MeterRegistry
) {

    @Scheduled(fixedRate = 60000)  // cada minuto
    fun registrarMetricas() {
        // Numero total de vectores
        val totalVectores = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM vector_store", Long::class.java
        ) ?: 0

        meterRegistry.gauge("vectorstore.total_documents", totalVectores)

        // Tamano de la tabla en disco
        val tamanoMB = jdbcTemplate.queryForObject(
            "SELECT pg_total_relation_size('vector_store') / (1024*1024)",
            Long::class.java
        ) ?: 0

        meterRegistry.gauge("vectorstore.size_mb", tamanoMB)
    }
}
```

### 9.4 Optimizaciones comunes

```kotlin
// 1. Batch inserts: insertar documentos en lotes
fun almacenarEnBatch(documentos: List<Document>, batchSize: Int = 100) {
    documentos.chunked(batchSize).forEach { batch ->
        vectorStore.add(batch)
    }
}

// 2. Cache de embeddings de consultas frecuentes
@Cacheable("query-embeddings", key = "#consulta")
fun buscarConCache(consulta: String): List<Document> {
    return vectorStore.similaritySearch(
        SearchRequest.builder().query(consulta).topK(5).build()
    )
}

// 3. Reducir dimensiones si la precision es suficiente
// text-embedding-3-small soporta reduccion de dimensiones:
// 1536 -> 512 ahorra ~66% de almacenamiento con ~5% perdida de precision
```

```yaml
# Configuracion de pool de conexiones para operaciones vectoriales pesadas
spring:
  datasource:
    hikari:
      maximum-pool-size: 15        # mas conexiones para busquedas concurrentes
      connection-timeout: 10000    # timeout mas corto para detectar problemas
```

---

## Resumen

| Concepto | Que es | Como usarlo |
|---------|--------|-------------|
| Base de datos vectorial | BD optimizada para almacenar y buscar vectores | pgvector, Qdrant, Pinecone, Weaviate |
| Embedding | Vector numerico que representa significado de texto | Generado por modelo de embeddings (OpenAI, etc.) |
| Similitud coseno | Metrica que mide angulo entre vectores | Operador `<=>` en pgvector, la mas usada |
| Distancia euclidiana | Metrica que mide distancia recta entre vectores | Operador `<->` en pgvector |
| pgvector | Extension de PostgreSQL para vectores | `CREATE EXTENSION vector`, columnas `vector(1536)` |
| HNSW | Indice basado en grafo navegable, rapido y preciso | `CREATE INDEX USING hnsw` (recomendado) |
| IVFFlat | Indice basado en clusters, rapido de construir | `CREATE INDEX USING ivfflat` (alternativa) |
| VectorStore | Abstraccion de Spring AI para BDs vectoriales | `vectorStore.add(docs)`, `vectorStore.similaritySearch(req)` |
| SearchRequest | Peticion de busqueda con topK, umbral y filtros | `SearchRequest.builder().query(...).topK(5).build()` |
| FilterExpression | Filtrado por metadatos en busquedas vectoriales | `FilterExpressionBuilder.eq("campo", "valor")` |
| Batch insert | Insertar documentos en lotes para eficiencia | `documentos.chunked(100).forEach { vectorStore.add(it) }` |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 01 | **pgvector basico** | Configurar PostgreSQL con pgvector en Docker, crear tabla con columna vector, insertar embeddings generados con Spring AI y consultar por similitud con SQL directo | pgvector, Docker, `CREATE EXTENSION vector`, operadores `<=>` `<->` |
| 02 | **Busqueda semantica** | Construir un motor de busqueda semantica sobre una coleccion de 100 articulos tecnicos. Comparar resultados de busqueda por palabras clave (LIKE) vs busqueda vectorial | Embeddings, similitud coseno, comparacion LIKE vs vector |
| 03 | **Spring AI VectorStore** | Usar `PgVectorStore` de Spring AI para almacenar documentos con metadatos y hacer busquedas filtradas por categoria, fecha y similitud minima | VectorStore, SearchRequest, FilterExpression, PgVectorStore |
| 04 | **Estrategias de indexacion** | Cargar 50K+ vectores, crear indices HNSW e IVFFlat, medir tiempos de busqueda y recall con diferentes parametros. Documentar las diferencias | HNSW, IVFFlat, ef_search, probes, benchmark, recall |

---

## Recursos

- [pgvector GitHub](https://github.com/pgvector/pgvector)
- [pgvector: Indexing](https://github.com/pgvector/pgvector#indexing)
- [Spring AI VectorStore Documentation](https://docs.spring.io/spring-ai/reference/api/vectordbs.html)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Pinecone Documentation](https://docs.pinecone.io/)
- [HNSW Paper (original)](https://arxiv.org/abs/1603.09320)
- [Choosing a Vector Database](https://benchmark.vectorview.ai/)

---

> **Consejo practico**: empieza siempre con pgvector. Si ya tienes PostgreSQL en tu stack,
> no necesitas infraestructura adicional. Solo migra a una BD vectorial dedicada cuando
> pgvector se quede corto (decenas de millones de vectores o requisitos de latencia extremos).
