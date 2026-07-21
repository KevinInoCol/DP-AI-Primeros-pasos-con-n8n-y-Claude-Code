---
name: n8n-03-agregar-tool-qdrant
description: >-
  PASO 3 del taller de n8n. Da al agente una HERRAMIENTA (tool) de base de
  conocimiento con Qdrant: añade el nodo "Qdrant Vector Store" en modo
  retrieve-as-tool conectado al AI Agent, más un nodo "Embeddings OpenAI" que lo
  alimenta. Así el agente puede consultar documentos (RAG) para responder. NO
  maneja credenciales (Qdrant y OpenAI las pone el alumno a mano después). Úsala
  cuando el alumno diga "agregar tool de qdrant", "base de conocimiento",
  "vector store", "RAG", "paso 3", o ejecute /n8n-03-agregar-tool-qdrant.
  Requiere haber ejecutado antes n8n-01 y n8n-02.
---

# Skill 3 — Añadir un tool de base de conocimiento con Qdrant (RAG)

Esta es la **tercera skill** del taller. Le da al agente una **herramienta** para
consultar una base de conocimiento vectorial en **Qdrant**. Cuando el cliente
pregunta algo cubierto por los documentos indexados, el agente busca la
información y responde con ella (patrón RAG).

El flujo quedará así:

```
Webhook (POST) ──main──▶ AI Agent
                          ▲  ▲  ▲
      ai_languageModel ───┘  │  └─── ai_tool ── Qdrant Vector Store (retrieve-as-tool)
      OpenAI Chat Model      │                        ▲
                    ai_memory┘                        └── ai_embedding ── Embeddings OpenAI
                    Postgres Chat Memory
```

## Regla de oro sobre credenciales (MUY IMPORTANTE)

> ⚠️ **NUNCA** pidas, recibas ni escribas las credenciales del alumno. Este paso
> usa DOS credenciales sensibles que coloca el alumno **manualmente** en la UI:
> la **API de Qdrant** (URL + API key) y la de **OpenAI** (para los embeddings).
> El profesor lo explicará en clase.
>
> Por eso los nodos nuevos se crean **sin** bloque `credentials`.

## Pasos que debes seguir

### 1. Verifica la conexión con n8n

Llama a `n8n_health_check` (mode `status`). Si falla, detente y dile al alumno
que revise su `.mcp.json` y reinicie Claude Code.

### 2. Localiza el flujo del taller

El flujo de cada alumno tiene un **ID distinto**, así que NO lo asumas:

1. Llama a `n8n_list_workflows` y busca el que se llame
   **`Agente Básico con conexión por Webhook`**.
2. Guarda su `id`.

Si **no existe**, detente y dile al alumno que ejecute primero las skills
`n8n-01-crear-flujo-base` y `n8n-02-agregar-memoria-postgres`.
Si aparece **más de uno**, pregúntale cuál usar.

### 3. Añade el tool de Qdrant + los embeddings

Usa `n8n_update_partial_workflow` con el `id` y estas operaciones (edición sobre
el MISMO flujo, no crees uno nuevo):

```json
{
  "id": "<ID_DEL_FLUJO>",
  "operations": [
    {
      "type": "addNode",
      "node": {
        "id": "e5e5c555-5555-4555-8555-555555555555",
        "name": "Qdrant Vector Store",
        "type": "@n8n/n8n-nodes-langchain.vectorStoreQdrant",
        "typeVersion": 1.3,
        "position": [940, 540],
        "parameters": {
          "mode": "retrieve-as-tool",
          "toolDescription": "Consulta la base de conocimiento de la inmobiliaria para responder preguntas sobre propiedades, servicios, zonas, procesos de compra/venta/alquiler y preguntas frecuentes. Úsala siempre que necesites datos concretos del negocio.",
          "qdrantCollection": {
            "__rl": true,
            "mode": "id",
            "value": "tenant_id_inmobiliaria_alpha_state"
          },
          "topK": 4
        }
      }
    },
    {
      "type": "addNode",
      "node": {
        "id": "f6f6c666-6666-4666-8666-666666666666",
        "name": "Embeddings OpenAI",
        "type": "@n8n/n8n-nodes-langchain.embeddingsOpenAi",
        "typeVersion": 1.2,
        "position": [940, 760],
        "parameters": {
          "model": "text-embedding-3-small"
        }
      }
    },
    {
      "type": "addConnection",
      "source": "Embeddings OpenAI",
      "sourceOutput": "ai_embedding",
      "target": "Qdrant Vector Store",
      "targetInput": "ai_embedding"
    },
    {
      "type": "addConnection",
      "source": "Qdrant Vector Store",
      "sourceOutput": "ai_tool",
      "target": "AI Agent",
      "targetInput": "ai_tool"
    }
  ]
}
```

