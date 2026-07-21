---
name: n8n-04-flujo-ingesta-qdrant-drive
description: >-
  PASO 4 del taller de n8n. Crea un flujo NUEVO e independiente de INGESTA (carga)
  ROBUSTA de documentos a Qdrant desde Google Drive: Google Drive Trigger ->
  Google Drive (download) -> HTTP Request (dedup: borra vectores del mismo archivo)
  -> Qdrant Vector Store (INSERT), alimentado por Default Data Loader con
  METADATOS + Recursive Character Text Splitter + Embeddings OpenAI
  (text-embedding-3-small). Es el "lado de carga" del RAG, separado del flujo del
  agente. NO maneja credenciales (Google Drive, Qdrant y OpenAI las pone el alumno
  a mano). Úsala cuando el alumno diga "cargar datos en qdrant", "flujo de
  ingesta", "indexar documentos", "paso 4", o ejecute
  /n8n-04-flujo-ingesta-qdrant-drive.
---

# Skill 4 — Flujo de ingesta ROBUSTA: Google Drive → Qdrant

Esta skill crea un **flujo NUEVO e independiente** (NO edita el flujo del agente).
Es el "lado de carga" del RAG, en versión **robusta** (no un juguete):

- **Embeddings `text-embedding-3-small`** (mismo modelo que el flujo del agente).
- **Metadatos** en cada fragmento (`file_id`, `file_name`, `ingested_at`) para
  poder filtrar, citar la fuente y borrar por documento.
- **Dedup / upsert**: antes de insertar, borra en Qdrant los vectores que ya
  existan de ese mismo archivo (por `file_id`). Así reindexar NO crea duplicados.

> 🔑 **Regla de oro del RAG:** este flujo debe usar la **MISMA colección**
> (`tenant_id_inmobiliaria_alpha_state`) y el **MISMO modelo de embeddings**
> (`text-embedding-3-small`) que el flujo del agente (skill 3).

Estructura del flujo:

```
Google Drive Trigger ─▶ Google Drive (download) ─▶ Borrar vectores previos (dedup) ─▶ Qdrant Vector Store (INSERT)
                                                                                          ▲          ▲            ▲
                                                        ai_document ── Default Data Loader (+metadatos)  ai_embedding ── Embeddings OpenAI (3-small)
                                                                            ▲
                                                    ai_textSplitter ── Recursive Character Text Splitter
```

## Regla de oro sobre credenciales (MUY IMPORTANTE)

> ⚠️ **NUNCA** pidas, recibas ni escribas las credenciales del alumno. Este flujo
> usa TRES credenciales sensibles que coloca el alumno **manualmente** en la UI:
> **Google Drive** (OAuth2), **Qdrant** (URL + API key) y **OpenAI** (embeddings).
> Los nodos se crean **sin** bloque `credentials`.

## Pasos que debes seguir

### 1. Verifica la conexión con n8n

Llama a `n8n_health_check` (mode `status`). Si falla, detente y avisa al alumno.

### 2. Crea el flujo NUEVO

Llama a `n8n_create_workflow` (flujo nuevo, NO uses update). Usa **exactamente**
estos datos:

- **name:** `Ingesta a Qdrant - Google Drive`
- **settings:** `{ "executionOrder": "v1" }`
- **nodes:**

```json
[
  {
    "id": "10000000-0000-4000-8000-000000000001",
    "name": "Google Drive Trigger",
    "type": "n8n-nodes-base.googleDriveTrigger",
    "typeVersion": 1,
    "position": [260, 300],
    "parameters": {
      "triggerOn": "specificFolder",
      "folderToWatch": {
        "__rl": true,
        "mode": "list",
        "value": "",
        "cachedResultName": "Selecciona tu carpeta en la UI"
      },
      "event": "fileCreated",
      "options": {}
    }
  },
  {
    "id": "10000000-0000-4000-8000-000000000002",
    "name": "Google Drive",
    "type": "n8n-nodes-base.googleDrive",
    "typeVersion": 3,
    "position": [480, 300],
    "parameters": {
      "operation": "download",
      "fileId": { "__rl": true, "mode": "id", "value": "={{ $json.id }}" },
      "options": {}
    }
  },
  {
    "id": "10000000-0000-4000-8000-000000000007",
    "name": "Borrar vectores previos (dedup)",
    "type": "n8n-nodes-base.httpRequest",
    "typeVersion": 4.2,
    "position": [660, 300],
    "onError": "continueRegularOutput",
    "parameters": {
      "method": "POST",
      "url": "=https://REEMPLAZA-CON-TU-QDRANT-URL/collections/tenant_id_inmobiliaria_alpha_state/points/delete",
      "authentication": "predefinedCredentialType",
      "nodeCredentialType": "qdrantApi",
      "sendBody": true,
      "specifyBody": "json",
      "jsonBody": "={ \"filter\": { \"must\": [ { \"key\": \"metadata.file_id\", \"match\": { \"value\": \"{{ $('Google Drive Trigger').item.json.id }}\" } } ] } }",
      "options": {}
    }
  },
  {
    "id": "10000000-0000-4000-8000-000000000003",
    "name": "Qdrant Vector Store",
    "type": "@n8n/n8n-nodes-langchain.vectorStoreQdrant",
    "typeVersion": 1.3,
    "position": [880, 300],
    "parameters": {
      "mode": "insert",
      "qdrantCollection": {
        "__rl": true,
        "mode": "id",
        "value": "tenant_id_inmobiliaria_alpha_state"
      },
      "options": {}
    }
  },
  {
    "id": "10000000-0000-4000-8000-000000000004",
    "name": "Default Data Loader",
    "type": "@n8n/n8n-nodes-langchain.documentDefaultDataLoader",
    "typeVersion": 1.1,
    "position": [840, 520],
    "parameters": {
      "dataType": "binary",
      "binaryMode": "allInputData",
      "loader": "auto",
      "binaryDataKey": "data",
      "textSplittingMode": "custom",
      "options": {
        "metadata": {
          "metadataValues": [
            { "name": "file_id", "value": "={{ $('Google Drive Trigger').item.json.id }}" },
            { "name": "file_name", "value": "={{ $('Google Drive Trigger').item.json.name }}" },
            { "name": "ingested_at", "value": "={{ $now.toISO() }}" }
          ]
        }
      }
    }
  },
  {
    "id": "10000000-0000-4000-8000-000000000005",
    "name": "Recursive Character Text Splitter",
    "type": "@n8n/n8n-nodes-langchain.textSplitterRecursiveCharacterTextSplitter",
    "typeVersion": 1,
    "position": [816, 704],
    "parameters": { "chunkSize": 1000, "chunkOverlap": 100 }
  },
  {
    "id": "10000000-0000-4000-8000-000000000006",
    "name": "Embeddings OpenAI",
    "type": "@n8n/n8n-nodes-langchain.embeddingsOpenAi",
    "typeVersion": 1.2,
    "position": [1100, 520],
    "parameters": { "model": "text-embedding-3-small" }
  }
]
```

