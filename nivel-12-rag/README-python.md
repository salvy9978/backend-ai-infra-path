# Nivel 12 — RAG (Retrieval-Augmented Generation) (Track Python)

> Los LLMs son potentes pero tienen dos limitaciones criticas: no conocen tus datos privados
> y su conocimiento tiene una fecha de corte. RAG resuelve ambos problemas. Este track cubre
> el ecosistema Python para RAG: LangChain como framework principal, LlamaIndex como alternativa,
> document loaders especializados, estrategias avanzadas de chunking, multiples estrategias de
> retrieval, evaluacion con RAGAS, y servir un pipeline RAG con FastAPI.

---

## Contenido

1. [LangChain RAG pipeline](#1-langchain-rag-pipeline)
2. [LlamaIndex para RAG](#2-llamaindex-para-rag)
3. [Document loaders especializados](#3-document-loaders-especializados)
4. [Chunking avanzado](#4-chunking-avanzado)
5. [Estrategias de retrieval](#5-estrategias-de-retrieval)
6. [Evaluacion con RAGAS](#6-evaluacion-con-ragas)
7. [FastAPI + LangChain para servir RAG](#7-fastapi--langchain-para-servir-rag)
8. [Comparativa: LangChain vs LlamaIndex](#8-comparativa-langchain-vs-llamaindex)
9. [Resumen](#resumen)
10. [Ejercicios](#ejercicios)
11. [Recursos](#recursos)

---

## 1. LangChain RAG pipeline

### 1.1 Instalacion

```bash
pip install langchain langchain-openai langchain-community langchain-postgres
pip install pypdf unstructured faiss-cpu
```

### 1.2 Los componentes del pipeline

LangChain organiza RAG en componentes que se encadenan:

```
DocumentLoader   -> Carga documentos de distintas fuentes
TextSplitter     -> Divide documentos en chunks
Embeddings       -> Genera vectores de los chunks
VectorStore      -> Almacena y busca vectores
Retriever        -> Recupera chunks relevantes
Chain            -> Conecta retriever con LLM para generar respuestas
```

### 1.3 Pipeline minimo de RAG

```python
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import RetrievalQA

# 1. Cargar documento
loader = TextLoader("documentacion.txt", encoding="utf-8")
documentos = loader.load()

# 2. Dividir en chunks
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,              # caracteres por chunk
    chunk_overlap=100,           # solapamiento entre chunks
    separators=["\n\n", "\n", ". ", " ", ""]  # prioridad de separadores
)
chunks = splitter.split_documents(documentos)
print(f"Documento dividido en {len(chunks)} chunks")

# 3. Crear vectorstore con embeddings
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")
vectorstore = FAISS.from_documents(chunks, embeddings)

# 4. Crear retriever
retriever = vectorstore.as_retriever(
    search_type="similarity",     # tipo de busqueda
    search_kwargs={"k": 5}        # top-K resultados
)

# 5. Crear chain de RAG
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.0)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    chain_type="stuff",           # inyectar todos los docs en el prompt
    retriever=retriever,
    return_source_documents=True   # devolver los documentos usados
)

# 6. Consultar
resultado = qa_chain.invoke({"query": "Cual es la politica de devolucion?"})
print(f"Respuesta: {resultado['result']}")
print(f"Fuentes: {len(resultado['source_documents'])} documentos")
```

### 1.4 Pipeline con LCEL (LangChain Expression Language)

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough

# Prompt template para RAG
prompt = ChatPromptTemplate.from_template("""
Responde la pregunta basandote UNICAMENTE en el siguiente contexto.
Si no encuentras la respuesta en el contexto, di "No tengo suficiente informacion."

Contexto:
{context}

Pregunta: {question}

Respuesta:
""")

# Funcion para formatear documentos recuperados
def formatear_docs(docs):
    return "\n\n---\n\n".join(
        f"[Fuente: {doc.metadata.get('source', 'desconocida')}]\n{doc.page_content}"
        for doc in docs
    )

# Pipeline LCEL (mas moderno y flexible que RetrievalQA)
rag_chain = (
    {
        "context": retriever | formatear_docs,    # recuperar y formatear
        "question": RunnablePassthrough()          # pasar la pregunta tal cual
    }
    | prompt                                       # construir el prompt
    | llm                                          # generar respuesta
    | StrOutputParser()                            # extraer texto
)

# Consultar
respuesta = rag_chain.invoke("Cual es la politica de devolucion?")
print(respuesta)
```

### 1.5 Ingestion completa con metadatos

```python
from langchain_postgres import PGVector
from langchain_core.documents import Document

# Conexion a pgvector
CONNECTION = "postgresql+psycopg://usuario:password@localhost:5432/miapp"
embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

vectorstore = PGVector(
    embeddings=embeddings,
    collection_name="rag_documentos",
    connection=CONNECTION,
    use_jsonb=True
)

def ingestar_documentos(docs: list[Document]):
    """Ingesta documentos con chunking y metadatos."""
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=500,
        chunk_overlap=100
    )
    chunks = splitter.split_documents(docs)

    # Enriquecer metadatos
    for i, chunk in enumerate(chunks):
        chunk.metadata["chunk_index"] = i
        chunk.metadata["total_chunks"] = len(chunks)
        chunk.metadata["longitud"] = len(chunk.page_content)

    # Almacenar (genera embeddings automaticamente)
    vectorstore.add_documents(chunks)
    print(f"Ingestados {len(chunks)} chunks")

# Uso
docs = [
    Document(
        page_content="FastAPI es un framework web moderno para Python...",
        metadata={"fuente": "docs", "categoria": "python", "fecha": "2024-06-01"}
    )
]
ingestar_documentos(docs)
```

---

## 2. LlamaIndex para RAG

### 2.1 Instalacion

```bash
pip install llama-index llama-index-llms-openai llama-index-embeddings-openai
pip install llama-index-vector-stores-postgres
```

### 2.2 Pipeline minimo con LlamaIndex

```python
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex, Settings
from llama_index.llms.openai import OpenAI
from llama_index.embeddings.openai import OpenAIEmbedding

# Configurar modelos globalmente
Settings.llm = OpenAI(model="gpt-4o-mini", temperature=0.0)
Settings.embed_model = OpenAIEmbedding(model_name="text-embedding-3-small")

# 1. Cargar documentos desde un directorio
documents = SimpleDirectoryReader("./datos/").load_data()
print(f"Cargados {len(documents)} documentos")

# 2. Crear indice vectorial (chunk + embedding + almacenar, todo automatico)
index = VectorStoreIndex.from_documents(documents)

# 3. Crear query engine
query_engine = index.as_query_engine(
    similarity_top_k=5               # top-K documentos a recuperar
)

# 4. Consultar
response = query_engine.query("Cual es la politica de devolucion?")
print(f"Respuesta: {response}")
print(f"Fuentes: {[n.node.metadata for n in response.source_nodes]}")
```

### 2.3 LlamaIndex con pgvector

```python
from llama_index.vector_stores.postgres import PGVectorStore
from llama_index.core import StorageContext, VectorStoreIndex
import psycopg2

# Configurar pgvector como backend
vector_store = PGVectorStore.from_params(
    database="miapp",
    host="localhost",
    password="password",
    port=5432,
    user="usuario",
    table_name="llama_index_docs",
    embed_dim=1536                    # dimensiones del embedding
)

# Crear storage context con pgvector
storage_context = StorageContext.from_defaults(vector_store=vector_store)

# Crear indice con persistencia en pgvector
index = VectorStoreIndex.from_documents(
    documents,
    storage_context=storage_context
)

# Cargar indice existente (sin re-indexar)
index = VectorStoreIndex.from_vector_store(vector_store)
query_engine = index.as_query_engine(similarity_top_k=5)
```

### 2.4 Configurar chunking en LlamaIndex

```python
from llama_index.core.node_parser import SentenceSplitter, SemanticSplitterNodeParser
from llama_index.core import Settings

# Chunking por oraciones (default)
Settings.node_parser = SentenceSplitter(
    chunk_size=512,                   # tokens por chunk
    chunk_overlap=50                  # tokens de solapamiento
)

# Chunking semantico (agrupa por tema)
Settings.node_parser = SemanticSplitterNodeParser(
    buffer_size=1,                    # frases de contexto
    breakpoint_percentile_threshold=95,  # umbral de cambio de tema
    embed_model=Settings.embed_model
)

# Crear indice con el splitter configurado
index = VectorStoreIndex.from_documents(documents)
```

### 2.5 Chat engine con memoria

```python
# Query engine: una pregunta, una respuesta (sin memoria)
query_engine = index.as_query_engine()

# Chat engine: conversacion con memoria
chat_engine = index.as_chat_engine(
    chat_mode="condense_plus_context",   # reformula la pregunta con el historial
    similarity_top_k=5
)

# Conversacion con contexto
response1 = chat_engine.chat("Que lenguajes soporta el framework?")
print(response1)

response2 = chat_engine.chat("Y cual es el mas rapido?")  # entiende "el framework"
print(response2)

# Resetear memoria
chat_engine.reset()
```

---

## 3. Document loaders especializados

### 3.1 PyPDF para PDFs

```python
from langchain_community.document_loaders import PyPDFLoader

# Cargar PDF (un Document por pagina)
loader = PyPDFLoader("manual_tecnico.pdf")
paginas = loader.load()

print(f"Paginas cargadas: {len(paginas)}")
print(f"Pagina 1: {paginas[0].page_content[:200]}")
print(f"Metadata: {paginas[0].metadata}")
# Metadata incluye: source, page (numero de pagina)
```

### 3.2 Unstructured para multiples formatos

```python
from langchain_community.document_loaders import UnstructuredFileLoader

# Unstructured soporta: PDF, DOCX, PPTX, HTML, TXT, CSV, etc.
loader = UnstructuredFileLoader(
    "presentacion.pptx",
    mode="elements"                   # devuelve elementos individuales (titulos, parrafos, tablas)
)
elementos = loader.load()

for elem in elementos[:5]:
    print(f"  Tipo: {elem.metadata.get('category', '?')}")
    print(f"  Texto: {elem.page_content[:100]}")
```

### 3.3 Web scraping

```python
from langchain_community.document_loaders import WebBaseLoader

# Cargar una pagina web
loader = WebBaseLoader("https://docs.python.org/3/tutorial/index.html")
docs = loader.load()
print(f"Contenido: {docs[0].page_content[:200]}")
```

```python
from langchain_community.document_loaders import RecursiveUrlLoader
from bs4 import BeautifulSoup

# Cargar multiples paginas recursivamente
def extraer_texto(html: str) -> str:
    soup = BeautifulSoup(html, "html.parser")
    return soup.get_text(separator="\n", strip=True)

loader = RecursiveUrlLoader(
    url="https://docs.python.org/3/tutorial/",
    max_depth=2,                      # profundidad maxima de enlaces
    extractor=extraer_texto           # funcion para extraer texto del HTML
)
docs = loader.load()
print(f"Paginas cargadas: {len(docs)}")
```

### 3.4 DirectoryLoader para multiples archivos

```python
from langchain_community.document_loaders import DirectoryLoader, TextLoader

# Cargar todos los .txt de un directorio
loader = DirectoryLoader(
    "./documentos/",
    glob="**/*.txt",                  # patron de archivos
    loader_cls=TextLoader,            # clase de loader a usar
    show_progress=True,               # barra de progreso
    use_multithreading=True           # cargar en paralelo
)
docs = loader.load()
print(f"Documentos cargados: {len(docs)}")
```

### 3.5 Loader personalizado

```python
from langchain_core.documents import Document
import psycopg2

def cargar_desde_bd(query: str, connection_params: dict) -> list[Document]:
    """Carga documentos desde una base de datos PostgreSQL."""
    conn = psycopg2.connect(**connection_params)
    cur = conn.cursor()
    cur.execute(query)

    docs = []
    for row in cur.fetchall():
        docs.append(Document(
            page_content=row[1],              # columna de contenido
            metadata={
                "id": row[0],
                "titulo": row[2],
                "categoria": row[3],
                "fuente": "base_de_datos"
            }
        ))

    cur.close()
    conn.close()
    return docs

# Uso
docs = cargar_desde_bd(
    "SELECT id, contenido, titulo, categoria FROM articulos WHERE publicado = true",
    {"host": "localhost", "dbname": "miapp", "user": "usuario", "password": "password"}
)
```

---

## 4. Chunking avanzado

### 4.1 RecursiveCharacterTextSplitter

El splitter mas usado. Intenta cortar por limites naturales del texto:

```python
from langchain.text_splitter import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,                   # tamano maximo del chunk (caracteres)
    chunk_overlap=100,                # solapamiento entre chunks consecutivos
    length_function=len,              # funcion para medir tamano
    separators=[                      # orden de prioridad para cortar
        "\n\n",                       # primero: parrafos
        "\n",                         # segundo: lineas
        ". ",                         # tercero: oraciones
        ", ",                         # cuarto: clausulas
        " ",                          # quinto: palabras
        ""                            # ultimo: caracteres
    ]
)

chunks = splitter.split_documents(documentos)
```

### 4.2 Splitter por tokens

```python
from langchain.text_splitter import TokenTextSplitter

# Divide por tokens reales (no por caracteres)
splitter = TokenTextSplitter(
    chunk_size=400,                   # tokens por chunk
    chunk_overlap=50,                 # tokens de solapamiento
    encoding_name="cl100k_base"       # tokenizer de OpenAI
)

chunks = splitter.split_documents(documentos)
```

### 4.3 Splitter para Markdown

```python
from langchain.text_splitter import MarkdownHeaderTextSplitter

# Divide respetando la estructura de Markdown
headers_to_split_on = [
    ("#", "titulo_h1"),
    ("##", "titulo_h2"),
    ("###", "titulo_h3"),
]

md_splitter = MarkdownHeaderTextSplitter(
    headers_to_split_on=headers_to_split_on
)

texto_md = """
# Introduccion
Este es el capitulo introductorio.

## Seccion 1
Contenido de la seccion 1.

### Subseccion 1.1
Detalle de la subseccion.

## Seccion 2
Contenido de la seccion 2.
"""

chunks = md_splitter.split_text(texto_md)
for chunk in chunks:
    print(f"  Metadata: {chunk.metadata}")
    print(f"  Contenido: {chunk.page_content[:50]}")
```

### 4.4 SemanticChunker de LangChain

```python
from langchain_experimental.text_splitter import SemanticChunker
from langchain_openai import OpenAIEmbeddings

embeddings = OpenAIEmbeddings(model="text-embedding-3-small")

# Chunking semantico: agrupa oraciones por similitud de significado
semantic_splitter = SemanticChunker(
    embeddings=embeddings,
    breakpoint_threshold_type="percentile",  # metodo de deteccion de cambio de tema
    breakpoint_threshold_amount=95           # percentil para considerar cambio de tema
)

chunks = semantic_splitter.split_documents(documentos)
print(f"Chunks semanticos: {len(chunks)}")
for chunk in chunks[:3]:
    print(f"  [{len(chunk.page_content)} chars] {chunk.page_content[:80]}...")
```

### 4.5 Parent Document Retriever

Almacena chunks pequenos para busqueda precisa, pero devuelve el documento padre
completo para dar mas contexto al LLM:

```python
from langchain.retrievers import ParentDocumentRetriever
from langchain.storage import InMemoryStore
from langchain.text_splitter import RecursiveCharacterTextSplitter

# Splitter para chunks pequenos (busqueda)
child_splitter = RecursiveCharacterTextSplitter(chunk_size=200, chunk_overlap=50)

# Splitter para chunks grandes (contexto al LLM)
parent_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=100)

# Almacen para los documentos padre
docstore = InMemoryStore()

retriever = ParentDocumentRetriever(
    vectorstore=vectorstore,          # vectorstore para chunks pequenos
    docstore=docstore,                # almacen para docs padre
    child_splitter=child_splitter,
    parent_splitter=parent_splitter
)

# Ingestar (almacena chunks pequenos en vectorstore, padres en docstore)
retriever.add_documents(documentos)

# Buscar: encuentra por chunks pequenos, devuelve docs padre
resultados = retriever.invoke("politica de devolucion")
# Los resultados son los documentos padre (1000 chars), no los chunks (200 chars)
```

---

## 5. Estrategias de retrieval

### 5.1 Similarity search (basico)

```python
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 5}
)

docs = retriever.invoke("como usar FastAPI")
```

### 5.2 Maximum Marginal Relevance (MMR)

```python
# MMR: equilibra relevancia y diversidad
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 5,                       # resultados finales
        "fetch_k": 20,                # candidatos iniciales
        "lambda_mult": 0.5            # 0=diversidad, 1=relevancia
    }
)

docs = retriever.invoke("como usar FastAPI")
```

### 5.3 Similarity score threshold

```python
# Solo devuelve documentos con similitud por encima del umbral
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "score_threshold": 0.7,       # umbral minimo de similitud
        "k": 10                       # maximo de resultados
    }
)

docs = retriever.invoke("como usar FastAPI")
```

### 5.4 Contextual Compression

```python
from langchain.retrievers import ContextualCompressionRetriever
from langchain.retrievers.document_compressors import LLMChainExtractor

# El compressor usa un LLM para extraer solo la parte relevante de cada chunk
compressor = LLMChainExtractor.from_llm(llm)

# Retriever base
base_retriever = vectorstore.as_retriever(search_kwargs={"k": 10})

# Retriever con compresion: primero recupera, luego comprime
compression_retriever = ContextualCompressionRetriever(
    base_compressor=compressor,
    base_retriever=base_retriever
)

# Los documentos devueltos contienen solo la informacion relevante
docs = compression_retriever.invoke("politica de devolucion")
for doc in docs:
    print(f"  Comprimido: {doc.page_content[:100]}...")
```

### 5.5 Ensemble Retriever (busqueda hibrida)

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# Retriever vectorial
vector_retriever = vectorstore.as_retriever(search_kwargs={"k": 5})

# Retriever BM25 (busqueda por palabras clave)
bm25_retriever = BM25Retriever.from_documents(chunks, k=5)

# Ensemble: combina ambos con pesos
ensemble_retriever = EnsembleRetriever(
    retrievers=[vector_retriever, bm25_retriever],
    weights=[0.7, 0.3]               # 70% vectorial, 30% BM25
)

docs = ensemble_retriever.invoke("como configurar FastAPI con PostgreSQL")
```

### 5.6 Multi-query Retriever

```python
from langchain.retrievers import MultiQueryRetriever

# Genera multiples versiones de la pregunta para mejor cobertura
multi_retriever = MultiQueryRetriever.from_llm(
    retriever=vectorstore.as_retriever(search_kwargs={"k": 5}),
    llm=llm
)

# Internamente genera variaciones como:
# Original: "como usar FastAPI"
# Variacion 1: "tutorial de FastAPI para principiantes"
# Variacion 2: "configuracion basica de FastAPI"
# Variacion 3: "primeros pasos con el framework FastAPI"
# Busca con todas y combina resultados unicos
docs = multi_retriever.invoke("como usar FastAPI")
```

---

## 6. Evaluacion con RAGAS

### 6.1 Instalacion

```bash
pip install ragas
```

### 6.2 Metricas de RAGAS

RAGAS (Retrieval Augmented Generation Assessment) evalua cuatro dimensiones:

```
1. Faithfulness (Fidelidad)
   - La respuesta se basa en el contexto recuperado?
   - Penaliza alucinaciones e informacion inventada
   - Rango: 0 a 1 (1 = completamente fiel al contexto)

2. Answer Relevancy (Relevancia de la respuesta)
   - La respuesta es relevante para la pregunta?
   - Penaliza respuestas vagas o fuera de tema
   - Rango: 0 a 1

3. Context Precision (Precision del contexto)
   - Los chunks recuperados son relevantes para la pregunta?
   - Mide la calidad del retrieval
   - Rango: 0 a 1

4. Context Recall (Recall del contexto)
   - Se recuperaron todos los chunks necesarios?
   - Requiere respuesta de referencia (ground truth)
   - Rango: 0 a 1
```

### 6.3 Preparar dataset de evaluacion

```python
from ragas import EvaluationDataset, SingleTurnSample

# Crear samples de evaluacion
samples = []
for testcase in testcases:
    # Ejecutar el pipeline RAG
    resultado = rag_chain.invoke(testcase["pregunta"])

    # Recuperar documentos usados
    docs = retriever.invoke(testcase["pregunta"])
    contextos = [doc.page_content for doc in docs]

    samples.append(SingleTurnSample(
        user_input=testcase["pregunta"],
        response=resultado,
        retrieved_contexts=contextos,
        reference=testcase["respuesta_esperada"]    # ground truth
    ))

dataset = EvaluationDataset(samples=samples)
```

### 6.4 Ejecutar evaluacion

```python
from ragas import evaluate
from ragas.metrics import (
    Faithfulness,
    ResponseRelevancy,
    LLMContextPrecisionWithoutReference,
    LLMContextRecall
)
from ragas.llms import LangchainLLMWrapper

# Configurar LLM evaluador
evaluator_llm = LangchainLLMWrapper(ChatOpenAI(model="gpt-4o-mini"))

# Definir metricas
metricas = [
    Faithfulness(llm=evaluator_llm),
    ResponseRelevancy(llm=evaluator_llm),
    LLMContextPrecisionWithoutReference(llm=evaluator_llm),
    LLMContextRecall(llm=evaluator_llm)
]

# Ejecutar evaluacion
resultado = evaluate(
    dataset=dataset,
    metrics=metricas
)

# Ver resultados
print(resultado)
# {'faithfulness': 0.87, 'answer_relevancy': 0.92, 'context_precision': 0.85, 'context_recall': 0.78}

# Convertir a DataFrame para analisis detallado
df = resultado.to_pandas()
print(df.describe())
```

### 6.5 Iterar y mejorar

```python
def evaluar_configuracion(
    chunk_size: int,
    chunk_overlap: int,
    top_k: int,
    search_type: str,
    testcases: list[dict]
) -> dict:
    """Evalua una configuracion especifica del pipeline RAG."""
    # Reconstruir pipeline con la nueva configuracion
    splitter = RecursiveCharacterTextSplitter(
        chunk_size=chunk_size,
        chunk_overlap=chunk_overlap
    )
    chunks = splitter.split_documents(documentos_originales)

    vs = FAISS.from_documents(chunks, embeddings)
    ret = vs.as_retriever(
        search_type=search_type,
        search_kwargs={"k": top_k}
    )

    chain = (
        {"context": ret | formatear_docs, "question": RunnablePassthrough()}
        | prompt | llm | StrOutputParser()
    )

    # Ejecutar y evaluar
    samples = []
    for tc in testcases:
        resp = chain.invoke(tc["pregunta"])
        docs = ret.invoke(tc["pregunta"])
        samples.append(SingleTurnSample(
            user_input=tc["pregunta"],
            response=resp,
            retrieved_contexts=[d.page_content for d in docs],
            reference=tc["respuesta_esperada"]
        ))

    resultado = evaluate(dataset=EvaluationDataset(samples=samples), metrics=metricas)
    return dict(resultado)

# Comparar configuraciones
configs = [
    {"chunk_size": 200, "chunk_overlap": 50, "top_k": 3, "search_type": "similarity"},
    {"chunk_size": 500, "chunk_overlap": 100, "top_k": 5, "search_type": "similarity"},
    {"chunk_size": 500, "chunk_overlap": 100, "top_k": 5, "search_type": "mmr"},
    {"chunk_size": 1000, "chunk_overlap": 200, "top_k": 3, "search_type": "similarity"},
]

for config in configs:
    scores = evaluar_configuracion(**config, testcases=testcases)
    print(f"Config {config}: {scores}")
```

---

## 7. FastAPI + LangChain para servir RAG

### 7.1 Estructura del proyecto

```
mi-rag-api/
  main.py                # aplicacion FastAPI
  rag/
    __init__.py
    pipeline.py          # pipeline RAG (ingestion + query)
    config.py            # configuracion
  requirements.txt
```

### 7.2 Pipeline RAG reutilizable

```python
# rag/pipeline.py
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_postgres import PGVector
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.runnables import RunnablePassthrough
from langchain_core.documents import Document

class RAGPipeline:
    def __init__(self, connection_string: str, openai_api_key: str):
        self.embeddings = OpenAIEmbeddings(
            model="text-embedding-3-small",
            openai_api_key=openai_api_key
        )
        self.llm = ChatOpenAI(
            model="gpt-4o-mini",
            temperature=0.0,
            openai_api_key=openai_api_key
        )
        self.vectorstore = PGVector(
            embeddings=self.embeddings,
            collection_name="rag_docs",
            connection=connection_string,
            use_jsonb=True
        )
        self.splitter = RecursiveCharacterTextSplitter(
            chunk_size=500,
            chunk_overlap=100
        )
        self._build_chain()

    def _build_chain(self):
        prompt = ChatPromptTemplate.from_template("""
Responde basandote UNICAMENTE en el contexto. Si no hay informacion suficiente, dilo.

Contexto:
{context}

Pregunta: {question}
""")
        retriever = self.vectorstore.as_retriever(
            search_type="similarity",
            search_kwargs={"k": 5}
        )

        self.chain = (
            {
                "context": retriever | self._formatear_docs,
                "question": RunnablePassthrough()
            }
            | prompt
            | self.llm
            | StrOutputParser()
        )

    @staticmethod
    def _formatear_docs(docs):
        return "\n\n".join(doc.page_content for doc in docs)

    def ingestar(self, textos: list[str], metadata: dict = None):
        """Ingesta textos con chunking automatico."""
        docs = [
            Document(page_content=t, metadata=metadata or {})
            for t in textos
        ]
        chunks = self.splitter.split_documents(docs)
        self.vectorstore.add_documents(chunks)
        return len(chunks)

    def consultar(self, pregunta: str) -> dict:
        """Consulta RAG con fuentes."""
        respuesta = self.chain.invoke(pregunta)
        fuentes = self.vectorstore.similarity_search(pregunta, k=5)
        return {
            "respuesta": respuesta,
            "fuentes": [
                {"contenido": d.page_content[:200], "metadata": d.metadata}
                for d in fuentes
            ]
        }
```

### 7.3 Aplicacion FastAPI

```python
# main.py
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from rag.pipeline import RAGPipeline
import os

app = FastAPI(title="RAG API", version="1.0.0")

# Inicializar pipeline al arrancar
pipeline = RAGPipeline(
    connection_string=os.getenv("DATABASE_URL", "postgresql+psycopg://usuario:password@localhost:5432/miapp"),
    openai_api_key=os.getenv("OPENAI_API_KEY")
)


class IngestionRequest(BaseModel):
    textos: list[str]
    fuente: str = "api"
    categoria: str = "general"


class ConsultaRequest(BaseModel):
    pregunta: str


class ConsultaResponse(BaseModel):
    respuesta: str
    fuentes: list[dict]


@app.post("/api/ingestar")
def ingestar(request: IngestionRequest):
    """Ingesta documentos en el pipeline RAG."""
    n_chunks = pipeline.ingestar(
        textos=request.textos,
        metadata={"fuente": request.fuente, "categoria": request.categoria}
    )
    return {"mensaje": f"{n_chunks} chunks ingestados correctamente"}


@app.post("/api/consultar", response_model=ConsultaResponse)
def consultar(request: ConsultaRequest):
    """Consulta el sistema RAG."""
    if not request.pregunta.strip():
        raise HTTPException(status_code=400, detail="La pregunta no puede estar vacia")

    resultado = pipeline.consultar(request.pregunta)
    return ConsultaResponse(**resultado)


@app.get("/api/health")
def health():
    return {"status": "ok"}
```

### 7.4 Streaming de respuestas

```python
from fastapi.responses import StreamingResponse
from langchain_core.output_parsers import StrOutputParser

@app.post("/api/consultar/stream")
async def consultar_stream(request: ConsultaRequest):
    """Consulta RAG con streaming de la respuesta."""

    async def generar():
        async for chunk in pipeline.chain.astream(request.pregunta):
            yield chunk

    return StreamingResponse(generar(), media_type="text/plain")
```

### 7.5 Docker Compose completo

```yaml
# docker-compose.yml
services:
  postgres:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: miapp
      POSTGRES_USER: usuario
      POSTGRES_PASSWORD: password
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql+psycopg://usuario:password@postgres:5432/miapp
      OPENAI_API_KEY: ${OPENAI_API_KEY}
    depends_on:
      - postgres

volumes:
  pgdata:
```

---

## 8. Comparativa: LangChain vs LlamaIndex

### 8.1 Filosofia

```
LangChain:
  - Framework general para aplicaciones LLM
  - RAG es una de muchas capacidades
  - Mas control y flexibilidad
  - Mas componentes para combinar
  - Curva de aprendizaje mayor
  - Ideal cuando necesitas personalizar cada paso

LlamaIndex:
  - Especializado en indexacion y busqueda de datos
  - RAG es su caso de uso principal
  - Menos codigo para RAG basico
  - Mas opinado (menos decisiones que tomar)
  - Curva de aprendizaje menor para RAG
  - Ideal cuando RAG es tu caso de uso principal
```

### 8.2 Tabla comparativa

| Aspecto | LangChain | LlamaIndex |
|---------|-----------|------------|
| Enfoque | Framework general LLM | Especializado en datos/RAG |
| Lineas de codigo (RAG basico) | ~30 | ~10 |
| Flexibilidad | Muy alta | Media-alta |
| Document loaders | 100+ | 100+ |
| Vector stores | 50+ | 30+ |
| Chunking | Manual (TextSplitter) | Automatico (NodeParser) |
| Retrieval avanzado | MMR, ensemble, compression, multi-query | Fusion, auto-merging, sentence window |
| Evaluacion | Via RAGAS (externo) | Built-in (ResponseEvaluator) |
| Produccion (API) | Excelente (LangServe, integra con FastAPI) | Bueno (integra con FastAPI) |
| Comunidad | Muy grande | Grande |
| Documentacion | Extensa | Extensa |

### 8.3 Cuando usar cual

```
Usa LangChain cuando:
  - Necesitas control total sobre cada paso del pipeline
  - Tu aplicacion va mas alla de RAG (agentes, cadenas complejas)
  - Necesitas busqueda hibrida, re-ranking avanzado
  - Quieres integrar con FastAPI de forma nativa

Usa LlamaIndex cuando:
  - RAG es tu caso de uso principal
  - Quieres el pipeline mas simple posible
  - Trabajas con muchos tipos de documentos
  - Quieres evaluacion integrada

Usa ambos:
  - LlamaIndex para ingestion e indexacion
  - LangChain para chains complejas y API
  - Son compatibles y se pueden combinar
```

### 8.4 Ejemplo equivalente lado a lado

```python
# === LangChain ===
from langchain_openai import OpenAIEmbeddings, ChatOpenAI
from langchain_community.vectorstores import FAISS
from langchain_community.document_loaders import TextLoader
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.chains import RetrievalQA

docs = TextLoader("datos.txt").load()
chunks = RecursiveCharacterTextSplitter(chunk_size=500, chunk_overlap=100).split_documents(docs)
vs = FAISS.from_documents(chunks, OpenAIEmbeddings())
chain = RetrievalQA.from_chain_type(ChatOpenAI(), retriever=vs.as_retriever())
respuesta = chain.invoke({"query": "mi pregunta"})


# === LlamaIndex ===
from llama_index.core import SimpleDirectoryReader, VectorStoreIndex

docs = SimpleDirectoryReader("./datos/").load_data()
index = VectorStoreIndex.from_documents(docs)
engine = index.as_query_engine()
respuesta = engine.query("mi pregunta")
```

---

## Resumen

| Concepto | Que es | Como usarlo en Python |
|---------|--------|----------------------|
| LangChain RAG | Framework para pipelines RAG con componentes encadenables | DocumentLoader -> TextSplitter -> VectorStore -> RetrievalQA |
| LCEL | LangChain Expression Language para pipelines declarativos | `retriever | format | prompt | llm | parser` |
| LlamaIndex | Framework especializado en indexacion y RAG | `VectorStoreIndex.from_documents()`, `as_query_engine()` |
| PyPDFLoader | Cargador de PDFs para LangChain | `PyPDFLoader("file.pdf").load()` |
| Unstructured | Cargador multi-formato (PDF, DOCX, PPTX, HTML) | `UnstructuredFileLoader("file").load()` |
| RecursiveCharacterTextSplitter | Chunking que respeta estructura del texto | `splitter.split_documents(docs)` con separadores jerarquicos |
| SemanticChunker | Chunking que agrupa por significado semantico | Usa embeddings para detectar cambios de tema |
| ParentDocumentRetriever | Busca por chunks pequenos, devuelve documento padre | Precision en busqueda + contexto amplio para LLM |
| MMR | Maximum Marginal Relevance: relevancia + diversidad | `search_type="mmr"`, `lambda_mult=0.5` |
| ContextualCompression | Comprime chunks para extraer solo lo relevante | `ContextualCompressionRetriever` con LLMChainExtractor |
| EnsembleRetriever | Combina retrieval vectorial + BM25 | `EnsembleRetriever(retrievers=[...], weights=[0.7, 0.3])` |
| RAGAS | Framework de evaluacion para RAG | Faithfulness, Answer Relevancy, Context Precision/Recall |
| FastAPI + LangChain | Servir pipeline RAG como API REST | `RAGPipeline` + endpoints POST/GET |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 05 | **LangChain RAG basico** | Construir un pipeline RAG completo con LangChain: cargar 10 PDFs con PyPDFLoader, chunk con RecursiveCharacterTextSplitter, almacenar en FAISS, y consultar con RetrievalQA. Comparar chain_type "stuff" vs "map_reduce" | LangChain, PyPDFLoader, FAISS, RetrievalQA, LCEL |
| 06 | **LlamaIndex RAG** | Construir el mismo pipeline con LlamaIndex: SimpleDirectoryReader, VectorStoreIndex con pgvector, query_engine y chat_engine. Comparar lineas de codigo y resultados con LangChain | LlamaIndex, SimpleDirectoryReader, VectorStoreIndex, pgvector |
| 07 | **Chunking avanzado** | Comparar cuatro estrategias de chunking (fijo, recursivo, markdown, semantico) sobre un corpus de documentacion tecnica. Medir con RAGAS cual estrategia da mejores resultados de retrieval | RecursiveCharacterTextSplitter, SemanticChunker, MarkdownSplitter, RAGAS |
| 08 | **Evaluacion con RAGAS** | Crear un dataset de 30 preguntas con respuestas esperadas, evaluar un pipeline RAG con las 4 metricas de RAGAS, iterar cambiando chunk_size, top_k, y search_type hasta maximizar faithfulness y context_precision | RAGAS, evaluate, Faithfulness, Context Precision, tuning |

---

## Recursos

- [LangChain RAG Tutorial](https://python.langchain.com/docs/tutorials/rag/)
- [LlamaIndex Starter Tutorial](https://docs.llamaindex.ai/en/stable/getting_started/starter_example/)
- [RAGAS Documentation](https://docs.ragas.io/)
- [LangChain Document Loaders](https://python.langchain.com/docs/integrations/document_loaders/)
- [LangChain Text Splitters](https://python.langchain.com/docs/concepts/text_splitters/)
- [LangChain Retrievers](https://python.langchain.com/docs/concepts/retrievers/)
- [FastAPI Documentation](https://fastapi.tiangolo.com/)
- [RAG Paper (Lewis et al. 2020)](https://arxiv.org/abs/2005.11401)

---

> **Consejo practico**: en Python, empieza con LangChain + FAISS para prototipos rapidos
> (5 minutos hasta un RAG funcional). Cuando necesites persistencia, migra a pgvector.
> Usa RAGAS desde el principio para medir: sin metricas no sabes si tus cambios mejoran
> o empeoran el pipeline. La mayoria de mejoras en RAG vienen de mejorar la calidad
> de los documentos y la estrategia de chunking, no del algoritmo de retrieval.
