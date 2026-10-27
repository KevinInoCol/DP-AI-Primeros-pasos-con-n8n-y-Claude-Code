---
name: n8n-kommo-http
description: Construye cualquier integración con Kommo CRM en n8n usando el nodo HTTP Request, leyendo SIEMPRE la documentación oficial (developers.kommo.com) antes de armar el nodo para extraer el endpoint, método, parámetros y forma exacta del body. Prohíbe el nodo comunitario n8n-nodes-kommo. Use when the user asks for an n8n workflow, node, or automation that touches Kommo CRM — crear/mover/buscar leads, contactos, notas, tareas, custom fields, Salesbot, webhooks de Kommo — o cuando mencione "flujo de Kommo en n8n", "nodo HTTP para Kommo", "mueve el lead a la etapa", "consulta el lead desde n8n".
---

# Kommo CRM en n8n con nodos HTTP Request

En este proyecto **Kommo se integra siempre con el nodo HTTP Request de n8n**, nunca con el nodo de Kommo.

El nodo `n8n-nodes-kommo.kommo` es un **nodo comunitario** (autor `yatolstoy`, no verificado, no es core de n8n): cubre una fracción de la API, va por detrás de los cambios de Kommo y oculta el cuerpo real de la petición, así que cuando falla no se puede depurar contra la documentación. El HTTP Request expone exactamente lo que la doc describe.

**Y la doc se lee antes de escribir el nodo.** Kommo tiene trampas que no se adivinan: campos personalizados con forma propia según el tipo, `204` en respuestas vacías, límite de 7 req/s, endpoints que viven en `v2` en vez de `v4`. Inventar la forma del body produce un `400` genérico sin pista de qué campo estaba mal.

## Workflow obligatorio

### 1. Descompón la petición en operaciones de API

Antes de tocar n8n, escribe qué llamadas hacen falta. "Cuando llegue un formulario, crea el lead y avísame por Slack" son tres cosas, dos de ellas Kommo:

- buscar si el contacto ya existe (`GET /api/v4/contacts?query=...`)
- crear lead + contacto (`POST /api/v4/leads/complex`)

### 2. Lee la documentación oficial

El índice completo de la doc está en `llms.txt`, y **cualquier página admite `.md`** para obtener su markdown:

```bash
# Índice completo: qué página cubre qué recurso
curl -sL https://developers.kommo.com/llms.txt | grep -i "lead"

# La página concreta, en markdown
curl -sL https://developers.kommo.com/reference/adding-leads.md
```

Si ya sabes el recurso, salta el índice y ve directo a la página: el mapa recurso → URL está en [references/doc-map.md](references/doc-map.md).

De cada página extrae **tres cosas**:

1. **La ruta y el método exactos** — al final de cada página de referencia hay un bloque `# OpenAPI definition` con `paths`, `parameters` y el schema del request. Esa es la fuente de verdad, no la prosa de arriba.
2. **La forma del body**, incluidos los campos anidados (`_embedded`, `custom_fields_values`).
3. **Las limitaciones** que la propia página declara (máximo de entidades, parámetros `with` obligatorios, permisos).

```bash
# Extraer solo el contrato del endpoint
curl -sL https://developers.kommo.com/reference/adding-leads.md \
  | sed -n '/# OpenAPI definition/,$p'
```

### 3. Resume al usuario antes de construir

Tres o cuatro líneas, antes del JSON:

> Para mover el lead voy a usar `PATCH /api/v4/leads/{id}` (doc: `reference/updating-single-lead`) con body `{"status_id": N, "pipeline_id": M}`. Necesito el `status_id` real de tu cuenta — lo saco de `GET /api/v4/leads/pipelines`.

Si falta un ID de la cuenta del cliente (pipeline, etapa, campo, Salesbot), **no lo inventes**: pídelo o descúbrelo por API (`GET /api/v4/leads/pipelines` para pipelines y etapas, `GET /api/v4/leads/custom_fields` para campos).

### 4. Arma el nodo HTTP Request

Patrones listos para copiar — autenticación, batch, paginación, campos personalizados, Salesbot — en [references/n8n-http-patterns.md](references/n8n-http-patterns.md).