- **connections:**

```json
{
  "Google Drive Trigger": {
    "main": [[{ "node": "Google Drive", "type": "main", "index": 0 }]]
  },
  "Google Drive": {
    "main": [[{ "node": "Borrar vectores previos (dedup)", "type": "main", "index": 0 }]]
  },
  "Borrar vectores previos (dedup)": {
    "main": [[{ "node": "Qdrant Vector Store", "type": "main", "index": 0 }]]
  },
  "Default Data Loader": {
    "ai_document": [[{ "node": "Qdrant Vector Store", "type": "ai_document", "index": 0 }]]
  },
  "Recursive Character Text Splitter": {
    "ai_textSplitter": [[{ "node": "Default Data Loader", "type": "ai_textSplitter", "index": 0 }]]
  },
  "Embeddings OpenAI": {
    "ai_embedding": [[{ "node": "Qdrant Vector Store", "type": "ai_embedding", "index": 0 }]]
  }
}
```

### 3. Valida el flujo

Llama a `n8n_validate_workflow` con el `id` devuelto. Debe dar `valid: true`. Es
normal la sugerencia sobre manejo de errores.

### 4. Informa al alumno

Muestra un resumen corto y amable:

- ✅ Flujo de ingesta **robusto** creado (nombre + ID), separado del flujo del agente.
- 📋 **Qué falta (lo hace el alumno a mano en la UI de n8n):**
  1. **Google Drive Trigger** y **Google Drive**: credencial de **Google Drive
     (OAuth2)** y elegir la **carpeta** a vigilar.
  2. **Borrar vectores previos (dedup)** (HTTP Request): asignar la credencial de
     **Qdrant** (`Qdrant API`, opción *Predefined Credential Type*) y **editar la
     URL** reemplazando `REEMPLAZA-CON-TU-QDRANT-URL` por el host real de su
     Qdrant. *(La URL va en el nodo porque el HTTP Request no la toma de la
     credencial; la API key sí la inyecta la credencial.)*
  3. **Qdrant Vector Store**: credencial de **Qdrant** y confirmar colección
     `tenant_id_inmobiliaria_alpha_state`.
  4. **Embeddings OpenAI**: credencial de **OpenAI** (la misma del agente).
  *El profesor explicará estos pasos.*
- 🧪 **Cómo probarlo:** subir un documento (PDF/TXT/DOCX) a la carpeta vigilada de
  Drive → el flujo se ejecuta, borra lo previo de ese archivo e inserta los
  fragmentos con metadatos. Volver a subir el MISMO archivo NO debe duplicar
  vectores (eso demuestra el dedup). Luego preguntarle al agente (paso 3) algo del
  documento.

## Notas de robustez (para explicar en clase)

- **Dedup por `file_id`:** el HTTP Request llama a la API de Qdrant
  `POST /collections/{coll}/points/delete` con un filtro por `metadata.file_id`
  antes de insertar. Es un *upsert* por documento: reindexar no ensucia la base.
- **`onError: continueRegularOutput`** en el dedup: en la PRIMERA ingesta la
  colección puede no existir todavía (el borrado daría error); así el flujo
  continúa igualmente hasta el INSERT, que crea la colección.
- **Metadatos:** guardar `file_id`/`file_name`/`ingested_at` permite filtrar por
  documento, citar la fuente y hacer el dedup. Sin metadatos, nada de esto es posible.
- **Embeddings idénticos** en ingesta y agente (`text-embedding-3-small`): si no
  coinciden, las búsquedas fallan. Es el error #1 de RAG.
- **`chunkSize 1000 / chunkOverlap 100`:** punto de conversación sobre cómo el
  tamaño del fragmento afecta la calidad del retrieval.

## Qué NO debes hacer en esta skill

- ❌ No edites el flujo del agente: este es un flujo NUEVO e independiente.
- ❌ No actives el flujo automáticamente.
- ❌ No crees ni modifiques credenciales (Google Drive, Qdrant ni OpenAI).
- ❌ No cambies el nombre de la colección ni el modelo de embeddings: deben
  coincidir con el flujo del agente.
- ❌ Si el alumno ya tiene un flujo llamado `Ingesta a Qdrant - Google Drive`,
  avísale antes de crear otro.
