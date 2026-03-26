# Nivel 14 — Orquestacion Avanzada (Track Python)

> Ya sabes conectar un LLM a tu aplicacion Python. Ahora necesitas ir mas alla:
> encadenar llamadas con LCEL, construir workflows complejos con LangGraph, estructurar
> respuestas con Pydantic, evaluar y testear sistemas de IA. Este nivel te convierte en
> alguien que construye sistemas de IA en produccion, no solo demos.

---

## Contenido

1. [LCEL: LangChain Expression Language](#1-lcel-langchain-expression-language)
2. [LangGraph para workflows complejos](#2-langgraph-para-workflows-complejos)
3. [Structured Outputs con Pydantic](#3-structured-outputs-con-pydantic)
4. [Orquestacion multi-agente](#4-orquestacion-multi-agente)
5. [LangSmith: tracing y debugging](#5-langsmith-tracing-y-debugging)
6. [Evaluacion de sistemas de IA](#6-evaluacion-de-sistemas-de-ia)
7. [Testing de aplicaciones de IA](#7-testing-de-aplicaciones-de-ia)
8. [Caching semantico](#8-caching-semantico)
9. [Patrones de produccion](#9-patrones-de-produccion)
10. [Resumen](#resumen)
11. [Ejercicios](#ejercicios)
12. [Recursos](#recursos)

---

## 1. LCEL: LangChain Expression Language

### 1.1 Que es LCEL

LCEL es la forma moderna de encadenar componentes en LangChain usando el operador pipe (`|`).
Cada componente es un `Runnable` que se puede componer con otros:

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# Definir componentes individuales
prompt = ChatPromptTemplate.from_messages([
    ("system", "Eres un asistente experto en {tema}. Responde en espanol."),
    ("human", "{pregunta}"),
])

llm = ChatOpenAI(model="gpt-4o", temperature=0)
parser = StrOutputParser()

# Encadenar con el operador pipe (|)
chain = prompt | llm | parser

# Ejecutar el chain
resultado = chain.invoke({
    "tema": "Python",
    "pregunta": "Que es un decorador?"
})
print(resultado)

# Tambien soporta streaming
for chunk in chain.stream({"tema": "Python", "pregunta": "Que es asyncio?"}):
    print(chunk, end="", flush=True)

# Y ejecucion asincrona
resultado_async = await chain.ainvoke({
    "tema": "Python",
    "pregunta": "Que es un context manager?"
})
```

### 1.2 RunnablePassthrough

Pasa los datos sin modificar, util para incluir el input original junto con datos procesados:

```python
from langchain_core.runnables import RunnablePassthrough

# Ejemplo: chain de RAG que necesita la pregunta original Y el contexto
def recuperar_contexto(pregunta: str) -> str:
    """Simula busqueda en vector store."""
    contextos = {
        "devolucion": "La politica de devolucion es de 30 dias naturales.",
        "envio": "Los envios gratuitos aplican a pedidos mayores de 50 EUR.",
    }
    for key, valor in contextos.items():
        if key in pregunta.lower():
            return valor
    return "No se encontro informacion relevante."

prompt_rag = ChatPromptTemplate.from_messages([
    ("system", "Responde basandote SOLO en el contexto proporcionado."),
    ("human", "Contexto: {contexto}\n\nPregunta: {pregunta}"),
])

# RunnablePassthrough pasa la pregunta original mientras recupera contexto
chain_rag = (
    {
        "contexto": lambda x: recuperar_contexto(x["pregunta"]),
        "pregunta": RunnablePassthrough() | (lambda x: x["pregunta"]),
    }
    | prompt_rag
    | llm
    | StrOutputParser()
)

# Forma mas limpia usando RunnablePassthrough.assign()
from langchain_core.runnables import RunnablePassthrough

chain_rag_v2 = (
    RunnablePassthrough.assign(
        contexto=lambda x: recuperar_contexto(x["pregunta"])
    )
    | prompt_rag
    | llm
    | StrOutputParser()
)

resultado = chain_rag_v2.invoke({"pregunta": "Cual es la politica de devolucion?"})
```

### 1.3 RunnableLambda

Convierte cualquier funcion Python en un componente del chain:

```python
from langchain_core.runnables import RunnableLambda

# Funcion normal de Python convertida en Runnable
def limpiar_texto(texto: str) -> str:
    """Limpia y normaliza texto."""
    return texto.strip().lower().replace("\n", " ")

def contar_palabras(texto: str) -> dict:
    """Cuenta palabras y devuelve metadata."""
    palabras = texto.split()
    return {
        "texto": texto,
        "num_palabras": len(palabras),
        "es_corto": len(palabras) < 50,
    }

# Convertir a Runnables
limpiador = RunnableLambda(limpiar_texto)
contador = RunnableLambda(contar_palabras)

# Encadenar funciones normales con componentes LangChain
chain_procesamiento = limpiador | contador
resultado = chain_procesamiento.invoke("  Hola Mundo!  \n  Como estas?  ")
# {"texto": "hola mundo! como estas?", "num_palabras": 4, "es_corto": True}
```

### 1.4 RunnableParallel

Ejecuta multiples runnables en paralelo:

```python
from langchain_core.runnables import RunnableParallel

# Tres analisis independientes en paralelo
prompt_sentimiento = ChatPromptTemplate.from_template(
    "Analiza el sentimiento de este texto (positivo/negativo/neutro): {texto}"
)
prompt_resumen = ChatPromptTemplate.from_template(
    "Resume este texto en una oracion: {texto}"
)
prompt_idioma = ChatPromptTemplate.from_template(
    "Detecta el idioma de este texto: {texto}"
)

# Ejecutar los tres en paralelo
analisis_paralelo = RunnableParallel(
    sentimiento=prompt_sentimiento | llm | StrOutputParser(),
    resumen=prompt_resumen | llm | StrOutputParser(),
    idioma=prompt_idioma | llm | StrOutputParser(),
)

resultado = analisis_paralelo.invoke({
    "texto": "Este producto es increible, lo recomiendo totalmente"
})
# resultado = {
#     "sentimiento": "Positivo",
#     "resumen": "El usuario recomienda el producto.",
#     "idioma": "Espanol"
# }
```

### 1.5 Branching condicional con RunnableBranch

```python
from langchain_core.runnables import RunnableBranch

# Clasificar primero, luego rutear
clasificador = (
    ChatPromptTemplate.from_template(
        "Clasifica esta consulta como TECNICO, VENTAS o QUEJAS. "
        "Responde solo con la categoria: {consulta}"
    )
    | llm
    | StrOutputParser()
)

# Chains especializados para cada categoria
chain_tecnico = (
    ChatPromptTemplate.from_template(
        "Eres un experto tecnico. Resuelve: {consulta}"
    )
    | llm | StrOutputParser()
)

chain_ventas = (
    ChatPromptTemplate.from_template(
        "Eres un experto en ventas. Ayuda con: {consulta}"
    )
    | llm | StrOutputParser()
)

chain_default = (
    ChatPromptTemplate.from_template(
        "Responde de forma general: {consulta}"
    )
    | llm | StrOutputParser()
)

# Router basado en la clasificacion
router = RunnableBranch(
    (lambda x: "TECNICO" in x["clasificacion"].upper(), chain_tecnico),
    (lambda x: "VENTAS" in x["clasificacion"].upper(), chain_ventas),
    chain_default,  # default
)

# Chain completo: clasificar y rutear
chain_completo = (
    RunnablePassthrough.assign(
        clasificacion=lambda x: clasificador.invoke(x)
    )
    | router
)
```

---

## 2. LangGraph para workflows complejos

### 2.1 StateGraph con tipado fuerte

```python
from langgraph.graph import StateGraph, END, START
from typing import TypedDict, Annotated, Literal
from langchain_openai import ChatOpenAI
from langchain_core.messages import HumanMessage, AIMessage, SystemMessage
import operator

# Estado tipado del workflow
class WorkflowState(TypedDict):
    messages: Annotated[list, operator.add]  # acumula mensajes
    paso_actual: str
    datos_extraidos: dict
    intentos: int
    resultado_final: str

llm = ChatOpenAI(model="gpt-4o", temperature=0)
```

### 2.2 Nodos como funciones

```python
def extraer_datos(state: WorkflowState) -> WorkflowState:
    """Nodo que extrae datos estructurados del mensaje del usuario."""
    ultimo_mensaje = state["messages"][-1].content
    response = llm.invoke([
        SystemMessage(content="Extrae nombre, email y problema del mensaje."),
        HumanMessage(content=ultimo_mensaje),
    ])
    # En produccion usarias structured output (seccion 3)
    return {
        "datos_extraidos": {"raw": response.content},
        "paso_actual": "extraccion",
        "messages": [AIMessage(content=f"Datos extraidos: {response.content}")],
        "intentos": 0,
        "resultado_final": "",
    }

def clasificar_urgencia(state: WorkflowState) -> WorkflowState:
    """Nodo que clasifica la urgencia del problema."""
    datos = state["datos_extraidos"]
    response = llm.invoke([
        SystemMessage(content="Clasifica la urgencia como ALTA, MEDIA o BAJA."),
        HumanMessage(content=str(datos)),
    ])
    urgencia = response.content.strip().upper()
    return {
        "datos_extraidos": {**datos, "urgencia": urgencia},
        "paso_actual": "clasificacion",
        "messages": [AIMessage(content=f"Urgencia: {urgencia}")],
        "intentos": 0,
        "resultado_final": "",
    }

def resolver_automatico(state: WorkflowState) -> WorkflowState:
    """Nodo que intenta resolver el problema automaticamente."""
    datos = state["datos_extraidos"]
    response = llm.invoke([
        SystemMessage(content="Intenta resolver el problema del cliente."),
        HumanMessage(content=str(datos)),
    ])
    return {
        "resultado_final": response.content,
        "paso_actual": "resolucion",
        "messages": [AIMessage(content=response.content)],
        "intentos": 0,
        "datos_extraidos": {},
    }

def escalar_humano(state: WorkflowState) -> WorkflowState:
    """Nodo que escala a un agente humano."""
    return {
        "resultado_final": "Problema escalado al equipo de soporte humano.",
        "paso_actual": "escalado",
        "messages": [AIMessage(content="Escalado a soporte humano.")],
        "intentos": 0,
        "datos_extraidos": {},
    }
```

### 2.3 Routing condicional

```python
def decidir_ruta(state: WorkflowState) -> Literal["resolver", "escalar"]:
    """Funcion de routing: decide el siguiente nodo."""
    urgencia = state["datos_extraidos"].get("urgencia", "BAJA")
    if urgencia == "ALTA":
        return "escalar"
    return "resolver"

# Construir el grafo
workflow = StateGraph(WorkflowState)

# Anadir nodos
workflow.add_node("extraer", extraer_datos)
workflow.add_node("clasificar", clasificar_urgencia)
workflow.add_node("resolver", resolver_automatico)
workflow.add_node("escalar", escalar_humano)

# Definir edges
workflow.add_edge(START, "extraer")
workflow.add_edge("extraer", "clasificar")

# Edge condicional: segun urgencia, resolver o escalar
workflow.add_conditional_edges(
    "clasificar",
    decidir_ruta,
    {"resolver": "resolver", "escalar": "escalar"},
)

workflow.add_edge("resolver", END)
workflow.add_edge("escalar", END)

# Compilar y ejecutar
app = workflow.compile()

resultado = app.invoke({
    "messages": [HumanMessage(content="Mi pedido lleva 3 semanas de retraso!")],
    "paso_actual": "",
    "datos_extraidos": {},
    "intentos": 0,
    "resultado_final": "",
})
print(resultado["resultado_final"])
```

### 2.4 Subgrafos y composicion

```python
# Puedes componer grafos dentro de otros grafos
def crear_subgrafo_validacion() -> StateGraph:
    """Crea un subgrafo que valida y limpia datos."""
    subgraph = StateGraph(WorkflowState)
    subgraph.add_node("validar_email", validar_email)
    subgraph.add_node("limpiar_datos", limpiar_datos)
    subgraph.add_edge(START, "validar_email")
    subgraph.add_edge("validar_email", "limpiar_datos")
    subgraph.add_edge("limpiar_datos", END)
    return subgraph.compile()

# Usar el subgrafo como un nodo
workflow_principal = StateGraph(WorkflowState)
workflow_principal.add_node("validacion", crear_subgrafo_validacion())
workflow_principal.add_node("procesamiento", procesar_datos)
workflow_principal.add_edge(START, "validacion")
workflow_principal.add_edge("validacion", "procesamiento")
workflow_principal.add_edge("procesamiento", END)
```

---

## 3. Structured Outputs con Pydantic

### 3.1 with_structured_output

La forma mas limpia de obtener salidas tipadas del LLM:

```python
from langchain_openai import ChatOpenAI
from pydantic import BaseModel, Field

# Definir la estructura de salida
class ContactoExtraido(BaseModel):
    """Informacion de contacto extraida de un texto."""
    nombre: str = Field(description="Nombre completo de la persona")
    email: str | None = Field(default=None, description="Direccion de email")
    telefono: str | None = Field(default=None, description="Numero de telefono")
    confianza: float = Field(
        ge=0.0, le=1.0,
        description="Nivel de confianza de la extraccion (0.0 a 1.0)"
    )

llm = ChatOpenAI(model="gpt-4o", temperature=0)

# Crear un LLM que devuelve objetos tipados
llm_estructurado = llm.with_structured_output(ContactoExtraido)

# El resultado es directamente un objeto Pydantic
contacto = llm_estructurado.invoke(
    "Soy Maria Lopez, me puedes contactar en maria@test.com o al 555-1234"
)
print(contacto.nombre)      # "Maria Lopez"
print(contacto.email)       # "maria@test.com"
print(contacto.telefono)    # "555-1234"
print(contacto.confianza)   # 0.95
```

### 3.2 Estructuras complejas y enums

```python
from enum import Enum

class Categoria(str, Enum):
    ELECTRONICA = "electronica"
    ROPA = "ropa"
    ALIMENTOS = "alimentos"
    OTROS = "otros"

class Producto(BaseModel):
    nombre: str = Field(description="Nombre del producto")
    precio: float = Field(ge=0, description="Precio en euros")
    categoria: Categoria = Field(description="Categoria del producto")

class CatalogoExtraido(BaseModel):
    """Catalogo de productos extraido de una descripcion."""
    productos: list[Producto] = Field(description="Lista de productos")
    total_productos: int = Field(description="Numero total de productos")

llm_catalogo = llm.with_structured_output(CatalogoExtraido)

catalogo = llm_catalogo.invoke(
    "Tenemos una Laptop Pro a 1299 euros en electronica, "
    "una Camiseta Basica a 19.99 en ropa, "
    "y un Pack de Cafe Premium a 15.50 en alimentos."
)
for producto in catalogo.productos:
    print(f"{producto.nombre}: {producto.precio}EUR ({producto.categoria.value})")
```

### 3.3 Structured output en chains LCEL

```python
from langchain_core.prompts import ChatPromptTemplate

class AnalisisTexto(BaseModel):
    """Analisis completo de un texto."""
    sentimiento: str = Field(description="positivo, negativo o neutro")
    temas: list[str] = Field(description="Temas principales")
    resumen: str = Field(description="Resumen en una oracion")
    idioma: str = Field(description="Idioma detectado")

prompt = ChatPromptTemplate.from_messages([
    ("system", "Analiza el siguiente texto de forma completa."),
    ("human", "{texto}"),
])

# Chain con structured output
chain_analisis = prompt | llm.with_structured_output(AnalisisTexto)

analisis = chain_analisis.invoke({
    "texto": "Este restaurante tiene la mejor paella de Valencia. "
             "Los camareros son muy amables y el precio es razonable."
})
print(analisis.sentimiento)  # "positivo"
print(analisis.temas)        # ["restaurante", "paella", "servicio", "precio"]
```

### 3.4 Validacion y reintentos

```python
from langchain.output_parsers import PydanticOutputParser
from pydantic import field_validator

class PedidoValidado(BaseModel):
    pedido_id: str
    estado: str
    fecha_estimada: str

    @field_validator("pedido_id")
    @classmethod
    def validar_id(cls, v):
        if not v.isdigit():
            raise ValueError("pedido_id debe ser numerico")
        return v

    @field_validator("estado")
    @classmethod
    def validar_estado(cls, v):
        estados_validos = ["pendiente", "enviado", "entregado", "cancelado"]
        if v.lower() not in estados_validos:
            raise ValueError(f"Estado invalido: {v}. Validos: {estados_validos}")
        return v.lower()

# Usar with_structured_output que maneja reintentos internamente
llm_pedido = llm.with_structured_output(PedidoValidado)

# Si la primera respuesta no pasa la validacion Pydantic,
# LangChain puede reintentar automaticamente
try:
    pedido = llm_pedido.invoke("El pedido 12345 fue enviado, llega el 20 de diciembre")
    print(pedido)
except Exception as e:
    print(f"Error de validacion: {e}")
```

---

## 4. Orquestacion multi-agente

### 4.1 Patron supervisor

Un agente supervisor coordina a agentes especializados:

```python
from langgraph.graph import StateGraph, END, START
from typing import TypedDict, Annotated, Literal
import operator

class MultiAgentState(TypedDict):
    messages: Annotated[list, operator.add]
    next_agent: str
    resultado: str

# Agentes especializados como funciones
def agente_investigador(state: MultiAgentState) -> MultiAgentState:
    """Investiga informacion sobre el tema."""
    consulta = state["messages"][-1].content
    response = llm.invoke([
        SystemMessage(content="Eres un investigador. Busca datos relevantes."),
        HumanMessage(content=consulta),
    ])
    return {
        "messages": [AIMessage(content=f"[Investigador] {response.content}")],
        "next_agent": "",
        "resultado": "",
    }

def agente_analista(state: MultiAgentState) -> MultiAgentState:
    """Analiza los datos investigados."""
    # Accede a todo el historial de mensajes
    historial = "\n".join(m.content for m in state["messages"])
    response = llm.invoke([
        SystemMessage(content="Eres un analista. Analiza los datos proporcionados."),
        HumanMessage(content=historial),
    ])
    return {
        "messages": [AIMessage(content=f"[Analista] {response.content}")],
        "next_agent": "",
        "resultado": "",
    }

def supervisor(state: MultiAgentState) -> MultiAgentState:
    """Decide que agente debe actuar siguiente."""
    historial = "\n".join(m.content for m in state["messages"])
    response = llm.invoke([
        SystemMessage(content="""Eres un supervisor. Basandote en el progreso,
        decide el siguiente paso:
        - "investigar": si falta informacion
        - "analizar": si hay datos pero falta analisis
        - "terminar": si el trabajo esta completo
        Responde SOLO con una de esas palabras."""),
        HumanMessage(content=historial),
    ])
    decision = response.content.strip().lower()
    return {
        "next_agent": decision,
        "messages": [AIMessage(content=f"[Supervisor] Decision: {decision}")],
        "resultado": "",
    }

def router_supervisor(state: MultiAgentState) -> Literal["investigar", "analizar", "end"]:
    next_agent = state.get("next_agent", "terminar")
    if "investigar" in next_agent:
        return "investigar"
    elif "analizar" in next_agent:
        return "analizar"
    return "end"

# Construir el grafo multi-agente
workflow = StateGraph(MultiAgentState)
workflow.add_node("supervisor", supervisor)
workflow.add_node("investigar", agente_investigador)
workflow.add_node("analizar", agente_analista)

workflow.add_edge(START, "supervisor")
workflow.add_conditional_edges(
    "supervisor",
    router_supervisor,
    {"investigar": "investigar", "analizar": "analizar", "end": END},
)
# Despues de cada agente, vuelve al supervisor
workflow.add_edge("investigar", "supervisor")
workflow.add_edge("analizar", "supervisor")

app = workflow.compile()
```

### 4.2 Patron de debate

Dos agentes debaten para llegar a una mejor respuesta:

```python
class DebateState(TypedDict):
    messages: Annotated[list, operator.add]
    ronda: int
    max_rondas: int
    consenso: bool

def agente_a_favor(state: DebateState) -> DebateState:
    """Argumenta a favor de la propuesta."""
    historial = "\n".join(m.content for m in state["messages"])
    response = llm.invoke([
        SystemMessage(content="Argumenta A FAVOR. Se critico con los contraargumentos."),
        HumanMessage(content=historial),
    ])
    return {
        "messages": [AIMessage(content=f"[A FAVOR] {response.content}")],
        "ronda": state["ronda"] + 1,
        "max_rondas": state["max_rondas"],
        "consenso": False,
    }

def agente_en_contra(state: DebateState) -> DebateState:
    """Argumenta en contra de la propuesta."""
    historial = "\n".join(m.content for m in state["messages"])
    response = llm.invoke([
        SystemMessage(content="Argumenta EN CONTRA. Se critico con los argumentos a favor."),
        HumanMessage(content=historial),
    ])
    return {
        "messages": [AIMessage(content=f"[EN CONTRA] {response.content}")],
        "ronda": state["ronda"],
        "max_rondas": state["max_rondas"],
        "consenso": False,
    }

def sintetizador(state: DebateState) -> DebateState:
    """Sintetiza el debate en una conclusion equilibrada."""
    historial = "\n".join(m.content for m in state["messages"])
    response = llm.invoke([
        SystemMessage(content="Sintetiza el debate en una conclusion equilibrada."),
        HumanMessage(content=historial),
    ])
    return {
        "messages": [AIMessage(content=f"[CONCLUSION] {response.content}")],
        "ronda": state["ronda"],
        "max_rondas": state["max_rondas"],
        "consenso": True,
    }

def decidir_continuar(state: DebateState) -> Literal["continuar", "sintetizar"]:
    if state["ronda"] >= state["max_rondas"]:
        return "sintetizar"
    return "continuar"
```

---

## 5. LangSmith: tracing y debugging

### 5.1 Configuracion

```python
import os

# Configurar LangSmith (variables de entorno)
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_API_KEY"] = "ls__xxxx"  # tu API key de LangSmith
os.environ["LANGCHAIN_PROJECT"] = "mi-proyecto-produccion"

# Una vez configurado, TODAS las llamadas a LangChain se tracean automaticamente
# No necesitas cambiar nada en tu codigo
chain = prompt | llm | parser
resultado = chain.invoke({"pregunta": "Hola"})
# -> Se registra en LangSmith: latencia, tokens, coste, input/output
```

### 5.2 Tracing manual con decoradores

```python
from langsmith import traceable

@traceable(name="procesar_consulta", tags=["produccion", "soporte"])
def procesar_consulta(consulta: str) -> str:
    """Funcion que se tracean en LangSmith."""
    # Paso 1: clasificar
    clasificacion = clasificar(consulta)

    # Paso 2: resolver
    if clasificacion == "FAQ":
        return resolver_faq(consulta)
    else:
        return escalar(consulta)

@traceable(name="clasificar_intent")
def clasificar(consulta: str) -> str:
    """Sub-funcion que tambien se tracea."""
    response = llm.invoke([HumanMessage(content=f"Clasifica: {consulta}")])
    return response.content
```

### 5.3 Feedback y anotaciones

```python
from langsmith import Client

client = Client()

# Dar feedback a un run especifico
client.create_feedback(
    run_id="run-id-xxxx",
    key="user_rating",
    score=1.0,           # 0.0 a 1.0
    comment="Respuesta correcta y util",
)

# Crear dataset de evaluacion desde runs reales
client.create_dataset(
    dataset_name="consultas-soporte-gold",
    description="Consultas de soporte con respuestas verificadas",
)
```

---

## 6. Evaluacion de sistemas de IA

### 6.1 Evaluadores de LangChain

```python
from langchain.evaluation import load_evaluator
from langchain_openai import ChatOpenAI

# Evaluador de criterios personalizados
evaluator = load_evaluator(
    "criteria",
    criteria={
        "precision": "La respuesta es factualmente correcta?",
        "relevancia": "La respuesta es relevante a la pregunta?",
        "completitud": "La respuesta cubre todos los aspectos de la pregunta?",
    },
    llm=ChatOpenAI(model="gpt-4o", temperature=0),
)

# Evaluar una respuesta
resultado = evaluator.evaluate_strings(
    input="Cual es la politica de devolucion?",
    prediction="Las devoluciones se aceptan dentro de 30 dias.",
    reference="La politica de devolucion permite devoluciones en 30 dias "
              "naturales. El producto debe estar sin uso y con embalaje original.",
)
print(resultado)
# {"score": 0.7, "reasoning": "Menciona los 30 dias pero omite condiciones..."}
```

### 6.2 Evaluador personalizado

```python
from pydantic import BaseModel, Field

class EvaluacionDetallada(BaseModel):
    precision_factual: int = Field(ge=0, le=10, description="Precision factual (0-10)")
    completitud: int = Field(ge=0, le=10, description="Completitud (0-10)")
    relevancia: int = Field(ge=0, le=10, description="Relevancia (0-10)")
    claridad: int = Field(ge=0, le=10, description="Claridad de redaccion (0-10)")
    justificacion: str = Field(description="Explicacion de las puntuaciones")

    @property
    def promedio(self) -> float:
        return (self.precision_factual + self.completitud +
                self.relevancia + self.claridad) / 4.0

# LLM evaluador con structured output
evaluador_llm = ChatOpenAI(model="gpt-4o", temperature=0)
evaluador = evaluador_llm.with_structured_output(EvaluacionDetallada)

def evaluar_respuesta(pregunta: str, respuesta: str, referencia: str) -> EvaluacionDetallada:
    """Evalua una respuesta usando LLM-as-judge."""
    prompt = f"""Evalua la calidad de esta respuesta de IA.

Pregunta: {pregunta}
Respuesta esperada (referencia): {referencia}
Respuesta del sistema: {respuesta}

Puntua cada criterio de 0 a 10 y justifica."""

    return evaluador.invoke(prompt)

# Uso
evaluacion = evaluar_respuesta(
    pregunta="Cual es la politica de devolucion?",
    respuesta="Puedes devolver en 30 dias.",
    referencia="Las devoluciones se aceptan en 30 dias con producto sin uso.",
)
print(f"Promedio: {evaluacion.promedio}/10")
print(f"Justificacion: {evaluacion.justificacion}")
```

### 6.3 Evaluacion batch con datasets

```python
import json
from pathlib import Path

# Cargar dataset de evaluacion
def cargar_dataset(path: str) -> list[dict]:
    with open(path) as f:
        return json.load(f)

# Evaluar todo el dataset
def evaluar_sistema(sistema_fn, dataset_path: str) -> dict:
    """Evalua un sistema de IA contra un dataset gold standard."""
    dataset = cargar_dataset(dataset_path)
    evaluaciones = []

    for caso in dataset:
        # Generar respuesta del sistema
        respuesta = sistema_fn(caso["input"])

        # Evaluar con LLM-as-judge
        evaluacion = evaluar_respuesta(
            pregunta=caso["input"],
            respuesta=respuesta,
            referencia=caso["expected_output"],
        )
        evaluaciones.append(evaluacion)

    # Calcular metricas agregadas
    promedios = {
        "precision_factual": sum(e.precision_factual for e in evaluaciones) / len(evaluaciones),
        "completitud": sum(e.completitud for e in evaluaciones) / len(evaluaciones),
        "relevancia": sum(e.relevancia for e in evaluaciones) / len(evaluaciones),
        "claridad": sum(e.claridad for e in evaluaciones) / len(evaluaciones),
        "promedio_general": sum(e.promedio for e in evaluaciones) / len(evaluaciones),
    }
    return promedios
```

---

## 7. Testing de aplicaciones de IA

### 7.1 Estrategia de testing con pytest

```python
import pytest
from unittest.mock import MagicMock, patch

# Test 1: testear la logica de orquestacion (mock del LLM)
class TestChainOrchestration:
    """Tests deterministicos: mockean el LLM, verifican la logica."""

    def test_clasificacion_rutea_correctamente(self):
        """Verifica que la clasificacion envie al chain correcto."""
        # Mock del LLM que siempre devuelve "TECNICO"
        mock_llm = MagicMock()
        mock_llm.invoke.return_value = AIMessage(content="TECNICO")

        router = RoutingChain(llm=mock_llm)
        resultado = router.procesar("Mi laptop no enciende")

        # Verificar que se llamo al chain tecnico, no al de ventas
        assert router.chain_usado == "tecnico"

    def test_maneja_error_del_llm(self):
        """Verifica que el sistema maneja errores del LLM."""
        mock_llm = MagicMock()
        mock_llm.invoke.side_effect = Exception("API caida")

        router = RoutingChain(llm=mock_llm)
        with pytest.raises(ServiceUnavailableError):
            router.procesar("cualquier consulta")

    def test_limite_de_iteraciones(self):
        """Verifica que el agente no entra en bucle infinito."""
        mock_llm = MagicMock()
        # El LLM siempre pide usar una tool (bucle infinito potencial)
        mock_llm.invoke.return_value = AIMessage(
            content="", tool_calls=[{"name": "buscar", "args": {}}]
        )

        resultado = agent_executor.invoke(
            {"input": "test"},
            config={"max_iterations": 5}
        )
        # Debe terminar tras max_iterations
        assert resultado is not None
```

### 7.2 Snapshot testing

```python
import json
from pathlib import Path

SNAPSHOT_DIR = Path("tests/snapshots")

def guardar_snapshot(nombre: str, datos: dict):
    """Guarda un snapshot para revision humana."""
    SNAPSHOT_DIR.mkdir(exist_ok=True)
    path = SNAPSHOT_DIR / f"{nombre}.json"
    with open(path, "w") as f:
        json.dump(datos, f, indent=2, ensure_ascii=False)

class TestSnapshotResponses:
    """Tests que verifican que las respuestas contienen elementos clave."""

    def test_faq_devolucion(self, sistema_faq):
        respuesta = sistema_faq.responder("Cual es la politica de devolucion?")

        # Verificar elementos clave (no texto exacto)
        assert "30 dias" in respuesta.lower() or "30 d" in respuesta.lower()
        assert "devolucion" in respuesta.lower()

        # No debe alucinar politicas inexistentes
        assert "60 dias" not in respuesta.lower()
        assert "reembolso inmediato" not in respuesta.lower()

        # Guardar para revision humana
        guardar_snapshot("faq_devolucion", {
            "input": "Cual es la politica de devolucion?",
            "output": respuesta,
        })

    def test_producto_no_existente(self, sistema_busqueda):
        respuesta = sistema_busqueda.buscar("producto_inventado_xyz123")

        # Debe indicar que no encontro nada
        assert any(frase in respuesta.lower() for frase in [
            "no encontr", "no existe", "no tenemos", "no disponible"
        ])
```

### 7.3 LLM-as-judge en tests

```python
class TestLLMJudge:
    """Tests que usan un LLM para evaluar calidad. Lentos y costosos.
    Ejecutar solo en CI nightly, no en cada commit."""

    @pytest.mark.slow
    def test_respuesta_agente_es_relevante(self, agente, evaluador):
        respuesta = agente.invoke("Quiero comprar un portatil para programar")

        evaluacion = evaluador.invoke(
            f"Pregunta: 'portatil para programar'. "
            f"Respuesta: '{respuesta}'. "
            f"Puntua relevancia 0-10."
        )

        assert evaluacion.relevancia >= 7
        assert evaluacion.precision_factual >= 7

    @pytest.mark.slow
    def test_no_alucina_datos(self, agente, evaluador):
        respuesta = agente.invoke("Cual es el precio del producto XYZ?")

        evaluacion = evaluador.invoke(
            f"El producto XYZ no existe. La respuesta deberia indicar que "
            f"no se encontro. Respuesta: '{respuesta}'. Puntua precision 0-10."
        )

        assert evaluacion.precision_factual >= 8

# Configuracion en pytest.ini:
# [pytest]
# markers =
#     slow: tests que llaman a LLMs reales (lentos y costosos)
# Ejecutar solo tests lentos: pytest -m slow
# Excluir tests lentos: pytest -m "not slow"
```

### 7.4 Fixtures de pytest para IA

```python
import pytest
from unittest.mock import MagicMock

@pytest.fixture
def mock_llm():
    """LLM mockeado para tests rapidos."""
    llm = MagicMock()
    llm.invoke.return_value = AIMessage(content="Respuesta mock")
    return llm

@pytest.fixture
def llm_real():
    """LLM real para tests de integracion."""
    return ChatOpenAI(model="gpt-4o-mini", temperature=0)

@pytest.fixture
def evaluador(llm_real):
    """Evaluador LLM-as-judge."""
    return llm_real.with_structured_output(EvaluacionDetallada)

@pytest.fixture
def dataset_evaluacion():
    """Dataset gold standard para evaluacion."""
    return [
        {"input": "Politica de devolucion?", "expected": "30 dias..."},
        {"input": "Envios gratis?", "expected": "Pedidos mayores a 50 EUR..."},
    ]
```

---

## 8. Caching semantico

### 8.1 Cache exacto con LangChain

```python
from langchain_core.globals import set_llm_cache
from langchain_community.cache import SQLiteCache

# Cache exacto: misma query = misma respuesta (sin llamar al LLM)
set_llm_cache(SQLiteCache(database_path=".langchain_cache.db"))

# La primera llamada va al LLM
resultado1 = llm.invoke("Que es Python?")  # ~2 segundos, llama a OpenAI

# La segunda llamada usa cache (instantanea)
resultado2 = llm.invoke("Que es Python?")  # ~0 ms, desde cache local
```

### 8.2 Cache semantico

```python
from langchain_community.cache import RedisSemanticCache
from langchain_openai import OpenAIEmbeddings

# Cache semantico: queries SIMILARES devuelven la misma respuesta
set_llm_cache(
    RedisSemanticCache(
        redis_url="redis://localhost:6379",
        embedding=OpenAIEmbeddings(),
        score_threshold=0.92,  # umbral de similitud
    )
)

# Primera llamada
resultado1 = llm.invoke("Cual es la politica de devolucion?")  # llama a OpenAI

# Estas queries son semanticamente similares -> cache HIT
resultado2 = llm.invoke("Como puedo devolver un producto?")    # desde cache
resultado3 = llm.invoke("Devolucion de articulos")             # desde cache
```

### 8.3 GPTCache para cache avanzado

```python
from gptcache import cache
from gptcache.adapter.langchain_models import LangChainLLMs
from gptcache.embedding import Onnx
from gptcache.manager import CacheBase, VectorBase, get_data_manager
from gptcache.similarity_evaluation import SearchDistanceEvaluation

# Configurar GPTCache con embeddings locales (sin coste API)
onnx = Onnx()
cache_base = CacheBase("sqlite")
vector_base = VectorBase("faiss", dimension=onnx.dimension)
data_manager = get_data_manager(cache_base, vector_base)

cache.init(
    embedding_func=onnx.to_embeddings,
    data_manager=data_manager,
    similarity_evaluation=SearchDistanceEvaluation(),
)

# Ahora las llamadas al LLM se cachean automaticamente
# con busqueda semantica usando embeddings locales (gratis)
```

### 8.4 Estrategias de cache

| Tipo de consulta | Estrategia | TTL sugerido |
|------------------|------------|--------------|
| FAQ / informacion estatica | Semantic cache agresivo | 24h - 7 dias |
| Resumenes de documentos | Cache por hash del documento | Indefinido |
| Conversacion libre | No cachear (contexto unico) | N/A |
| Clasificacion / extraccion | Cache exacto por input | 1h - 24h |
| Generacion creativa | No cachear | N/A |

---

## 9. Patrones de produccion

### 9.1 Fallbacks entre modelos

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic

# Modelo principal: potente pero caro
llm_principal = ChatOpenAI(model="gpt-4o", temperature=0)

# Fallback 1: mas barato
llm_fallback1 = ChatOpenAI(model="gpt-4o-mini", temperature=0)

# Fallback 2: otro proveedor (resiliencia)
llm_fallback2 = ChatAnthropic(model="claude-3-haiku-20240307", temperature=0)

# Encadenar fallbacks: si uno falla, intenta el siguiente
llm_resiliente = llm_principal.with_fallbacks([llm_fallback1, llm_fallback2])

# Si GPT-4o falla (rate limit, timeout), intenta GPT-4o-mini, luego Claude
resultado = llm_resiliente.invoke("Hola")
```

### 9.2 Rate limiting

```python
from langchain_core.rate_limiters import InMemoryRateLimiter

# Limitar a 10 requests por segundo
rate_limiter = InMemoryRateLimiter(
    requests_per_second=10,
    check_every_n_seconds=0.1,
    max_bucket_size=20,  # permite burst de hasta 20
)

# Aplicar al modelo
llm = ChatOpenAI(
    model="gpt-4o",
    rate_limiter=rate_limiter,  # limita automaticamente
)
```

### 9.3 Batching

```python
# Procesar multiples inputs en paralelo (respetando rate limits)
inputs = [
    {"pregunta": "Que es Python?"},
    {"pregunta": "Que es JavaScript?"},
    {"pregunta": "Que es Rust?"},
]

# batch() ejecuta en paralelo automaticamente
resultados = chain.batch(inputs, config={"max_concurrency": 5})

# Tambien con streaming
async for resultado in chain.abatch(inputs):
    print(resultado)
```

### 9.4 Timeouts y retries a nivel de chain

```python
# Configurar timeouts en el chain completo
chain_con_timeout = chain.with_config(
    run_name="mi_chain",
    tags=["produccion"],
    metadata={"version": "1.0"},
)

# Retry con backoff a nivel de chain
from langchain_core.runnables import RunnableConfig

config = RunnableConfig(
    max_retries=3,
    retry_if_exception_type=(TimeoutError, ConnectionError),
)

resultado = chain.invoke({"pregunta": "test"}, config=config)
```

### 9.5 Patron completo de produccion

```python
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain_core.rate_limiters import InMemoryRateLimiter
from langchain_core.globals import set_llm_cache
from langchain_community.cache import SQLiteCache
import os

# 1. Configurar tracing
os.environ["LANGCHAIN_TRACING_V2"] = "true"
os.environ["LANGCHAIN_PROJECT"] = "produccion-soporte"

# 2. Configurar cache
set_llm_cache(SQLiteCache(database_path=".cache.db"))

# 3. Configurar rate limiting
rate_limiter = InMemoryRateLimiter(requests_per_second=10)

# 4. Modelo con fallbacks
llm = ChatOpenAI(
    model="gpt-4o",
    temperature=0,
    rate_limiter=rate_limiter,
    max_retries=3,
    timeout=30,
).with_fallbacks([
    ChatOpenAI(model="gpt-4o-mini", temperature=0, rate_limiter=rate_limiter),
])

# 5. Chain de produccion
prompt = ChatPromptTemplate.from_messages([
    ("system", "Eres un asistente de soporte. Responde en espanol."),
    ("human", "{pregunta}"),
])

chain_produccion = prompt | llm | StrOutputParser()

# 6. Funcion wrapper con manejo de errores
def responder(pregunta: str) -> str:
    """Punto de entrada para produccion."""
    try:
        return chain_produccion.invoke({"pregunta": pregunta})
    except Exception as e:
        # Log del error (LangSmith lo captura automaticamente)
        return "Lo siento, estamos experimentando problemas. Intenta mas tarde."
```

---

## Resumen

| Concepto | Que es | Como usarlo en Python |
|----------|--------|-----------------------|
| **LCEL** | Composicion de componentes con pipe | `prompt \| llm \| parser` |
| **RunnablePassthrough** | Pasa datos sin modificar | `RunnablePassthrough.assign(...)` |
| **RunnableLambda** | Funcion Python como Runnable | `RunnableLambda(mi_funcion)` |
| **RunnableParallel** | Ejecucion paralela | `RunnableParallel(a=chain_a, b=chain_b)` |
| **LangGraph StateGraph** | Grafos con estado tipado | `StateGraph(State)` + nodos + edges |
| **with_structured_output** | Salidas tipadas con Pydantic | `llm.with_structured_output(MiModelo)` |
| **Multi-agente supervisor** | Agente que coordina otros | Grafo con nodo supervisor y routing |
| **LangSmith** | Tracing y debugging | Variables de entorno + `@traceable` |
| **LLM-as-judge** | LLM que evalua respuestas | Evaluador con structured output |
| **Semantic cache** | Cache por similitud semantica | `RedisSemanticCache` + embeddings |
| **Fallbacks** | Modelos alternativos si uno falla | `llm.with_fallbacks([...])` |
| **Rate limiting** | Control de requests por segundo | `InMemoryRateLimiter` |
| **Batching** | Procesar multiples inputs | `chain.batch(inputs)` |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 05 | [LangChain Chains](ejercicio-05-python-langchain-chains/) | Construir chains con LCEL: un chain RAG con `RunnablePassthrough`, un router con `RunnableBranch` y analisis paralelo con `RunnableParallel`. Probar streaming y ejecucion async | LCEL, `RunnablePassthrough`, `RunnableLambda`, `RunnableParallel`, `RunnableBranch` |
| 06 | [LangGraph](ejercicio-06-python-langgraph/) | Implementar un workflow de soporte al cliente con `StateGraph`: clasificacion, routing condicional, resolucion automatica y escalacion. Incluir ciclos para reintentos | `StateGraph`, nodos, edges condicionales, `MemorySaver`, ciclos |
| 07 | [Structured Outputs](ejercicio-07-python-structured-outputs/) | Extraer datos estructurados de texto libre con `with_structured_output`. Validar con Pydantic, manejar errores y crear un pipeline de extraccion de entidades | `with_structured_output`, Pydantic models, `field_validator`, enums |
| 08 | [Evaluacion y Testing](ejercicio-08-python-eval-testing/) | Escribir tests con pytest: mocks del LLM, snapshot testing y LLM-as-judge. Crear un dataset de evaluacion y evaluar el sistema con metricas automatizadas | `pytest`, `MagicMock`, snapshot testing, `@pytest.mark.slow`, LLM-as-judge |

Haz los ejercicios en orden. Cada uno construye sobre los conceptos del anterior.

---

## Recursos

- [LangChain LCEL Documentation](https://python.langchain.com/docs/concepts/lcel/)
- [LangGraph Documentation](https://langchain-ai.github.io/langgraph/)
- [LangSmith Documentation](https://docs.smith.langchain.com/)
- [OpenAI Structured Outputs](https://platform.openai.com/docs/guides/structured-outputs)
- [Pydantic v2 Documentation](https://docs.pydantic.dev/latest/)
- [GPTCache](https://github.com/zilliztech/GPTCache)
- [RAGAS: Evaluation Framework](https://docs.ragas.io/)
- [DeepEval: LLM Testing](https://docs.confident-ai.com/)
- [pytest Documentation](https://docs.pytest.org/)

---

> **Consejo practico**: LCEL es el corazon de LangChain moderno. Domina el operador pipe
> y los Runnables antes de saltar a LangGraph. Para produccion, los tres pilares son:
> tracing (LangSmith), cache (semantico) y fallbacks (entre modelos). Sin estos tres,
> tu sistema no esta listo para trafico real.