Notas sobre la configuración:
- **`mode: retrieve-as-tool`**: hace que el Vector Store se comporte como una
  herramienta que el agente puede invocar por su cuenta.
- **`toolDescription`**: describe al LLM cuándo usar la herramienta. Cuanto más
  clara sea, mejor la usará el agente. Se puede ajustar al negocio del alumno.
- **`qdrantCollection: tenant_id_inmobiliaria_alpha_state`**: nombre de la
  colección en Qdrant. El alumno debe usar el nombre real de SU colección
  (cámbialo si el alumno indica otro).
- **`topK: 4`**: cuántos fragmentos recupera por consulta.
- **Embeddings** fijados a **`text-embedding-3-small`**: deben usar el **mismo
  modelo** con el que se indexó la colección en Qdrant (el flujo de ingesta del
  paso 4 usa el mismo), o las búsquedas no coincidirán. Punto clave de RAG.

### 4. Valida el flujo

Llama a `n8n_validate_workflow` con el `id`.

> ℹ️ **FALSO POSITIVO ESPERADO.** El validador dirá algo como *"Qdrant Vector
> Store cannot output ai_tool connections"* y marcará `valid: false`. **Es un
> límite del validador, NO un error real.** El validador se guía por una bandera
> estática del nodo y no reconoce el modo `retrieve-as-tool`, que SÍ está pensado
> para conectarse al AI Agent como herramienta (es el patrón oficial de RAG-as-tool
> en n8n). En n8n real funciona correctamente.
>
> Para confirmarlo, revisa la estructura con `n8n_get_workflow` (mode `structure`)
> y verifica que exista la conexión `Qdrant Vector Store --ai_tool--> AI Agent` y
> `Embeddings OpenAI --ai_embedding--> Qdrant Vector Store`. Si esas dos
> conexiones están, el paso está BIEN. Tranquiliza al alumno sobre este mensaje.

### 5. Informa al alumno

Muestra un resumen corto y amable:

- ✅ Tool de conocimiento añadido: **Qdrant Vector Store** (modo tool) conectado al
  **AI Agent**, alimentado por **Embeddings OpenAI**.
- 📋 **Qué falta (lo hace el alumno a mano en la UI de n8n):**
  1. Abrir **Qdrant Vector Store** y asignar/crear **su** credencial de **Qdrant**
     (URL del servidor + API key), y confirmar el nombre de su **colección**.
  2. Abrir **Embeddings OpenAI** y asignar su credencial de **OpenAI** (puede
     reutilizar la del paso 1).
  3. Verificar que la colección de Qdrant ya tenga documentos indexados con el
     mismo modelo de embeddings.
  *El profesor explicará estos pasos.*
- 🧪 **Cómo probarlo** (con credenciales y flujo activo): preguntar algo que solo
  esté en la base de conocimiento y ver que el agente responde con esa info:
  - `POST <URL-de-su-n8n>/webhook/agente-basico`
  - Body: `{ "sessionId": "cliente-123", "message": "¿Qué zonas cubre la inmobiliaria?" }`

## Qué NO debes hacer en esta skill

- ❌ No crees un flujo nuevo: edita el existente con `n8n_update_partial_workflow`.
- ❌ No añadas otras herramientas ni cambies el system message: eso es de otros pasos.
- ❌ No actives el flujo automáticamente.
- ❌ No crees ni modifiques credenciales (ni Qdrant ni OpenAI).
- ❌ No dupliques nodos si ya existen; si el flujo ya tiene el tool de Qdrant,
  avísale al alumno y no lo agregues otra vez.
