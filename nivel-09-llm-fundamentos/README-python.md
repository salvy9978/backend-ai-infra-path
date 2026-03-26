# Nivel 09 — LLMs: Track Python

> Este README es el companero Python del nivel 09. Cubre los mismos conceptos fundamentales
> de LLMs pero usando el ecosistema Python: OpenAI SDK, Anthropic SDK, LangChain, tiktoken
> y las herramientas de gestion de entorno propias de Python. Si ya leiste el README principal
> (enfocado en Kotlin/Spring), aqui encontraras los equivalentes directos en Python.

---

## Contenido

1. [Entorno de desarrollo Python](#1-entorno-de-desarrollo-python)
2. [OpenAI Python SDK](#2-openai-python-sdk)
3. [Anthropic Python SDK](#3-anthropic-python-sdk)
4. [Patrones async en Python](#4-patrones-async-en-python)
5. [LangChain para Python](#5-langchain-para-python)
6. [Prompt engineering en Python](#6-prompt-engineering-en-python)
7. [Embeddings con Python](#7-embeddings-con-python)
8. [Conteo de tokens con tiktoken](#8-conteo-de-tokens-con-tiktoken)
9. [Costes y tracking en Python](#9-costes-y-tracking-en-python)
10. [Streaming de respuestas](#10-streaming-de-respuestas)
11. [Diferencias entre Python y Kotlin/Spring AI](#11-diferencias-entre-python-y-kotlinspring-ai)
12. [Resumen](#resumen)
13. [Ejercicios](#ejercicios)
14. [Recursos](#recursos)

---

## 1. Entorno de desarrollo Python

### 1.1 Entornos virtuales

Python requiere aislar dependencias por proyecto. Nunca instales paquetes globalmente
para un proyecto de LLMs.

```bash
# Opcion 1: venv (incluido en Python 3.3+)
python3 -m venv .venv
source .venv/bin/activate        # Linux/macOS
# .venv\Scripts\activate         # Windows

# Opcion 2: uv (mas rapido, recomendado)
pip install uv
uv venv
source .venv/bin/activate
uv pip install openai anthropic langchain tiktoken

# Opcion 3: poetry (gestion completa de dependencias)
pip install poetry
poetry init
poetry add openai anthropic langchain tiktoken python-dotenv
poetry shell
```

### 1.2 Gestion de variables de entorno

```bash
# Archivo .env (NUNCA commitear al repositorio)
OPENAI_API_KEY=sk-xxxxx
ANTHROPIC_API_KEY=sk-ant-xxxxx
DEFAULT_MODEL=gpt-4o-mini
DEFAULT_TEMPERATURE=0.7
MAX_TOKENS=1000
```

```python
# Cargar variables con python-dotenv
from dotenv import load_dotenv
import os

load_dotenv()  # carga el archivo .env del directorio actual

openai_api_key = os.getenv("OPENAI_API_KEY")
anthropic_api_key = os.getenv("ANTHROPIC_API_KEY")

# Validar que las variables existen
if not openai_api_key:
    raise ValueError("OPENAI_API_KEY no esta definida en .env")
```

### 1.3 Estructura de proyecto recomendada

```
proyecto-llm-python/
├── .env                    # variables de entorno (en .gitignore)
├── .gitignore
├── pyproject.toml          # dependencias (poetry) o requirements.txt
├── requirements.txt        # alternativa a pyproject.toml
├── src/
│   ├── __init__.py
│   ├── clients/
│   │   ├── __init__.py
│   │   ├── openai_client.py
│   │   └── anthropic_client.py
│   ├── prompts/
│   │   ├── __init__.py
│   │   └── templates.py
│   ├── embeddings/
│   │   ├── __init__.py
│   │   └── search.py
│   └── utils/
│       ├── __init__.py
│       ├── tokens.py
│       └── costs.py
└── tests/
    ├── __init__.py
    └── test_clients.py
```

### 1.4 Dependencias principales

```txt
# requirements.txt
openai>=1.30.0
anthropic>=0.25.0
langchain>=0.2.0
langchain-openai>=0.1.0
langchain-anthropic>=0.1.0
tiktoken>=0.7.0
python-dotenv>=1.0.0
pydantic>=2.0.0
httpx>=0.27.0
numpy>=1.26.0
```

---

## 2. OpenAI Python SDK

### 2.1 Configuracion basica

```python
from openai import OpenAI
from dotenv import load_dotenv

load_dotenv()

# El SDK lee OPENAI_API_KEY del entorno automaticamente
client = OpenAI()

# O puedes pasar la key explicitamente
# client = OpenAI(api_key="sk-xxxxx")
```

### 2.2 Chat completion basico

```python
def chat_simple(prompt: str, modelo: str = "gpt-4o-mini") -> str:
    """Envia un mensaje simple y devuelve la respuesta."""
    response = client.chat.completions.create(
        model=modelo,
        messages=[
            {"role": "system", "content": "Eres un asistente tecnico experto."},
            {"role": "user", "content": prompt}
        ],
        temperature=0.7,
        max_tokens=1000
    )
    return response.choices[0].message.content

# Uso
respuesta = chat_simple("Explica que es FastAPI en 2 frases.")
print(respuesta)
```

### 2.3 Conversacion multi-turno

```python
def chat_conversacion(mensajes: list[dict], modelo: str = "gpt-4o-mini") -> dict:
    """Mantiene una conversacion con historial de mensajes."""
    response = client.chat.completions.create(
        model=modelo,
        messages=mensajes,
        temperature=0.7,
        max_tokens=1000
    )

    # Extraer la respuesta y los datos de uso
    mensaje_respuesta = response.choices[0].message
    uso = response.usage

    return {
        "content": mensaje_respuesta.content,
        "role": mensaje_respuesta.role,
        "prompt_tokens": uso.prompt_tokens,
        "completion_tokens": uso.completion_tokens,
        "total_tokens": uso.total_tokens,
        "finish_reason": response.choices[0].finish_reason
    }

# Uso: conversacion multi-turno
historial = [
    {"role": "system", "content": "Eres un experto en Python. Responde de forma concisa."},
    {"role": "user", "content": "Que es un dataclass?"},
]

respuesta = chat_conversacion(historial)
print(respuesta["content"])

# Anadir la respuesta al historial para continuar la conversacion
historial.append({"role": "assistant", "content": respuesta["content"]})
historial.append({"role": "user", "content": "Dame un ejemplo con validacion."})

respuesta2 = chat_conversacion(historial)
print(respuesta2["content"])
```

### 2.4 Salida estructurada con response_format

```python
import json

def extraer_datos_json(texto: str) -> dict:
    """Extrae datos estructurados de texto libre usando JSON mode."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": """Extraes informacion de contacto de textos.
Responde SOLO con JSON valido con este schema:
{"nombre": "string", "email": "string|null", "telefono": "string|null", "empresa": "string|null"}"""
            },
            {"role": "user", "content": texto}
        ],
        response_format={"type": "json_object"},  # fuerza JSON valido
        temperature=0.0  # determinista para extraccion
    )

    # Parsear el JSON de la respuesta
    return json.loads(response.choices[0].message.content)

# Uso
datos = extraer_datos_json(
    "Hola, soy Maria Garcia de TechCorp. Mi email es maria@techcorp.com"
)
print(datos)
# {"nombre": "Maria Garcia", "email": "maria@techcorp.com", "telefono": null, "empresa": "TechCorp"}
```

### 2.5 Validacion con Pydantic

```python
from pydantic import BaseModel, Field
from typing import Optional

# Definir el schema con Pydantic
class ContactoExtraido(BaseModel):
    nombre: str = Field(description="Nombre completo de la persona")
    email: Optional[str] = Field(default=None, description="Direccion de email")
    telefono: Optional[str] = Field(default=None, description="Numero de telefono")
    empresa: Optional[str] = Field(default=None, description="Nombre de la empresa")

def extraer_contacto_validado(texto: str) -> ContactoExtraido:
    """Extrae y valida datos de contacto usando Pydantic."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": f"Extraes informacion de contacto. Schema: {ContactoExtraido.model_json_schema()}"
            },
            {"role": "user", "content": texto}
        ],
        response_format={"type": "json_object"},
        temperature=0.0
    )

    # Pydantic valida automaticamente el JSON
    datos_raw = json.loads(response.choices[0].message.content)
    return ContactoExtraido(**datos_raw)

# Uso: Pydantic lanza ValidationError si el JSON no cumple el schema
contacto = extraer_contacto_validado("Soy Carlos Lopez, email: carlos@ejemplo.com")
print(contacto.nombre)   # "Carlos Lopez"
print(contacto.email)    # "carlos@ejemplo.com"
print(contacto.empresa)  # None
```

### 2.6 Manejo de errores

```python
from openai import (
    OpenAI,
    APIError,
    APIConnectionError,
    RateLimitError,
    AuthenticationError,
)
import time

def chat_con_reintentos(
    client: OpenAI,
    mensajes: list[dict],
    max_reintentos: int = 3
) -> str:
    """Llamada a OpenAI con reintentos y backoff exponencial."""
    for intento in range(max_reintentos):
        try:
            response = client.chat.completions.create(
                model="gpt-4o-mini",
                messages=mensajes,
                temperature=0.7
            )
            return response.choices[0].message.content

        except RateLimitError:
            # 429: demasiadas peticiones
            espera = (2 ** intento) * 1.0  # 1s, 2s, 4s
            print(f"Rate limited. Esperando {espera}s (intento {intento + 1})")
            time.sleep(espera)

        except APIConnectionError:
            # Error de conexion: reintentar
            espera = (2 ** intento) * 0.5
            print(f"Error de conexion. Reintentando en {espera}s")
            time.sleep(espera)

        except AuthenticationError:
            # API key invalida: no reintentar
            raise ValueError("API key de OpenAI invalida. Verifica OPENAI_API_KEY.")

        except APIError as e:
            # Otros errores de la API
            if e.status_code and e.status_code >= 500:
                espera = (2 ** intento) * 0.5
                time.sleep(espera)
            else:
                raise

    raise RuntimeError(f"Fallo tras {max_reintentos} reintentos")
```

---

## 3. Anthropic Python SDK

### 3.1 Configuracion basica

```python
from anthropic import Anthropic
from dotenv import load_dotenv

load_dotenv()

# Lee ANTHROPIC_API_KEY del entorno automaticamente
client_anthropic = Anthropic()
```

### 3.2 Chat con Claude

```python
def chat_claude(prompt: str, modelo: str = "claude-sonnet-4-20250514") -> str:
    """Envia un mensaje a Claude y devuelve la respuesta."""
    message = client_anthropic.messages.create(
        model=modelo,
        max_tokens=1024,
        system="Eres un asistente tecnico experto en Python.",
        messages=[
            {"role": "user", "content": prompt}
        ]
    )
    # La respuesta de Anthropic tiene estructura diferente a OpenAI
    return message.content[0].text

# Uso
respuesta = chat_claude("Explica que es asyncio en Python.")
print(respuesta)
```

### 3.3 Diferencias clave entre OpenAI y Anthropic SDK

```python
# --- OpenAI ---
# El system prompt va como un mensaje mas en la lista
response_openai = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "system", "content": "Eres experto en Python."},  # system dentro de messages
        {"role": "user", "content": "Hola"}
    ],
    max_tokens=1000
)
texto_openai = response_openai.choices[0].message.content
tokens_openai = response_openai.usage.total_tokens

# --- Anthropic ---
# El system prompt es un parametro separado
response_anthropic = client_anthropic.messages.create(
    model="claude-sonnet-4-20250514",
    system="Eres experto en Python.",  # system como parametro aparte
    messages=[
        {"role": "user", "content": "Hola"}
    ],
    max_tokens=1000
)
texto_anthropic = response_anthropic.content[0].text  # .content[0].text, no .choices[0].message.content
tokens_anthropic = response_anthropic.usage.input_tokens + response_anthropic.usage.output_tokens
```

### 3.4 Clase unificada para ambos proveedores

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional

@dataclass
class LlmResponse:
    """Respuesta unificada independiente del proveedor."""
    content: str
    prompt_tokens: int
    completion_tokens: int
    model: str
    finish_reason: str

class LlmClient(ABC):
    """Interfaz comun para cualquier proveedor de LLM."""

    @abstractmethod
    def chat(self, mensajes: list[dict], system: Optional[str] = None) -> LlmResponse:
        pass

class OpenAiClient(LlmClient):
    def __init__(self, modelo: str = "gpt-4o-mini"):
        self.client = OpenAI()
        self.modelo = modelo

    def chat(self, mensajes: list[dict], system: Optional[str] = None) -> LlmResponse:
        msgs = []
        if system:
            msgs.append({"role": "system", "content": system})
        msgs.extend(mensajes)

        response = self.client.chat.completions.create(
            model=self.modelo,
            messages=msgs,
            temperature=0.7
        )
        choice = response.choices[0]
        return LlmResponse(
            content=choice.message.content,
            prompt_tokens=response.usage.prompt_tokens,
            completion_tokens=response.usage.completion_tokens,
            model=response.model,
            finish_reason=choice.finish_reason
        )

class AnthropicClient(LlmClient):
    def __init__(self, modelo: str = "claude-sonnet-4-20250514"):
        self.client = Anthropic()
        self.modelo = modelo

    def chat(self, mensajes: list[dict], system: Optional[str] = None) -> LlmResponse:
        kwargs = {
            "model": self.modelo,
            "messages": mensajes,
            "max_tokens": 1024
        }
        if system:
            kwargs["system"] = system

        response = self.client.messages.create(**kwargs)
        return LlmResponse(
            content=response.content[0].text,
            prompt_tokens=response.usage.input_tokens,
            completion_tokens=response.usage.output_tokens,
            model=response.model,
            finish_reason=response.stop_reason
        )

# Uso: cambiar de proveedor sin cambiar el codigo de negocio
llm: LlmClient = OpenAiClient()          # o AnthropicClient()
respuesta = llm.chat(
    [{"role": "user", "content": "Hola, que sabes hacer?"}],
    system="Eres un asistente tecnico."
)
print(respuesta.content)
print(f"Tokens usados: {respuesta.prompt_tokens + respuesta.completion_tokens}")
```

---

## 4. Patrones async en Python

### 4.1 Cliente async de OpenAI

```python
import asyncio
from openai import AsyncOpenAI

async_client = AsyncOpenAI()

async def chat_async(prompt: str) -> str:
    """Llamada asincrona a OpenAI."""
    response = await async_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {"role": "user", "content": prompt}
        ],
        temperature=0.7
    )
    return response.choices[0].message.content

# Ejecutar
resultado = asyncio.run(chat_async("Que es asyncio?"))
print(resultado)
```

### 4.2 Llamadas concurrentes

```python
async def procesar_multiples_prompts(prompts: list[str]) -> list[str]:
    """Enviar multiples prompts en paralelo para reducir latencia total."""
    tareas = [chat_async(prompt) for prompt in prompts]
    resultados = await asyncio.gather(*tareas)
    return list(resultados)

# Ejemplo: clasificar 5 textos en paralelo en lugar de secuencialmente
prompts = [
    "Clasifica: 'No puedo iniciar sesion'",
    "Clasifica: 'La pagina carga lento'",
    "Clasifica: 'Quiero exportar a PDF'",
    "Clasifica: 'Error 500 al guardar'",
    "Clasifica: 'Sugerencia: modo oscuro'"
]

resultados = asyncio.run(procesar_multiples_prompts(prompts))
for prompt, resultado in zip(prompts, resultados):
    print(f"{prompt} -> {resultado}")
```

### 4.3 Cliente async de Anthropic

```python
from anthropic import AsyncAnthropic

async_anthropic = AsyncAnthropic()

async def chat_claude_async(prompt: str) -> str:
    """Llamada asincrona a Claude."""
    message = await async_anthropic.messages.create(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    )
    return message.content[0].text
```

### 4.4 Configuracion avanzada con httpx

```python
import httpx
from openai import OpenAI

# Configurar timeouts y limites de conexion personalizados
http_client = httpx.Client(
    timeout=httpx.Timeout(
        connect=5.0,     # timeout de conexion: 5 segundos
        read=60.0,       # timeout de lectura: 60 segundos (LLMs pueden tardar)
        write=10.0,      # timeout de escritura
        pool=10.0        # timeout de pool de conexiones
    ),
    limits=httpx.Limits(
        max_connections=20,           # maximo 20 conexiones simultaneas
        max_keepalive_connections=10  # mantener 10 conexiones vivas
    )
)

# Pasar el cliente httpx personalizado a OpenAI
client = OpenAI(http_client=http_client)

# Para async, usar httpx.AsyncClient
async_http_client = httpx.AsyncClient(
    timeout=httpx.Timeout(connect=5.0, read=60.0, write=10.0, pool=10.0)
)
async_client = AsyncOpenAI(http_client=async_http_client)
```

---

## 5. LangChain para Python

### 5.1 Que es LangChain

LangChain es un framework que simplifica la construccion de aplicaciones con LLMs.
Proporciona abstracciones sobre proveedores, templates de prompts, parsers de salida
y cadenas (chains) que combinan multiples pasos.

### 5.2 ChatOpenAI y ChatAnthropic

```python
from langchain_openai import ChatOpenAI
from langchain_anthropic import ChatAnthropic
from langchain_core.messages import SystemMessage, HumanMessage

# Inicializar modelos (leen API keys del entorno)
llm_openai = ChatOpenAI(
    model="gpt-4o-mini",
    temperature=0.7,
    max_tokens=1000
)

llm_anthropic = ChatAnthropic(
    model="claude-sonnet-4-20250514",
    temperature=0.7,
    max_tokens=1000
)

# Llamada basica (misma interfaz para ambos)
mensajes = [
    SystemMessage(content="Eres un experto en Python."),
    HumanMessage(content="Que es un decorador?")
]

respuesta = llm_openai.invoke(mensajes)
print(respuesta.content)

# Cambiar a Claude sin cambiar nada mas
respuesta_claude = llm_anthropic.invoke(mensajes)
print(respuesta_claude.content)
```

### 5.3 Prompt Templates

```python
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate

# Template simple con variables
template = ChatPromptTemplate.from_messages([
    ("system", "Eres un experto en {lenguaje}. Responde en espanol."),
    ("human", "{pregunta}")
])

# Formatear y enviar
prompt_formateado = template.invoke({
    "lenguaje": "Python",
    "pregunta": "Que son los type hints?"
})
respuesta = llm_openai.invoke(prompt_formateado)
print(respuesta.content)
```

### 5.4 Few-shot con LangChain

```python
from langchain_core.prompts import ChatPromptTemplate, FewShotChatMessagePromptTemplate

# Definir los ejemplos
ejemplos = [
    {"input": "No puedo iniciar sesion", "output": '{"categoria": "autenticacion", "prioridad": "alta"}'},
    {"input": "El boton de exportar no funciona", "output": '{"categoria": "funcionalidad", "prioridad": "media"}'},
    {"input": "Me gustaria modo oscuro", "output": '{"categoria": "feature_request", "prioridad": "baja"}'},
]

# Template para cada ejemplo
ejemplo_template = ChatPromptTemplate.from_messages([
    ("human", "{input}"),
    ("ai", "{output}")
])

# Few-shot template
few_shot = FewShotChatMessagePromptTemplate(
    example_prompt=ejemplo_template,
    examples=ejemplos
)

# Template final completo
template_clasificador = ChatPromptTemplate.from_messages([
    ("system", "Clasificas tickets de soporte. Responde solo con JSON."),
    few_shot,
    ("human", "{input}")
])

# Uso
cadena = template_clasificador | llm_openai
resultado = cadena.invoke({"input": "La pagina tarda 30 segundos en cargar"})
print(resultado.content)
# {"categoria": "rendimiento", "prioridad": "alta"}
```

### 5.5 Output Parsers

```python
from langchain_core.output_parsers import JsonOutputParser
from langchain_core.prompts import ChatPromptTemplate
from pydantic import BaseModel, Field

# Definir el schema de salida con Pydantic
class ClasificacionTicket(BaseModel):
    categoria: str = Field(description="Categoria del ticket")
    prioridad: str = Field(description="Prioridad: alta, media, baja")
    resumen: str = Field(description="Resumen breve del problema")

# Crear el parser
parser = JsonOutputParser(pydantic_object=ClasificacionTicket)

# Template con instrucciones de formato del parser
template = ChatPromptTemplate.from_messages([
    ("system", "Clasificas tickets de soporte.\n{format_instructions}"),
    ("human", "{ticket}")
])

# Cadena completa: template -> LLM -> parser
cadena = template | llm_openai | parser

# Ejecutar
resultado = cadena.invoke({
    "ticket": "La aplicacion se cierra sola cuando subo archivos grandes",
    "format_instructions": parser.get_format_instructions()
})

# resultado es un dict validado, no un string
print(resultado["categoria"])   # "error"
print(resultado["prioridad"])   # "alta"
print(resultado["resumen"])     # "Crash al subir archivos grandes"
```

### 5.6 Chains (cadenas)

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# Cadena simple: prompt -> LLM -> parser de string
cadena_simple = (
    ChatPromptTemplate.from_messages([
        ("system", "Eres un experto en {tema}."),
        ("human", "{pregunta}")
    ])
    | llm_openai
    | StrOutputParser()  # extrae solo el texto
)

respuesta = cadena_simple.invoke({
    "tema": "bases de datos",
    "pregunta": "Cuando usar PostgreSQL vs MongoDB?"
})
print(respuesta)  # string directo, no un objeto Message

# Cadena secuencial: la salida de una alimenta la siguiente
cadena_analisis = (
    ChatPromptTemplate.from_messages([
        ("system", "Analiza el codigo y lista los problemas encontrados."),
        ("human", "{codigo}")
    ])
    | llm_openai
    | StrOutputParser()
)

cadena_solucion = (
    ChatPromptTemplate.from_messages([
        ("system", "Dado un analisis de problemas, proporciona soluciones concretas."),
        ("human", "Problemas encontrados:\n{analisis}")
    ])
    | llm_openai
    | StrOutputParser()
)

# Ejecutar secuencialmente
codigo = "def divide(a, b): return a / b"
analisis = cadena_analisis.invoke({"codigo": codigo})
soluciones = cadena_solucion.invoke({"analisis": analisis})
print(soluciones)
```

---

## 6. Prompt engineering en Python

### 6.1 System prompts

```python
# Mal: vago e impreciso
system_malo = "Eres un asistente util."

# Bien: especifico y estructurado
system_bueno = """Eres un asistente tecnico especializado en Python y FastAPI.

Reglas:
- Responde siempre en espanol
- Incluye ejemplos de codigo cuando sea relevante
- Si no sabes algo, dilo claramente en lugar de inventar
- Formato: usa markdown con bloques de codigo
- Longitud: respuestas concisas, maximo 200 palabras salvo que el usuario pida mas
- No incluyas disclaimers innecesarios

Contexto: el usuario es un desarrollador backend con experiencia intermedia."""
```

### 6.2 Chain of Thought en Python

```python
def razonar_paso_a_paso(problema: str) -> str:
    """Usa Chain of Thought para problemas que requieren logica."""
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[
            {
                "role": "system",
                "content": "Resuelves problemas tecnicos razonando paso a paso."
            },
            {
                "role": "user",
                "content": f"{problema}\n\nRazona paso a paso antes de dar la respuesta final."
            }
        ],
        temperature=0.2  # baja para razonamiento preciso
    )
    return response.choices[0].message.content

# Uso
resultado = razonar_paso_a_paso(
    "Si tengo 3 pods con 4 workers cada uno y cada worker maneja 50 req/s, "
    "cuantas req/s puedo manejar? Si el SLA exige 99.9% uptime y cada pod "
    "tiene 99.5% de disponibilidad individual, cual es la disponibilidad real?"
)
print(resultado)
```

### 6.3 Delimitadores y formato

```python
def analizar_codigo(codigo: str) -> str:
    """Analiza codigo usando delimitadores claros en el prompt."""
    prompt = f"""Analiza el siguiente codigo Python y devuelve los errores encontrados.

###CODIGO###
{codigo}
###FIN_CODIGO###

Formato de respuesta: lista numerada con linea y descripcion del error."""

    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        temperature=0.0
    )
    return response.choices[0].message.content
```

---

## 7. Embeddings con Python

### 7.1 Generar embeddings con OpenAI

```python
def generar_embedding(texto: str) -> list[float]:
    """Genera un embedding para un texto dado."""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=texto
    )
    return response.data[0].embedding

# Generar embeddings para multiples textos en una sola llamada
def generar_embeddings_batch(textos: list[str]) -> list[list[float]]:
    """Genera embeddings para una lista de textos (mas eficiente que uno a uno)."""
    response = client.embeddings.create(
        model="text-embedding-3-small",
        input=textos
    )
    # Ordenar por indice para mantener el orden original
    return [item.embedding for item in sorted(response.data, key=lambda x: x.index)]

# Uso
textos = [
    "Python es un lenguaje de programacion",
    "Java es un lenguaje de programacion",
    "Me gusta la pizza con jamon"
]
embeddings = generar_embeddings_batch(textos)
print(f"Dimension de cada embedding: {len(embeddings[0])}")  # 1536
```

### 7.2 Similitud del coseno con numpy

```python
import numpy as np

def cosine_similarity(a: list[float], b: list[float]) -> float:
    """Calcula la similitud del coseno entre dos vectores."""
    vec_a = np.array(a)
    vec_b = np.array(b)
    # similitud = (a . b) / (||a|| * ||b||)
    return float(np.dot(vec_a, vec_b) / (np.linalg.norm(vec_a) * np.linalg.norm(vec_b)))

# Comparar embeddings
sim_python_java = cosine_similarity(embeddings[0], embeddings[1])
sim_python_pizza = cosine_similarity(embeddings[0], embeddings[2])
print(f"Python-Java: {sim_python_java:.4f}")   # ~0.95
print(f"Python-Pizza: {sim_python_pizza:.4f}")  # ~0.15
```

### 7.3 Busqueda semantica en memoria

```python
from dataclasses import dataclass

@dataclass
class Documento:
    id: str
    texto: str
    embedding: list[float]

class BuscadorSemantico:
    """Motor de busqueda semantica simple en memoria."""

    def __init__(self):
        self.documentos: list[Documento] = []

    def indexar(self, documentos: list[dict]):
        """Indexa una lista de documentos generando embeddings."""
        textos = [doc["texto"] for doc in documentos]
        embeddings = generar_embeddings_batch(textos)

        for doc, emb in zip(documentos, embeddings):
            self.documentos.append(Documento(
                id=doc["id"],
                texto=doc["texto"],
                embedding=emb
            ))

    def buscar(self, query: str, top_k: int = 3) -> list[tuple[Documento, float]]:
        """Busca los documentos mas similares a la query."""
        query_embedding = generar_embedding(query)

        # Calcular similitud con todos los documentos
        resultados = []
        for doc in self.documentos:
            sim = cosine_similarity(query_embedding, doc.embedding)
            resultados.append((doc, sim))

        # Ordenar por similitud descendente
        resultados.sort(key=lambda x: x[1], reverse=True)
        return resultados[:top_k]

# Uso
buscador = BuscadorSemantico()
buscador.indexar([
    {"id": "1", "texto": "FastAPI es un framework web moderno para Python"},
    {"id": "2", "texto": "PostgreSQL es una base de datos relacional"},
    {"id": "3", "texto": "Docker permite empaquetar aplicaciones en contenedores"},
    {"id": "4", "texto": "Redis es un almacen de datos en memoria tipo clave-valor"},
])

# Busqueda por significado, no por palabras clave
resultados = buscador.buscar("como crear una API REST")
for doc, similitud in resultados:
    print(f"  [{similitud:.3f}] {doc.texto}")
# FastAPI aparecera primero aunque la query no contiene "FastAPI"
```

---

## 8. Conteo de tokens con tiktoken

### 8.1 Uso basico

```python
import tiktoken

def contar_tokens(texto: str, modelo: str = "gpt-4o") -> int:
    """Cuenta los tokens de un texto para un modelo dado."""
    # Obtener el encoding del modelo
    encoding = tiktoken.encoding_for_model(modelo)
    tokens = encoding.encode(texto)
    return len(tokens)

# Ejemplos
texto_es = "Hola, necesito ayuda con mi aplicacion en Python"
texto_codigo = "def factorial(n): return 1 if n <= 1 else n * factorial(n - 1)"

print(f"Texto espanol: {contar_tokens(texto_es)} tokens")
print(f"Codigo Python: {contar_tokens(texto_codigo)} tokens")
```

### 8.2 Desglose de tokens

```python
def desglosar_tokens(texto: str, modelo: str = "gpt-4o"):
    """Muestra como se tokeniza un texto (util para debugging)."""
    encoding = tiktoken.encoding_for_model(modelo)
    tokens = encoding.encode(texto)

    print(f"Texto: '{texto}'")
    print(f"Total tokens: {len(tokens)}")
    print(f"Token IDs: {tokens}")

    # Decodificar cada token individualmente
    for i, token_id in enumerate(tokens):
        token_texto = encoding.decode([token_id])
        print(f"  Token {i}: '{token_texto}' (id: {token_id})")

# Uso
desglosar_tokens("La programacion en Python es productiva")
# Token 0: 'La' (id: 8921)
# Token 1: ' program' (id: 2068)
# Token 2: 'acion' (id: 33173)
# ...
```

### 8.3 Estimar tokens de una conversacion completa

```python
def contar_tokens_mensajes(mensajes: list[dict], modelo: str = "gpt-4o") -> int:
    """Cuenta tokens de una lista de mensajes incluyendo overhead del formato."""
    encoding = tiktoken.encoding_for_model(modelo)

    # Cada mensaje tiene overhead por el formato de chat
    # (esto es una aproximacion; el overhead exacto varia por modelo)
    tokens_por_mensaje = 3  # <|start|>role\ncontent<|end|>
    tokens_total = 0

    for mensaje in mensajes:
        tokens_total += tokens_por_mensaje
        for key, value in mensaje.items():
            tokens_total += len(encoding.encode(str(value)))

    tokens_total += 3  # overhead de la respuesta del asistente
    return tokens_total

# Uso
mensajes = [
    {"role": "system", "content": "Eres un experto en Python."},
    {"role": "user", "content": "Que son los generadores?"},
    {"role": "assistant", "content": "Los generadores son funciones que usan yield..."},
    {"role": "user", "content": "Dame un ejemplo practico."}
]

tokens = contar_tokens_mensajes(mensajes)
print(f"Tokens del prompt completo: {tokens}")
```

---

## 9. Costes y tracking en Python

### 9.1 Calculadora de costes

```python
from dataclasses import dataclass
from typing import Optional

# Precios por millon de tokens (actualizar segun proveedor)
PRECIOS = {
    "gpt-4o": {"input": 2.50, "output": 10.00},
    "gpt-4o-mini": {"input": 0.15, "output": 0.60},
    "claude-3-5-sonnet": {"input": 3.00, "output": 15.00},
    "claude-3-haiku": {"input": 0.25, "output": 1.25},
    "text-embedding-3-small": {"input": 0.02, "output": 0.0},
}

@dataclass
class UsageRecord:
    modelo: str
    prompt_tokens: int
    completion_tokens: int
    endpoint: Optional[str] = None

    @property
    def coste_input(self) -> float:
        precio = PRECIOS.get(self.modelo, {"input": 0})["input"]
        return (self.prompt_tokens / 1_000_000) * precio

    @property
    def coste_output(self) -> float:
        precio = PRECIOS.get(self.modelo, {"output": 0})["output"]
        return (self.completion_tokens / 1_000_000) * precio

    @property
    def coste_total(self) -> float:
        return self.coste_input + self.coste_output

# Ejemplo
uso = UsageRecord(
    modelo="gpt-4o-mini",
    prompt_tokens=500,
    completion_tokens=300,
    endpoint="/api/clasificar"
)
print(f"Coste de esta llamada: ${uso.coste_total:.6f}")
```

### 9.2 Tracker de costes acumulados

```python
from collections import defaultdict
from datetime import datetime

class CostTracker:
    """Registra y acumula costes de uso de LLMs."""

    def __init__(self):
        self.registros: list[UsageRecord] = []
        self.costes_por_endpoint: dict[str, float] = defaultdict(float)
        self.costes_por_modelo: dict[str, float] = defaultdict(float)

    def registrar(self, record: UsageRecord):
        """Registra un uso de LLM."""
        self.registros.append(record)
        self.costes_por_endpoint[record.endpoint or "desconocido"] += record.coste_total
        self.costes_por_modelo[record.modelo] += record.coste_total

    def reporte(self) -> dict:
        """Genera un reporte de costes."""
        total = sum(r.coste_total for r in self.registros)
        total_tokens = sum(r.prompt_tokens + r.completion_tokens for r in self.registros)
        return {
            "total_usd": round(total, 6),
            "total_llamadas": len(self.registros),
            "total_tokens": total_tokens,
            "por_endpoint": dict(self.costes_por_endpoint),
            "por_modelo": dict(self.costes_por_modelo),
        }

# Uso global
tracker = CostTracker()

# Despues de cada llamada a la API
def chat_con_tracking(prompt: str) -> str:
    response = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}]
    )

    # Registrar el uso
    tracker.registrar(UsageRecord(
        modelo="gpt-4o-mini",
        prompt_tokens=response.usage.prompt_tokens,
        completion_tokens=response.usage.completion_tokens,
        endpoint="/chat"
    ))

    return response.choices[0].message.content

# Revisar costes
print(tracker.reporte())
```

### 9.3 Estimacion de costes antes de enviar

```python
def estimar_coste_antes_de_enviar(
    mensajes: list[dict],
    modelo: str = "gpt-4o-mini",
    max_completion_tokens: int = 1000
) -> dict:
    """Estima el coste de una llamada ANTES de hacerla."""
    tokens_prompt = contar_tokens_mensajes(mensajes, modelo)
    precios = PRECIOS.get(modelo, {"input": 0, "output": 0})

    coste_input = (tokens_prompt / 1_000_000) * precios["input"]
    coste_output_max = (max_completion_tokens / 1_000_000) * precios["output"]

    return {
        "tokens_prompt": tokens_prompt,
        "max_completion_tokens": max_completion_tokens,
        "coste_estimado_min": round(coste_input, 6),
        "coste_estimado_max": round(coste_input + coste_output_max, 6),
    }

# Uso: verificar coste antes de enviar un prompt largo
mensajes = [{"role": "user", "content": "Un texto muy largo..." * 100}]
estimacion = estimar_coste_antes_de_enviar(mensajes)
print(f"Coste estimado: ${estimacion['coste_estimado_min']} - ${estimacion['coste_estimado_max']}")
```

---

## 10. Streaming de respuestas

### 10.1 Streaming con OpenAI

```python
def chat_streaming(prompt: str):
    """Muestra la respuesta token a token conforme se genera."""
    stream = client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True  # activar streaming
    )

    texto_completo = ""
    for chunk in stream:
        # Cada chunk contiene un fragmento de la respuesta
        delta = chunk.choices[0].delta
        if delta.content:
            print(delta.content, end="", flush=True)
            texto_completo += delta.content

    print()  # salto de linea al final
    return texto_completo

# Uso
respuesta = chat_streaming("Escribe un poema corto sobre Python.")
```

### 10.2 Streaming async

```python
async def chat_streaming_async(prompt: str) -> str:
    """Streaming asincrono con OpenAI."""
    stream = await async_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    texto_completo = ""
    async for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content:
            print(delta.content, end="", flush=True)
            texto_completo += delta.content

    print()
    return texto_completo
```

### 10.3 Streaming con Anthropic

```python
def chat_claude_streaming(prompt: str) -> str:
    """Streaming con el SDK de Anthropic."""
    texto_completo = ""

    with client_anthropic.messages.stream(
        model="claude-sonnet-4-20250514",
        max_tokens=1024,
        messages=[{"role": "user", "content": prompt}]
    ) as stream:
        for text in stream.text_stream:
            print(text, end="", flush=True)
            texto_completo += text

    print()
    return texto_completo
```

### 10.4 Streaming para APIs web (FastAPI)

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse

app = FastAPI()

async def generar_stream(prompt: str):
    """Generador async que emite tokens como Server-Sent Events."""
    stream = await async_client.chat.completions.create(
        model="gpt-4o-mini",
        messages=[{"role": "user", "content": prompt}],
        stream=True
    )

    async for chunk in stream:
        delta = chunk.choices[0].delta
        if delta.content:
            # Formato SSE (Server-Sent Events)
            yield f"data: {delta.content}\n\n"

    yield "data: [DONE]\n\n"

@app.get("/chat/stream")
async def chat_endpoint(prompt: str):
    return StreamingResponse(
        generar_stream(prompt),
        media_type="text/event-stream"
    )
```

---

## 11. Diferencias entre Python y Kotlin/Spring AI

### 11.1 Comparativa rapida

| Aspecto | Python | Kotlin/Spring AI |
|---------|--------|------------------|
| SDK oficial | `openai`, `anthropic` (1ra clase) | WebClient manual o Spring AI |
| Async | `asyncio` + `AsyncOpenAI` | Coroutines / WebFlux |
| Streaming | Generadores (`for chunk in stream`) | `Flux<String>` en WebFlux |
| Validacion | Pydantic | `@ConfigurationProperties`, Jackson |
| Tokens | tiktoken (nativo OpenAI) | jtokkit (port a JVM) |
| Entorno | venv/poetry/uv + .env | Gradle/Maven + application.yml |
| Framework LLM | LangChain (maduro, enorme ecosistema) | Spring AI (mas nuevo, en crecimiento) |
| Tipado | Opcional (type hints) | Estatico (compilador) |
| Ecosistema ML | numpy, pandas, scikit-learn | Limitado en JVM |

### 11.2 Cuando elegir cada uno

```
Python es mejor cuando:
  - Prototipado rapido de ideas con LLMs
  - Data science / ML junto con LLMs
  - El equipo ya trabaja en Python
  - Necesitas LangChain con todas sus integraciones
  - Scripts y herramientas CLI

Kotlin/Spring es mejor cuando:
  - Ya tienes un backend Spring Boot en produccion
  - Necesitas tipado estatico fuerte
  - La aplicacion LLM es parte de un monolito JVM
  - Necesitas el ecosistema Spring (Security, Data, Cloud)
  - Rendimiento en aplicaciones de alta concurrencia
```

### 11.3 Equivalencias de codigo

```python
# Python: llamada basica
from openai import OpenAI
client = OpenAI()
response = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "Hola"}]
)
print(response.choices[0].message.content)
```

```kotlin
// Kotlin/Spring: equivalente
val request = mapOf(
    "model" to "gpt-4o-mini",
    "messages" to listOf(mapOf("role" to "user", "content" to "Hola"))
)
val response = webClient.post()
    .uri("https://api.openai.com/v1/chat/completions")
    .header("Authorization", "Bearer $apiKey")
    .bodyValue(request)
    .retrieve()
    .bodyToMono(ChatCompletion::class.java)
    .block()
println(response?.choices?.first()?.message?.content)
```

Python requiere ~5 lineas; Kotlin sin Spring AI requiere ~10-15 lineas y definir
las data classes manualmente. Con Spring AI la diferencia se reduce, pero el
ecosistema Python sigue siendo mas conciso para LLMs.

---

## Resumen

| Concepto | Herramienta Python | Equivalente Kotlin/JVM |
|---------|-------------------|----------------------|
| SDK OpenAI | `openai` (pip) | WebClient + data classes / Spring AI |
| SDK Anthropic | `anthropic` (pip) | WebClient + data classes |
| Async | `AsyncOpenAI` + `asyncio` | Coroutines + WebFlux |
| Streaming | `stream=True` + generadores | `Flux<ServerSentEvent>` |
| Prompts | LangChain `ChatPromptTemplate` | String templates / Spring AI prompts |
| Output parsing | LangChain `JsonOutputParser` + Pydantic | Jackson + data classes |
| Tokens | `tiktoken` | `jtokkit` |
| Costes | Tracking manual con dataclasses | Tracking manual con data classes |
| Validacion | Pydantic `BaseModel` | `@ConfigurationProperties` |
| Entorno | `python-dotenv` + `.env` | `application.yml` + env vars |
| HTTP client | `httpx` (bajo el SDK) | WebClient / RestTemplate |
| Cache | `functools.lru_cache`, Redis | `@Cacheable`, Caffeine |

---

## Ejercicios

| # | Ejercicio | Descripcion | Conceptos clave |
|---|-----------|-------------|-----------------|
| 05 | [OpenAI SDK en Python](ejercicio-05-python-openai-sdk/) | Crear un script Python que se conecte a OpenAI, envie prompts, procese respuestas y maneje errores. Implementar reintentos con backoff exponencial y validacion con Pydantic | openai, ChatCompletion, Pydantic, reintentos, .env |
| 06 | [Prompt engineering en Python](ejercicio-06-python-prompt-engineering/) | Construir un clasificador de tickets usando LangChain con few-shot prompting, chain of thought y output parsers. Comparar resultados entre GPT y Claude | ChatOpenAI, ChatAnthropic, PromptTemplate, few-shot, JsonOutputParser |
| 07 | [Embeddings en Python](ejercicio-07-python-embeddings/) | Generar embeddings con la API de OpenAI, implementar busqueda semantica en memoria usando numpy y comparar similitudes entre documentos | openai.embeddings, cosine_similarity, numpy, busqueda semantica |
| 08 | [Costes y streaming en Python](ejercicio-08-python-costes-streaming/) | Implementar un tracker de costes con tiktoken, respuestas streaming via FastAPI con Server-Sent Events y un endpoint de reporte de consumo | tiktoken, streaming, SSE, FastAPI, token counting, cost tracking |

---

## Recursos

- [OpenAI Python SDK - GitHub](https://github.com/openai/openai-python)
- [Anthropic Python SDK - GitHub](https://github.com/anthropics/anthropic-sdk-python)
- [LangChain Python Docs](https://python.langchain.com/docs/get_started/introduction)
- [tiktoken - GitHub](https://github.com/openai/tiktoken)
- [Pydantic v2 Docs](https://docs.pydantic.dev/latest/)
- [httpx - Async HTTP Client](https://www.python-httpx.org/)
- [FastAPI - Framework web async](https://fastapi.tiangolo.com/)
- [uv - Gestor de paquetes rapido](https://github.com/astral-sh/uv)

---

> **Consejo practico**: Python es el lenguaje con mejor soporte de primera clase para LLMs.
> Los SDKs oficiales de OpenAI y Anthropic son Python-first, y LangChain nacio en Python.
> Si tu backend es Kotlin/Spring pero necesitas experimentar rapido con LLMs, considera
> tener un microservicio Python dedicado a la capa de IA.
