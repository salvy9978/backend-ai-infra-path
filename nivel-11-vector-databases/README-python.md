# Nivel 11 — Bases de datos vectoriales (Track Python)

> Los LLMs no pueden acceder a tus datos privados ni recordar mas alla de su ventana de contexto.
> Las bases de datos vectoriales resuelven esto: almacenan representaciones numericas (embeddings)
> de tus documentos y permiten buscar por significado, no por palabras clave. Este track cubre
> las mismas ideas que el track Kotlin/Spring AI, pero con el ecosistema Python: psycopg2/asyncpg
> con pgvector, Qdrant client, Pinecone SDK, numpy/scipy para similitud manual, y LangChain
> como capa de abstraccion sobre todas las bases vectoriales.

---

## Contenido

1. [pgvector desde Python](#1-pgvector-desde-python)
2. [Similitud manual con numpy y scipy](#2-similitud-manual-con-numpy-y-scipy)
3. [Qdrant Python client](#3-qdrant-python-client)
4. [Pinecone Python SDK](#4-pinecone-python-sdk)
5. [LangChain VectorStore integrations](#5-langchain-vectorstore-integrations)
6. [Filtrado por metadatos en Python](#6-filtrado-por-metadatos-en-python)
7. [Generacion de embeddings en batch con OpenAI](#7-generacion-de-embeddings-en-batch-con-openai)
8. [Comparativa de rendimiento desde Python](#8-comparativa-de-rendimiento-desde-python)
9. [Resumen](#resumen)
10. [Ejercicios](#ejercicios)
11. [Recursos](#recursos)

---

## 1. pgvector desde Python

### 1.1 Instalacion de dependencias

```bash
pip install psycopg2-binary asyncpg pgvector openai numpy
```

pgvector expone los mismos operadores SQL independientemente del lenguaje cliente.
Desde Python tienes dos drivers principales: psycopg2 (sincrono) y asyncpg (asincrono).

### 1.2 Conexion y setup con psycopg2

```python
import psycopg2
from pgvector.psycopg2 import register_vector

# Conectar a PostgreSQL con pgvector
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    dbname="miapp",
    user="usuario",
    password="password"
)

# Registrar el tipo vector para que psycopg2 lo entienda
register_vector(conn)

cur = conn.cursor()

# Habilitar la extension pgvector
cur.execute("CREATE EXTENSION IF NOT EXISTS vector")
conn.commit()
```

### 1.3 Crear tabla y insertar vectores

```python
import numpy as np

# Crear tabla con columna vectorial
cur.execute("""
    CREATE TABLE IF NOT EXISTS documentos (
        id          BIGSERIAL PRIMARY KEY,
        contenido   TEXT NOT NULL,
        embedding   vector(1536) NOT NULL,       -- 1536 dimensiones (OpenAI)
        metadata    JSONB DEFAULT '{}',
        creado_en   TIMESTAMP NOT NULL DEFAULT NOW()
    )
""")
conn.commit()

# Insertar un documento con su embedding
def insertar_documento(contenido: str, embedding: list[float], metadata: dict):
    """Inserta un documento con su vector de embedding."""
    import json
    cur.execute(
        """INSERT INTO documentos (contenido, embedding, metadata)
           VALUES (%s, %s, %s)""",
        (contenido, np.array(embedding), json.dumps(metadata))
    )
    conn.commit()

# Ejemplo de uso
insertar_documento(
    contenido="FastAPI simplifica la creacion de APIs REST en Python",
    embedding=[0.023, -0.156, 0.891] + [0.0] * 1533,  # vector de 1536 dims
    metadata={"fuente": "documentacion", "version": "0.100"}
)
```

### 1.4 Busqueda por similitud con psycopg2

```python
def buscar_similares(query_embedding: list[float], limite: int = 5, umbral: float = 0.7):
    """Busca documentos similares usando distancia coseno."""
    query_vec = np.array(query_embedding)

    cur.execute(
        """SELECT id, contenido, metadata,
                  1 - (embedding <=> %s) AS similitud
           FROM documentos
           WHERE 1 - (embedding <=> %s) >= %s
           ORDER BY embedding <=> %s
           LIMIT %s""",
        (query_vec, query_vec, umbral, query_vec, limite)
    )

    resultados = []
    for row in cur.fetchall():
        resultados.append({
            "id": row[0],
            "contenido": row[1],
            "metadata": row[2],
            "similitud": row[3]
        })
    return resultados

# Ejemplo de uso
resultados = buscar_similares(
    query_embedding=[0.023, -0.156, 0.891] + [0.0] * 1533,
    limite=5,
    umbral=0.7
)
for r in resultados:
    print(f"  [{r['similitud']:.3f}] {r['contenido'][:80]}...")
```

### 1.5 asyncpg para aplicaciones asincronas

```python
import asyncio
import asyncpg
from pgvector.asyncpg import register_vector as async_register_vector

async def main():
    # Conexion asincrona
    conn = await asyncpg.connect(
        host="localhost",
        port=5432,
        database="miapp",
        user="usuario",
        password="password"
    )

    # Registrar tipo vector para asyncpg
    await async_register_vector(conn)

    # Busqueda asincrona por similitud
    query_vec = [0.023, -0.156, 0.891] + [0.0] * 1533
    rows = await conn.fetch(
        """SELECT id, contenido,
                  1 - (embedding <=> $1) AS similitud
           FROM documentos
           ORDER BY embedding <=> $1
           LIMIT $2""",
        query_vec,          # asyncpg acepta listas directamente
        5
    )

    for row in rows:
        print(f"  [{row['similitud']:.3f}] {row['contenido'][:80]}")

    await conn.close()

asyncio.run(main())
```

### 1.6 Indices en pgvector desde Python

```python
# Crear indice HNSW (recomendado para la mayoria de casos)
cur.execute("""
    CREATE INDEX IF NOT EXISTS idx_docs_embedding_hnsw
    ON documentos USING hnsw (embedding vector_cosine_ops)
    WITH (m = 16, ef_construction = 64)
""")
conn.commit()

# Ajustar precision de busqueda para la sesion
cur.execute("SET hnsw.ef_search = 100")  # mayor = mejor recall, mas lento

# Crear indice IVFFlat (alternativa, mas rapido de construir)
cur.execute("""
    CREATE INDEX IF NOT EXISTS idx_docs_embedding_ivf
    ON documentos USING ivfflat (embedding vector_cosine_ops)
    WITH (lists = 100)
""")
conn.commit()
```

### 1.7 Insercion en batch

```python
from pgvector.psycopg2 import register_vector
import psycopg2.extras

def insertar_batch(documentos: list[dict]):
    """Inserta multiples documentos en un solo roundtrip a la BD."""
    import json
    values = [
        (
            doc["contenido"],
            np.array(doc["embedding"]),
            json.dumps(doc.get("metadata", {}))
        )
        for doc in documentos
    ]
    psycopg2.extras.execute_values(
        cur,
        """INSERT INTO documentos (contenido, embedding, metadata)
           VALUES %s""",
        values,
        template="(%s, %s, %s::jsonb)",
        page_size=100        # tamano del batch SQL
    )
    conn.commit()

# Ejemplo: insertar 1000 documentos de golpe
docs_batch = [
    {
        "contenido": f"Documento de prueba numero {i}",
        "embedding": np.random.rand(1536).tolist(),
        "metadata": {"indice": i, "fuente": "generado"}
    }
    for i in range(1000)
]
insertar_batch(docs_batch)
```

---

## 2. Similitud manual con numpy y scipy

### 2.1 Por que calcularla manualmente

A veces necesitas calcular similitud fuera de la base de datos:
- Pre-filtrar resultados antes de guardarlos
- Evaluar la calidad de embeddings
- Tests unitarios sin BD
- Re-ranking en memoria

### 2.2 Cosine similarity con numpy

```python
import numpy as np

def cosine_similarity(a: np.ndarray, b: np.ndarray) -> float:
    """Similitud del coseno entre dos vectores."""
    dot = np.dot(a, b)                        # producto punto
    norm_a = np.linalg.norm(a)                # norma euclidiana de a
    norm_b = np.linalg.norm(b)                # norma euclidiana de b
    if norm_a == 0 or norm_b == 0:
        return 0.0
    return float(dot / (norm_a * norm_b))     # coseno del angulo

# Ejemplo
vec_a = np.array([0.1, 0.3, 0.5, 0.7])
vec_b = np.array([0.2, 0.4, 0.6, 0.8])
print(f"Similitud coseno: {cosine_similarity(vec_a, vec_b):.4f}")
# Salida: Similitud coseno: 0.9923
```

### 2.3 Distancia euclidiana con numpy

```python
def euclidean_distance(a: np.ndarray, b: np.ndarray) -> float:
    """Distancia euclidiana (L2) entre dos vectores."""
    return float(np.linalg.norm(a - b))

# Ejemplo
dist = euclidean_distance(vec_a, vec_b)
print(f"Distancia euclidiana: {dist:.4f}")
```

### 2.4 Batch similarity con scipy

```python
from scipy.spatial.distance import cdist, cosine

# Calcular similitud entre un query y muchos documentos a la vez
def buscar_similares_en_memoria(
    query: np.ndarray,
    documentos: np.ndarray,       # matriz (N, D) con N vectores de D dimensiones
    top_k: int = 5
) -> list[tuple[int, float]]:
    """Busca los top_k documentos mas similares al query en memoria."""
    # cdist calcula distancias entre todos los pares
    # metric='cosine' devuelve 1 - cosine_similarity
    distancias = cdist(
        query.reshape(1, -1),     # reshape a (1, D) para cdist
        documentos,
        metric="cosine"
    )[0]                          # [0] porque solo hay un query

    # Convertir distancia coseno a similitud
    similitudes = 1 - distancias

    # Obtener indices ordenados por similitud descendente
    indices_ordenados = np.argsort(-similitudes)[:top_k]

    return [(int(idx), float(similitudes[idx])) for idx in indices_ordenados]

# Ejemplo: buscar entre 10000 documentos
np.random.seed(42)
corpus = np.random.rand(10000, 1536).astype(np.float32)  # 10K docs de 1536 dims
query = np.random.rand(1536).astype(np.float32)

resultados = buscar_similares_en_memoria(query, corpus, top_k=5)
for idx, sim in resultados:
    print(f"  Doc {idx}: similitud = {sim:.4f}")
```

### 2.5 Comparativa de metricas

```python
from scipy.spatial.distance import cosine, euclidean
import numpy as np

def comparar_metricas(a: np.ndarray, b: np.ndarray):
    """Compara las tres metricas principales para un par de vectores."""
    sim_coseno = 1 - cosine(a, b)              # scipy devuelve distancia
    dist_euclidiana = euclidean(a, b)
    producto_punto = float(np.dot(a, b))

    print(f"Similitud coseno:     {sim_coseno:.4f}")
    print(f"Distancia euclidiana: {dist_euclidiana:.4f}")
    print(f"Producto punto:       {producto_punto:.4f}")

# Vectores normalizados (como los de OpenAI)
a = np.array([0.1, 0.3, 0.5, 0.7])
a = a / np.linalg.norm(a)                      # normalizar
b = np.array([0.2, 0.4, 0.6, 0.8])
b = b / np.linalg.norm(b)

comparar_metricas(a, b)
# Para vectores normalizados, coseno y producto punto dan resultado equivalente
```

---

## 3. Qdrant Python client

### 3.1 Instalacion y conexion

```bash
pip install qdrant-client
```

```python
from qdrant_client import QdrantClient
from qdrant_client.models import (
    Distance, VectorParams, PointStruct,
    Filter, FieldCondition, MatchValue
)

# Conectar a Qdrant local (Docker)
client = QdrantClient(host="localhost", port=6333)

# O conectar a Qdrant Cloud
# client = QdrantClient(url="https://xxxx.qdrant.io", api_key="tu-api-key")
```

### 3.2 Crear coleccion

```python
# Crear coleccion con configuracion de vectores
client.create_collection(
    collection_name="documentos",
    vectors_config=VectorParams(
        size=1536,                              # dimensiones del vector
        distance=Distance.COSINE                # metrica de distancia
    )
)
```

### 3.3 Insertar puntos (vectores con payload)

```python
import uuid

def insertar_en_qdrant(documentos: list[dict]):
    """Inserta documentos como puntos en Qdrant."""
    points = []
    for doc in documentos:
        points.append(PointStruct(
            id=str(uuid.uuid4()),               # ID unico
            vector=doc["embedding"],             # vector de 1536 dims
            payload={                            # metadatos (payload)
                "contenido": doc["contenido"],
                "fuente": doc.get("fuente", "desconocida"),
                "categoria": doc.get("categoria", "general"),
                "fecha": doc.get("fecha", "")
            }
        ))

    # Insertar en batch (Qdrant lo maneja eficientemente)
    client.upsert(
        collection_name="documentos",
        points=points
    )

# Ejemplo
insertar_en_qdrant([
    {
        "contenido": "FastAPI es un framework web moderno para Python",
        "embedding": [0.1] * 1536,
        "fuente": "docs",
        "categoria": "python"
    },
    {
        "contenido": "Django es un framework web full-stack para Python",
        "embedding": [0.2] * 1536,
        "fuente": "docs",
        "categoria": "python"
    }
])
```

### 3.4 Busqueda por similitud

```python
def buscar_en_qdrant(
    query_embedding: list[float],
    limite: int = 5,
    umbral: float = 0.7
) -> list[dict]:
    """Busca documentos similares en Qdrant."""
    resultados = client.search(
        collection_name="documentos",
        query_vector=query_embedding,
        limit=limite,
        score_threshold=umbral               # solo resultados con score >= umbral
    )

    return [
        {
            "id": str(r.id),
            "contenido": r.payload.get("contenido", ""),
            "fuente": r.payload.get("fuente", ""),
            "similitud": r.score
        }
        for r in resultados
    ]

# Ejemplo
resultados = buscar_en_qdrant([0.15] * 1536, limite=3)
for r in resultados:
    print(f"  [{r['similitud']:.3f}] {r['contenido'][:80]}")
```

### 3.5 Busqueda con filtros

```python
def buscar_con_filtros(
    query_embedding: list[float],
    categoria: str,
    limite: int = 5
) -> list[dict]:
    """Busqueda vectorial filtrada por metadatos en Qdrant."""
    resultados = client.search(
        collection_name="documentos",
        query_vector=query_embedding,
        query_filter=Filter(
            must=[                               # condiciones AND
                FieldCondition(
                    key="categoria",
                    match=MatchValue(value=categoria)
                )
            ]
        ),
        limit=limite
    )
    return [{"contenido": r.payload["contenido"], "score": r.score} for r in resultados]

# Buscar solo documentos de categoria "python"
resultados = buscar_con_filtros([0.15] * 1536, categoria="python")
```

### 3.6 Filtros avanzados en Qdrant

```python
from qdrant_client.models import (
    Filter, FieldCondition, MatchValue, Range,
    IsEmptyCondition, PayloadField
)

# Filtro compuesto: categoria = "python" Y fecha >= "2024-01-01"
filtro_complejo = Filter(
    must=[
        FieldCondition(key="categoria", match=MatchValue(value="python")),
        FieldCondition(key="fecha", range=Range(gte="2024-01-01"))
    ],
    must_not=[
        FieldCondition(key="fuente", match=MatchValue(value="borrador"))
    ]
)

resultados = client.search(
    collection_name="documentos",
    query_vector=[0.15] * 1536,
    query_filter=filtro_complejo,
    limit=10
)
```

---

## 4. Pinecone Python SDK

### 4.1 Instalacion y conexion

```bash
pip install pinecone-client
```

```python
from pinecone import Pinecone, ServerlessSpec

# Inicializar cliente
pc = Pinecone(api_key="tu-pinecone-api-key")

# Crear indice (solo la primera vez)
pc.create_index(
    name="mi-indice",
    dimension=1536,
    metric="cosine",                         # cosine, euclidean, dotproduct
    spec=ServerlessSpec(
        cloud="aws",
        region="us-east-1"
    )
)

# Conectar al indice
index = pc.Index("mi-indice")
```

### 4.2 Insertar vectores

```python
import uuid

def insertar_en_pinecone(documentos: list[dict], namespace: str = ""):
    """Inserta documentos en Pinecone."""
    vectors = []
    for doc in documentos:
        vectors.append({
            "id": str(uuid.uuid4()),
            "values": doc["embedding"],
            "metadata": {                    # metadatos para filtrado
                "contenido": doc["contenido"][:1000],  # Pinecone limita metadatos
                "fuente": doc.get("fuente", ""),
                "categoria": doc.get("categoria", "")
            }
        })

    # Insertar en batches de 100 (limite de Pinecone)
    for i in range(0, len(vectors), 100):
        batch = vectors[i:i+100]
        index.upsert(vectors=batch, namespace=namespace)
```

### 4.3 Busqueda

```python
def buscar_en_pinecone(
    query_embedding: list[float],
    top_k: int = 5,
    namespace: str = "",
    filtro: dict = None
) -> list[dict]:
    """Busca vectores similares en Pinecone."""
    resultados = index.query(
        vector=query_embedding,
        top_k=top_k,
        namespace=namespace,
        filter=filtro,                       # filtro por metadatos (opcional)
        include_metadata=True                # incluir metadatos en respuesta
    )

    return [
        {
            "id": match["id"],
            "contenido": match["metadata"].get("contenido", ""),
            "similitud": match["score"]
        }
        for match in resultados["matches"]
    ]

# Busqueda simple
resultados = buscar_en_pinecone([0.1] * 1536, top_k=3)

# Busqueda con filtro
resultados = buscar_en_pinecone(
    [0.1] * 1536,
    top_k=3,
    filtro={"categoria": {"$eq": "python"}}
)
```

### 4.4 Filtros de Pinecone

```python
# Operadores de filtrado en Pinecone
filtros_ejemplo = {
    # Igualdad
    "categoria": {"$eq": "python"},

    # Rango numerico
    "score": {"$gte": 0.8},

    # En una lista
    "tag": {"$in": ["tutorial", "guia"]},

    # Combinacion con AND
    "$and": [
        {"categoria": {"$eq": "python"}},
        {"fecha": {"$gte": "2024-01-01"}}
    ]
}
```

---

## 5. LangChain VectorStore integrations

### 5.1 Instalacion

```bash
pip install langchain langchain-openai langchain-community
pip install langchain-postgres     # para PGVector
pip install langchain-qdrant       # para Qdrant
pip install faiss-cpu              # para FAISS (local, en memoria)
```

### 5.2 PGVector con LangChain

```python
from langchain_postgres import PGVector
from langchain_openai import OpenAIEmbeddings

# Modelo de embeddings
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",
    openai_api_key="tu-api-key"
)

# Conexion a pgvector via LangChain
CONNECTION_STRING = "postgresql+psycopg://usuario:password@localhost:5432/miapp"

vectorstore = PGVector(
    embeddings=embeddings,
    collection_name="documentos_langchain",
    connection=CONNECTION_STRING,
    use_jsonb=True                            # usar JSONB para metadatos
)

# Insertar documentos (LangChain genera embeddings automaticamente)
from langchain_core.documents import Document

docs = [
    Document(
        page_content="FastAPI es un framework web moderno para Python",
        metadata={"fuente": "docs", "categoria": "python"}
    ),
    Document(
        page_content="Django es un framework web full-stack",
        metadata={"fuente": "docs", "categoria": "python"}
    ),
    Document(
        page_content="PostgreSQL es un RDBMS open-source muy potente",
        metadata={"fuente": "docs", "categoria": "databases"}
    )
]

vectorstore.add_documents(docs)              # genera embeddings y almacena
```

### 5.3 Busqueda con LangChain PGVector

```python
# Busqueda simple por similitud
resultados = vectorstore.similarity_search(
    query="framework web para Python",
    k=3                                       # top-K resultados
)
for doc in resultados:
    print(f"  {doc.page_content[:80]}")
    print(f"  Metadata: {doc.metadata}")

# Busqueda con scores
resultados_con_score = vectorstore.similarity_search_with_score(
    query="framework web para Python",
    k=3
)
for doc, score in resultados_con_score:
    print(f"  [{score:.3f}] {doc.page_content[:80]}")

# Busqueda con filtro por metadatos
resultados = vectorstore.similarity_search(
    query="framework web",
    k=3,
    filter={"categoria": "python"}            # solo docs de categoria python
)
```

### 5.4 Qdrant con LangChain

```python
from langchain_qdrant import QdrantVectorStore
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Conectar a Qdrant via LangChain
vectorstore = QdrantVectorStore.from_documents(
    documents=docs,                           # documentos a insertar
    embedding=embeddings,
    url="http://localhost:6333",
    collection_name="documentos_langchain"
)

# Busqueda (misma API que PGVector)
resultados = vectorstore.similarity_search("framework web", k=3)
```

### 5.5 FAISS con LangChain (local, en memoria)

```python
from langchain_community.vectorstores import FAISS
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Crear vectorstore en memoria con FAISS
vectorstore = FAISS.from_documents(docs, embeddings)

# Busqueda
resultados = vectorstore.similarity_search("framework web", k=3)

# Guardar a disco para persistencia
vectorstore.save_local("faiss_index")

# Cargar desde disco
vectorstore_cargado = FAISS.load_local(
    "faiss_index",
    embeddings,
    allow_dangerous_deserialization=True
)
```

### 5.6 API unificada de LangChain

La ventaja principal de LangChain: la misma API para cualquier vector store:

```python
# Esta funcion funciona con PGVector, Qdrant, FAISS, Pinecone, etc.
def buscar_documentos(vectorstore, consulta: str, k: int = 5) -> list:
    """Busqueda generica que funciona con cualquier VectorStore de LangChain."""
    return vectorstore.similarity_search_with_score(
        query=consulta,
        k=k
    )

# Puedes cambiar de backend sin modificar la logica de busqueda
resultados = buscar_documentos(vectorstore, "framework web para APIs", k=3)
```

---

## 6. Filtrado por metadatos en Python

### 6.1 Filtrado en pgvector nativo

```python
def buscar_con_filtro_pgvector(
    query_embedding: list[float],
    categoria: str,
    fecha_desde: str,
    limite: int = 5
):
    """Busqueda vectorial con filtros SQL en pgvector."""
    cur.execute(
        """SELECT id, contenido, metadata,
                  1 - (embedding <=> %s) AS similitud
           FROM documentos
           WHERE metadata->>'categoria' = %s
             AND (metadata->>'fecha')::date >= %s::date
             AND (metadata->>'activo')::boolean = true
           ORDER BY embedding <=> %s
           LIMIT %s""",
        (np.array(query_embedding), categoria, fecha_desde,
         np.array(query_embedding), limite)
    )
    return cur.fetchall()
```

### 6.2 Filtrado en LangChain

```python
# PGVector: filtro con diccionario
resultados = vectorstore.similarity_search(
    query="framework web",
    k=5,
    filter={"categoria": "python"}           # igualdad simple
)

# Filtros mas complejos dependen del backend
# PGVector soporta operadores de comparacion:
resultados = vectorstore.similarity_search(
    query="framework web",
    k=5,
    filter={
        "categoria": {"$eq": "python"},
        "fecha": {"$gte": "2024-01-01"}
    }
)
```

### 6.3 Filtrado en Qdrant nativo

```python
from qdrant_client.models import Filter, FieldCondition, MatchValue, Range

# Filtro compuesto
filtro = Filter(
    must=[
        FieldCondition(key="categoria", match=MatchValue(value="python")),
        FieldCondition(key="fecha", range=Range(gte="2024-01-01"))
    ],
    must_not=[
        FieldCondition(key="fuente", match=MatchValue(value="borrador"))
    ]
)

resultados = client.search(
    collection_name="documentos",
    query_vector=query_embedding,
    query_filter=filtro,
    limit=5
)
```

### 6.4 Indices parciales para filtros frecuentes

```python
# Crear indices en pgvector para mejorar filtros
cur.execute("""
    CREATE INDEX IF NOT EXISTS idx_docs_categoria
    ON documentos ((metadata->>'categoria'))
""")

cur.execute("""
    CREATE INDEX IF NOT EXISTS idx_docs_activos
    ON documentos (id)
    WHERE (metadata->>'activo')::boolean = true
""")
conn.commit()
```

---

## 7. Generacion de embeddings en batch con OpenAI

### 7.1 Cliente de OpenAI

```python
from openai import OpenAI

client_openai = OpenAI(api_key="tu-api-key")

def generar_embedding(texto: str) -> list[float]:
    """Genera embedding para un texto individual."""
    response = client_openai.embeddings.create(
        model="text-embedding-3-small",
        input=texto
    )
    return response.data[0].embedding          # vector de 1536 floats
```

### 7.2 Batch de embeddings

```python
def generar_embeddings_batch(textos: list[str], batch_size: int = 100) -> list[list[float]]:
    """Genera embeddings en batches para evitar limites de la API."""
    todos_los_embeddings = []

    for i in range(0, len(textos), batch_size):
        batch = textos[i:i + batch_size]
        response = client_openai.embeddings.create(
            model="text-embedding-3-small",
            input=batch                        # la API acepta listas
        )
        # Mantener el orden correcto
        batch_embeddings = [item.embedding for item in response.data]
        todos_los_embeddings.extend(batch_embeddings)

        print(f"  Procesados {min(i + batch_size, len(textos))}/{len(textos)}")

    return todos_los_embeddings

# Ejemplo: generar embeddings para 500 documentos
textos = [f"Documento de ejemplo numero {i}" for i in range(500)]
embeddings = generar_embeddings_batch(textos, batch_size=100)
print(f"Generados {len(embeddings)} embeddings de {len(embeddings[0])} dimensiones")
```

### 7.3 Pipeline completo: generar e insertar

```python
def pipeline_ingestion(documentos: list[dict]):
    """Pipeline completo: genera embeddings y los inserta en pgvector."""
    # 1. Extraer textos
    textos = [doc["contenido"] for doc in documentos]

    # 2. Generar embeddings en batch
    print(f"Generando embeddings para {len(textos)} documentos...")
    embeddings = generar_embeddings_batch(textos, batch_size=100)

    # 3. Asociar embeddings con documentos
    for doc, emb in zip(documentos, embeddings):
        doc["embedding"] = emb

    # 4. Insertar en pgvector en batch
    print("Insertando en pgvector...")
    insertar_batch(documentos)

    print(f"Pipeline completado: {len(documentos)} documentos ingestados")

# Ejemplo
docs = [
    {"contenido": "FastAPI es un framework web moderno", "metadata": {"fuente": "docs"}},
    {"contenido": "Django usa el patron MTV", "metadata": {"fuente": "docs"}},
    # ... mas documentos
]
pipeline_ingestion(docs)
```

### 7.4 Manejo de errores y reintentos

```python
import time
from openai import RateLimitError, APIError

def generar_embeddings_con_reintentos(
    textos: list[str],
    max_reintentos: int = 3,
    batch_size: int = 100
) -> list[list[float]]:
    """Genera embeddings con manejo de errores y reintentos."""
    todos = []

    for i in range(0, len(textos), batch_size):
        batch = textos[i:i + batch_size]

        for intento in range(max_reintentos):
            try:
                response = client_openai.embeddings.create(
                    model="text-embedding-3-small",
                    input=batch
                )
                batch_embeddings = [item.embedding for item in response.data]
                todos.extend(batch_embeddings)
                break                           # exito, salir del loop de reintentos

            except RateLimitError:
                espera = 2 ** intento           # backoff exponencial: 1, 2, 4 segundos
                print(f"  Rate limit alcanzado, esperando {espera}s...")
                time.sleep(espera)

            except APIError as e:
                print(f"  Error de API: {e}")
                if intento == max_reintentos - 1:
                    raise                       # re-lanzar si agotamos reintentos

    return todos
```

---

## 8. Comparativa de rendimiento desde Python

### 8.1 Script de benchmark

```python
import time
import numpy as np

def benchmark_busqueda(nombre: str, funcion_busqueda, query: list[float], n_iter: int = 100):
    """Mide el tiempo promedio de busqueda."""
    tiempos = []

    for _ in range(n_iter):
        inicio = time.perf_counter()
        funcion_busqueda(query)
        fin = time.perf_counter()
        tiempos.append((fin - inicio) * 1000)  # milisegundos

    tiempos_np = np.array(tiempos)
    print(f"\n{nombre}:")
    print(f"  Media:   {tiempos_np.mean():.2f} ms")
    print(f"  Mediana: {np.median(tiempos_np):.2f} ms")
    print(f"  P95:     {np.percentile(tiempos_np, 95):.2f} ms")
    print(f"  P99:     {np.percentile(tiempos_np, 99):.2f} ms")

# Benchmark pgvector
def buscar_pgvector(query):
    cur.execute(
        "SELECT id FROM documentos ORDER BY embedding <=> %s LIMIT 5",
        (np.array(query),)
    )
    return cur.fetchall()

# Benchmark Qdrant
def buscar_qdrant(query):
    return client.search(collection_name="documentos", query_vector=query, limit=5)

# Ejecutar benchmarks
query = np.random.rand(1536).tolist()

benchmark_busqueda("pgvector (HNSW)", buscar_pgvector, query)
benchmark_busqueda("Qdrant", buscar_qdrant, query)
```

### 8.2 Comparativa de recall

```python
def medir_recall(
    busqueda_aproximada,
    busqueda_exacta,
    queries: list[list[float]],
    k: int = 10
) -> float:
    """Mide el recall comparando busqueda aproximada con busqueda exacta (brute force)."""
    total_recall = 0.0

    for query in queries:
        # Resultados exactos (ground truth)
        exactos = set(r[0] for r in busqueda_exacta(query, k))

        # Resultados aproximados (con indice)
        aproximados = set(r[0] for r in busqueda_aproximada(query, k))

        # Recall: que fraccion de los exactos se encontraron
        if len(exactos) > 0:
            recall = len(exactos & aproximados) / len(exactos)
            total_recall += recall

    return total_recall / len(queries)

# Ejemplo de uso
# recall = medir_recall(buscar_hnsw, buscar_brute_force, queries_test)
# print(f"Recall HNSW: {recall:.2%}")
```

### 8.3 Tabla comparativa tipica

```
Resultados tipicos con 1M vectores de 1536 dimensiones:

| Motor            | Latencia media | P99    | Recall | Notas                      |
|------------------|---------------|--------|--------|----------------------------|
| pgvector (flat)  | 450 ms        | 800 ms | 100%   | Scan secuencial completo   |
| pgvector (HNSW)  | 5 ms          | 12 ms  | 97%    | m=16, ef_search=40         |
| pgvector (IVF)   | 8 ms          | 20 ms  | 92%    | lists=100, probes=10       |
| Qdrant           | 3 ms          | 8 ms   | 98%    | Optimizado para vectores   |
| Pinecone         | 15 ms         | 50 ms  | 99%    | Incluye latencia de red    |
| FAISS (memoria)  | 1 ms          | 3 ms   | 99%    | Solo en memoria, no persiste |
```

---

## Resumen

| Concepto | Que es | Como usarlo en Python |
|---------|--------|----------------------|
| psycopg2 + pgvector | Driver sincrono para PostgreSQL con soporte vectorial | `register_vector(conn)`, operadores `<=>` `<->` |
| asyncpg + pgvector | Driver asincrono para PostgreSQL con soporte vectorial | `await async_register_vector(conn)`, ideal para FastAPI |
| numpy cosine | Calculo manual de similitud del coseno | `np.dot(a,b) / (norm(a) * norm(b))` |
| scipy cdist | Calculo batch de distancias entre vectores | `cdist(query, corpus, metric="cosine")` |
| qdrant_client | Cliente Python para Qdrant | `QdrantClient(host, port)`, `client.search()` |
| Pinecone SDK | Cliente Python para Pinecone (managed) | `Pinecone(api_key=...)`, `index.query()` |
| LangChain PGVector | Abstraccion de LangChain sobre pgvector | `PGVector(embeddings, connection)`, `similarity_search()` |
| LangChain Qdrant | Abstraccion de LangChain sobre Qdrant | `QdrantVectorStore.from_documents()` |
| LangChain FAISS | Vectorstore en memoria con FAISS | `FAISS.from_documents()`, `save_local()` / `load_local()` |
| Batch embeddings | Generar embeddings en lotes con OpenAI | `client.embeddings.create(input=lista_textos)` |
| Filtrado metadata | Filtrar busqueda vectorial por atributos | SQL WHERE en pgvector, Filter en Qdrant, dict en LangChain |
| Benchmark | Medir latencia y recall de diferentes backends | `time.perf_counter()`, comparar con brute force |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 05 | **pgvector desde Python** | Configurar pgvector con Docker, conectar con psycopg2, crear tabla con columna vector, insertar 1000 documentos con embeddings generados por OpenAI, y buscar por similitud. Comparar resultados con y sin indice HNSW | psycopg2, pgvector, register_vector, HNSW, batch insert, benchmark |
| 06 | **Qdrant desde Python** | Levantar Qdrant con Docker, usar qdrant_client para crear coleccion, insertar documentos con payload, buscar con filtros por metadatos. Comparar latencia y facilidad con pgvector | qdrant_client, PointStruct, Filter, FieldCondition, payload |
| 07 | **Similitud manual con numpy** | Implementar cosine, euclidiana y dot product con numpy. Buscar los 10 documentos mas similares en un corpus de 50K vectores en memoria con scipy cdist. Comparar metricas | numpy, scipy.spatial.distance, cdist, cosine_similarity, benchmark |
| 08 | **Busqueda hibrida con LangChain** | Usar LangChain PGVector para ingestar 200 documentos, implementar busqueda hibrida combinando similarity_search con full-text search de PostgreSQL. Evaluar precision vs busqueda vectorial pura | LangChain, PGVector, full-text search, Reciprocal Rank Fusion |

---

## Recursos

- [pgvector Python (pgvector-python)](https://github.com/pgvector/pgvector-python)
- [asyncpg Documentation](https://magicstack.github.io/asyncpg/)
- [Qdrant Python Client](https://python-client.qdrant.tech/)
- [Pinecone Python SDK](https://docs.pinecone.io/reference/python-sdk)
- [LangChain VectorStores](https://python.langchain.com/docs/integrations/vectorstores/)
- [FAISS (Facebook AI Similarity Search)](https://faiss.ai/)
- [OpenAI Embeddings API](https://platform.openai.com/docs/guides/embeddings)
- [numpy Documentation](https://numpy.org/doc/)
- [scipy.spatial.distance](https://docs.scipy.org/doc/scipy/reference/spatial.distance.html)

---

> **Consejo practico**: en Python, LangChain te da la mayor flexibilidad para cambiar
> de backend vectorial sin reescribir codigo. Empieza con FAISS para prototipos locales,
> migra a pgvector cuando necesites persistencia, y considera Qdrant o Pinecone solo
> cuando superes los limites de pgvector.
