---
name: n8n-01-crear-flujo-base
description: >-
  PASO 1 del taller de n8n. Crea en la instancia n8n del alumno el flujo base de
  un agente de IA conectado por webhook: Webhook (POST) -> AI Agent -> OpenAI
  Chat Model. NO maneja credenciales (las pone el alumno a mano después). Úsala
  cuando el alumno diga "crear flujo base", "empezar el taller", "paso 1", o
  ejecute /n8n-01-crear-flujo-base.
---

# Skill 1 — Crear el flujo base del agente (Webhook + AI Agent + OpenAI)

Esta es la **primera skill** del taller. Su único objetivo es crear en la
instancia de n8n del alumno el **flujo base** de un agente de IA.

El flujo tiene 3 nodos:

```
Webhook (POST /agente-basico)  ──main──▶  AI Agent  ◀──ai_languageModel──  OpenAI Chat Model
```

## Regla de oro sobre credenciales (MUY IMPORTANTE)

> ⚠️ **NUNCA** pidas, recibas ni escribas la API key de OpenAI del alumno.
> Las credenciales son datos sensibles y las coloca el alumno **manualmente**
> en la interfaz de n8n. El profesor lo explicará en clase.
>
> Por eso el nodo *OpenAI Chat Model* se crea **sin** bloque `credentials`.

## Pasos que debes seguir

### 1. Verifica la conexión con n8n

Llama a `n8n_health_check` (mode `status`). Si falla, detente y dile al alumno
que revise su archivo `.mcp.json` y reinicie Claude Code — no continúes.

### 2. Crea el flujo

Llama a `n8n_create_workflow` con **exactamente** estos datos:

- **name:** `Agente Básico con conexión por Webhook`
- **settings:** `{ "executionOrder": "v1" }`
- **nodes:**

```json
[
  {
    "id": "b1a1c111-1111-4111-8111-111111111111",
    "name": "Webhook",
    "type": "n8n-nodes-base.webhook",
    "typeVersion": 2,
    "position": [260, 320],
    "webhookId": "agente-basico-webhook",
    "parameters": {
      "httpMethod": "POST",
      "path": "agente-basico",
      "responseMode": "lastNode",
      "options": {}
    }
  },
  {
    "id": "a2a2c222-2222-4222-8222-222222222222",
    "name": "AI Agent",
    "type": "@n8n/n8n-nodes-langchain.agent",
    "typeVersion": 3.1,
    "position": [560, 320],
    "parameters": {
      "promptType": "define",
      "text": "={{ $json.body.message }}",
      "options": {}
    }
  },
  {
    "id": "c3c3c333-3333-4333-8333-333333333333",
    "name": "OpenAI Chat Model",
    "type": "@n8n/n8n-nodes-langchain.lmChatOpenAi",
    "typeVersion": 1.2,
    "position": [480, 540],
    "parameters": {
      "model": {
        "__rl": true,
        "value": "gpt-4.1",
        "mode": "list",
        "cachedResultName": "gpt-4.1"
      },
      "options": {}
    }
  }
]
```

- **connections:**

```json
{
  "Webhook": {
    "main": [[{ "node": "AI Agent", "type": "main", "index": 0 }]]
  },
  "OpenAI Chat Model": {
    "ai_languageModel": [[{ "node": "AI Agent", "type": "ai_languageModel", "index": 0 }]]
  }
}
```

### 3. Valida el flujo

Llama a `n8n_validate_workflow` con el ID devuelto. Confirma que
`valid: true` y que no haya errores. (Es normal ver *sugerencias* sobre añadir
un system message o herramientas: eso lo haremos en skills siguientes.)

### 4. Informa al alumno

Muestra un resumen corto y amable:

- ✅ Flujo creado, con su **nombre** e **ID**.
- 📋 **Qué falta (lo hace el alumno a mano en la UI de n8n):**
  1. Abrir el nodo **OpenAI Chat Model** y asignar/crear **su** credencial de
     OpenAI (pegando su propia API key). *El profesor explicará este paso.*
  2. **Activar** el flujo con el toggle de arriba a la derecha.
- 🧪 **Cómo probarlo** una vez activo y con credencial:
  - Método: `POST`
  - URL: `<URL-de-su-n8n>/webhook/agente-basico`
  - Body de ejemplo: `{ "message": "Hola, ¿quién eres?" }`

## Qué NO debes hacer en esta skill

- ❌ No añadas system message, herramientas (tools) ni memoria: eso viene en
  skills posteriores del taller.
- ❌ No actives el flujo automáticamente (lo hace el alumno tras poner su credencial).
- ❌ No crees ni modifiques credenciales.
- ❌ No crees un flujo distinto si ya existe; si el alumno repite la skill,
  avísale que ya tiene uno con ese nombre y pregúntale si quiere otro.