Base mínima:

```json
{
  "name": "Kommo · Actualizar lead",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [0, 0],
  "parameters": {
    "method": "PATCH",
    "url": "=https://{{ $vars.KOMMO_SUBDOMAIN }}.kommo.com/api/v4/leads/{{ $json.lead_id }}",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify({ status_id: 12345678 }) }}",
    "options": {}
  },
  "credentials": {
    "httpHeaderAuth": { "id": "REEMPLAZAR", "name": "Kommo token" }
  }
}
```

### 5. Verifica contra la lista de trampas

Antes de dar el flujo por terminado, repasa la tabla de abajo. La mayoría de los fallos de Kommo en n8n están ahí.

## Reglas duras

- **Nunca uses `n8n-nodes-kommo.kommo`.** Ni siquiera "para esta que es simple".
- **Nunca escribas un endpoint de memoria.** Verifica la ruta en la doc aunque estés seguro: Kommo mezcla `v4`, `v2` (Salesbot) y `amojo` (Chats API) con autenticación distinta.
- **El token nunca va en el JSON del workflow.** Va en una credencial *Header Auth* (`Authorization` / `Bearer <token>`). Un workflow exportado con el token dentro es una fuga.
- **Nunca inventes un ID numérico** de pipeline, etapa, campo o bot. Un ID equivocado no falla: escribe en el lugar equivocado y nadie se entera.
- **Si la doc no responde o no encuentras el endpoint**, párate y dilo. No improvises la ruta.

## Trampas de Kommo en n8n

| Síntoma | Causa | Solución |
|---|---|---|
| El nodo no saca nada y el flujo se corta | Kommo devuelve **`204 No Content`** cuando la consulta no encuentra entidades (no un array vacío) | Activa *Always Output Data* y comprueba con un IF antes de leer `_embedded` |
| `429`, y luego `403` persistente | Más de **7 req/s**; si insistes, Kommo bloquea la IP | `options.batching` con `batchSize: 1` y `batchInterval: 200` |
| `400` al escribir un campo personalizado | `custom_fields_values` mal formado, o campo `select` con texto que no coincide con ningún enum | Ver el bloque de custom fields en las referencias |
| `400` al crear varios registros | POST/PATCH de lote esperan **array** en la raíz, no objeto | `jsonBody: "={{ JSON.stringify([ {...}, {...} ]) }}"` |
| Solo llegan 250 registros | Es el máximo por página (y el máximo de creación/actualización por request) | Paginación con `_links.next.href`; para escritura, lotes de ≤50 |
| Un campo viene `null` que debería tener datos | Falta el parámetro `with` (`contacts`, `catalog_elements`, `source`…) | Añádelo en query params; la doc de cada GET lista los valores válidos |
| `{{ $json.body.leads }}` sale `undefined` en un webhook | El webhook de Kommo llega **`x-www-form-urlencoded`**, con claves planas tipo `leads[status][0][id]` | Bracket notation; ver [references/webhooks-n8n.md](references/webhooks-n8n.md) |
| El bot se responde a sí mismo en bucle | No se filtran los mensajes `outgoing` / `created_by != 0` | Filtro IF inmediatamente después del Webhook |
| `401` de golpe a las 24h | Se está usando un token OAuth (caduca) en vez del **long-lived token** de integración privada | Integración privada → token de larga duración |

## Referencias

- [references/doc-map.md](references/doc-map.md) — recurso → página de doc. Empieza aquí para no adivinar la URL.
- [references/n8n-http-patterns.md](references/n8n-http-patterns.md) — nodos HTTP Request listos: auth, paginación, throttling, custom fields, complex leads, notas, Salesbot.
- [references/webhooks-n8n.md](references/webhooks-n8n.md) — recibir webhooks de Kommo en n8n: parseo del form-urlencoded, filtros anti-bucle, seguridad.

> Existe un MCP comunitario de Kommo. Por ahora **no se usa**: la API directa cubre todo y no añade una dependencia sin mantenimiento garantizado. Si en algún momento aporta algo que la API no da, se reevalúa.
