---
name: n8n-02-agregar-memoria-postgres
description: >-
  PASO 2 del taller de n8n. Añade MEMORIA con Postgres al flujo base creado en el
  paso 1: agrega el nodo "Postgres Chat Memory" y lo conecta al AI Agent para que
  el agente recuerde la conversación. NO maneja credenciales (la credencial de
  Postgres la pone el alumno a mano después). Úsala cuando el alumno diga "agregar
  memoria", "memoria con postgres", "paso 2", o ejecute
  /n8n-02-agregar-memoria-postgres. Requiere haber ejecutado antes
  n8n-01-crear-flujo-base.
---

# Skill 2 — Añadir memoria con Postgres al agente

Esta es la **segunda skill** del taller. Parte del flujo base creado en el paso 1
y le añade **memoria persistente** con Postgres, para que el agente recuerde los
mensajes anteriores de cada conversación.

El flujo quedará así:

```
Webhook (POST) ──main──▶ AI Agent
                            ▲  ▲
        ai_languageModel ───┘  └─── ai_memory
        OpenAI Chat Model        Postgres Chat Memory   ← nodo nuevo de este paso
```

## Regla de oro sobre credenciales (MUY IMPORTANTE)

> ⚠️ **NUNCA** pidas, recibas ni escribas la credencial de Postgres del alumno
> (host, usuario, contraseña, base de datos). Son datos sensibles y los coloca el
> alumno **manualmente** en la interfaz de n8n. El profesor lo explicará en clase.
>
> Por eso el nodo *Postgres Chat Memory* se crea **sin** bloque `credentials`.

## Pasos que debes seguir

### 1. Verifica la conexión con n8n

Llama a `n8n_health_check` (mode `status`). Si falla, detente y dile al alumno
que revise su `.mcp.json` y reinicie Claude Code.

### 2. Localiza el flujo base del paso 1

El flujo de cada alumno tiene un **ID distinto**, así que NO lo asumas. Búscalo:

1. Llama a `n8n_list_workflows` y busca el que se llame
   **`Agente Básico con conexión por Webhook`**.
2. Guarda su `id`.

Si **no existe**, detente y dile al alumno que primero ejecute la skill
`n8n-01-crear-flujo-base` (paso 1).

Si aparece **más de uno** con ese nombre, pregúntale al alumno cuál usar antes de
continuar.

### 3. Añade el nodo de memoria y su conexión

Usa `n8n_update_partial_workflow` con el `id` encontrado y estas dos operaciones
(edición sobre el MISMO flujo, no crees uno nuevo):

```json
{
  "id": "<ID_DEL_FLUJO>",
  "operations": [
    {
      "type": "addNode",
      "node": {
        "id": "d4d4c444-4444-4444-8444-444444444444",
        "name": "Postgres Chat Memory",
        "type": "@n8n/n8n-nodes-langchain.memoryPostgresChat",
        "typeVersion": 1.4,
        "position": [740, 540],
        "parameters": {
          "sessionIdType": "customKey",
          "sessionKey": "={{ $json.body.sessionId }}",
          "tableName": "n8n_chat_histories"
        }
      }
    },
    {
      "type": "addConnection",
      "source": "Postgres Chat Memory",
      "sourceOutput": "ai_memory",
      "target": "AI Agent",
      "targetInput": "ai_memory"
    }
  ]
}
```

Notas sobre la configuración:
- **`sessionIdType: customKey`** con **`sessionKey: {{ $json.body.sessionId }}`**:
  como el disparador es un Webhook (no un Chat Trigger), la sesión se identifica
  con un `sessionId` que llega en el cuerpo de la petición. Así cada conversación
  tiene su propia memoria.
- **`tableName: n8n_chat_histories`**: si la tabla no existe, n8n la crea sola la
  primera vez.

### 4. Valida el flujo

Llama a `n8n_validate_workflow` con el `id`. Confirma `valid: true`. Es normal que
sigan apareciendo *sugerencias* (system message, tools): se abordan en pasos
posteriores.

### 5. Informa al alumno

Muestra un resumen corto y amable:

- ✅ Memoria añadida: nodo **Postgres Chat Memory** conectado al **AI Agent**.
- 📋 **Qué falta (lo hace el alumno a mano en la UI de n8n):**
  1. Abrir el nodo **Postgres Chat Memory** y asignar/crear **su** credencial de
     **Postgres** (host, base de datos, usuario, contraseña). *El profesor
     explicará este paso.*
  2. Asegurarse de que el nodo **OpenAI Chat Model** ya tenga su credencial (del
     paso 1).
- 🧪 **Cómo probar la memoria** una vez con credenciales y flujo activo: enviar dos
  peticiones seguidas con el **mismo `sessionId`** y comprobar que recuerda:
  - `POST <URL-de-su-n8n>/webhook/agente-basico`
  - Body 1: `{ "sessionId": "cliente-123", "message": "Me llamo Ana" }`
  - Body 2: `{ "sessionId": "cliente-123", "message": "¿Cómo me llamo?" }`
  - El agente debería responder "Ana" en el segundo mensaje.

## Qué NO debes hacer en esta skill

- ❌ No crees un flujo nuevo: edita el del paso 1 con `n8n_update_partial_workflow`.
- ❌ No añadas system message ni herramientas (tools): eso viene en pasos siguientes.
- ❌ No actives el flujo automáticamente.
- ❌ No crees ni modifiques credenciales (ni de Postgres ni de OpenAI).
- ❌ No dupliques el nodo de memoria si el flujo ya tiene uno; si ya existe,
  avísale al alumno y no lo agregues otra vez.
