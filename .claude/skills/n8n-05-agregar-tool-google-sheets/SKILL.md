---
name: n8n-05-agregar-tool-google-sheets
description: >-
  PASO 5 del taller de n8n. Da al agente una HERRAMIENTA (tool) para extraer
  información de tablas de Google Sheets: añade el nodo Google Sheets (operación
  "Get Row(s)") conectado al AI Agent por ai_tool, para que el agente pueda leer
  filas de una hoja (p. ej. un listado de propiedades) y responder con esos datos.
  NO maneja credenciales (Google Sheets la pone el alumno a mano). Úsala cuando el
  alumno diga "tool de google sheets", "consultar tablas", "leer una hoja",
  "paso 5", o ejecute /n8n-05-agregar-tool-google-sheets. Requiere haber ejecutado
  antes n8n-01, n8n-02 y n8n-03.
---

# Skill 5 — Añadir un tool de Google Sheets al agente

Esta es la **quinta skill** del taller. Le da al agente una **herramienta** para
leer datos de una tabla de **Google Sheets** (por ejemplo, un listado de
propiedades de la inmobiliaria con precio, zona, habitaciones, estado). Cuando el
cliente pregunta por datos tabulares, el agente consulta la hoja y responde.

El flujo del agente quedará así (se añade una herramienta más al AI Agent):

```
Webhook ─main─▶ AI Agent
                 ▲  ▲  ▲  ▲
    ai_languageModel │  │  └─ ai_tool ── Consultar tabla (Google Sheets)
    ai_memory ───────┘  └─ ai_tool ── Qdrant Vector Store (del paso 3)
```

## Regla de oro sobre credenciales (MUY IMPORTANTE)

> ⚠️ **NUNCA** pidas, recibas ni escribas la credencial de Google del alumno. La
> credencial de **Google Sheets (OAuth2)** la coloca el alumno **manualmente** en
> la UI, y también elige a mano su **documento** y su **hoja**. El profesor lo
> explicará en clase. El nodo se crea **sin** bloque `credentials`.

## Nota técnica: no existe un nodo "Google Sheets Tool" separado

En las versiones modernas de n8n **cualquier nodo regular puede usarse como
herramienta** del AI Agent (capacidad `usableAsTool`). Por eso se usa el nodo base
**Google Sheets** (`n8n-nodes-base.googleSheets`) conectado por `ai_tool`, no un
nodo `*Tool` aparte.

## Pasos que debes seguir

### 1. Verifica la conexión con n8n

Llama a `n8n_health_check` (mode `status`). Si falla, detente y avisa al alumno.

### 2. Localiza el flujo del agente

El flujo de cada alumno tiene un **ID distinto**, así que NO lo asumas:

1. Llama a `n8n_list_workflows` y busca el que se llame
   **`Agente Básico con conexión por Webhook`**.
2. Guarda su `id`.

Si no existe, dile que ejecute primero las skills previas (01, 02 y 03). Si hay
más de uno, pregúntale cuál usar.

### 3. Añade el tool de Google Sheets y su conexión

Usa `n8n_update_partial_workflow` con el `id` y estas operaciones (edición sobre el
MISMO flujo, no crees uno nuevo):

```json
{
  "id": "<ID_DEL_FLUJO>",
  "operations": [
    {
      "type": "addNode",
      "node": {
        "id": "77777777-7777-4777-8777-777777777777",
        "name": "Consultar tabla (Google Sheets)",
        "type": "n8n-nodes-base.googleSheets",
        "typeVersion": 4.7,
        "position": [1180, 320],
        "parameters": {
          "resource": "sheet",
          "operation": "read",
          "documentId": {
            "__rl": true,
            "mode": "list",
            "value": "",
            "cachedResultName": "Selecciona tu Google Sheet en la UI"
          },
          "sheetName": {
            "__rl": true,
            "mode": "list",
            "value": "",
            "cachedResultName": "Selecciona la hoja en la UI"
          },
          "options": {}
        }
      }
    },
    {
      "type": "addConnection",
      "source": "Consultar tabla (Google Sheets)",
      "sourceOutput": "ai_tool",
      "target": "AI Agent",
      "targetInput": "ai_tool"
    }
  ]
}
```

Notas sobre la configuración:
- **`operation: read` (Get Row(s))**: la herramienta devuelve las filas de la hoja
  para que el agente razone sobre ellas. Es la forma más simple y fiable para el
  taller. (Más adelante se puede filtrar por columnas usando expresiones `$fromAI`.)
- **`documentId` y `sheetName`** quedan vacíos a propósito: el alumno selecciona su
  documento y su hoja en la UI.
- El **nombre** del nodo ("Consultar tabla (Google Sheets)") es importante: el
  agente lo usa para decidir cuándo invocar la herramienta. Puede ajustarse al
  negocio (p. ej. "Consultar propiedades disponibles").

### 4. Valida el flujo

Llama a `n8n_validate_workflow` con el `id`.

> ℹ️ **FALSO POSITIVO ESPERADO** (igual que con Qdrant en el paso 3). El validador
> dirá que *"Consultar tabla (Google Sheets) cannot output ai_tool connections"* y
> marcará `valid: false`. **Es un límite del validador, NO un error real.** No
> reconoce la capacidad `usableAsTool` de los nodos regulares. En n8n real la
> conexión funciona y sobrevive al guardado en la UI.
>
> Confírmalo con `n8n_get_workflow` (mode `structure`): si existe la conexión
> `Consultar tabla (Google Sheets) --ai_tool--> AI Agent`, el paso está BIEN.
> Tranquiliza al alumno sobre este mensaje.

### 5. Informa al alumno

Muestra un resumen corto y amable:

- ✅ Tool añadido: **Consultar tabla (Google Sheets)** conectado al **AI Agent**.
- 📋 **Qué falta (lo hace el alumno a mano en la UI de n8n):**
  1. Abrir el nodo y asignar/crear **su** credencial de **Google Sheets (OAuth2)**.
  2. Seleccionar su **documento** (Google Sheet) y la **hoja** concreta.
  3. Opcional: renombrar el nodo y ajustar el rango si su tabla no está en `A:F`.
  *El profesor explicará estos pasos.*
- 🧪 **Cómo probarlo** (con credenciales y flujo activo): preguntar por algo que
  esté en la tabla, p. ej.:
  - `POST <URL-de-su-n8n>/webhook/agente-basico`
  - Body: `{ "sessionId": "cliente-123", "message": "¿Qué pisos tienes en la zona centro por menos de 200.000?" }`
  - El agente debería consultar la hoja y responder con las filas que encajan.

## Qué NO debes hacer en esta skill

- ❌ No crees un flujo nuevo: edita el del agente con `n8n_update_partial_workflow`.
- ❌ No actives el flujo automáticamente.
- ❌ No crees ni modifiques credenciales de Google.
- ❌ No rellenes `documentId` ni `sheetName`: los elige el alumno en la UI.
- ❌ No dupliques el nodo si ya existe; si el flujo ya tiene el tool de Sheets,
  avísale al alumno y no lo agregues otra vez.
