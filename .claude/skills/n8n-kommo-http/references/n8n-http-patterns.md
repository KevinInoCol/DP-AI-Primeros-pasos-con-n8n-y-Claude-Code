# Patrones de nodo HTTP Request para Kommo

Todos los ejemplos usan `n8n-nodes-base.httpRequest` con `typeVersion: 4.2`.
Sustituye `SUBDOMINIO` y los IDs numéricos por los reales de la cuenta (descúbrelos, no los inventes).

## Autenticación

Kommo espera `Authorization: Bearer <long-lived token>`. En n8n se monta **una sola vez** como credencial genérica y se reutiliza en todos los nodos:

**Credentials → Header Auth →** Name: `Authorization`, Value: `Bearer eyJ0eXAi...`

En el nodo:

```json
"authentication": "genericCredentialType",
"genericAuthType": "httpHeaderAuth"
```

Nunca pongas el token en `headerParameters` del nodo: se exporta con el workflow.

Para el subdominio, una variable de instancia (`$vars.KOMMO_SUBDOMAIN`) o un nodo *Set* de configuración al inicio del flujo. Así el mismo flujo sirve para otra cuenta sin tocar cada nodo.

## GET — buscar un contacto por teléfono

`query` busca en todos los campos del contacto, incluidos teléfono y email.

```json
{
  "name": "Kommo · Buscar contacto",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [0, 0],
  "parameters": {
    "method": "GET",
    "url": "=https://SUBDOMINIO.kommo.com/api/v4/contacts",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendQuery": true,
    "queryParameters": {
      "parameters": [
        { "name": "query", "value": "={{ $json.telefono }}" },
        { "name": "limit", "value": "1" }
      ]
    },
    "options": {}
  },
  "alwaysOutputData": true
}
```

**`alwaysOutputData: true` no es opcional aquí.** Si no hay coincidencias Kommo responde `204 No Content` con cuerpo vacío; sin esa bandera el nodo no emite ítems y el flujo se corta en silencio en vez de seguir por la rama de "no existe".

El IF que sigue:

```
{{ $json._embedded?.contacts?.length > 0 }}
```

## POST — crear leads (lote)

Los endpoints de creación/actualización esperan un **array en la raíz**, incluso para un solo elemento.

```json
{
  "name": "Kommo · Crear lead",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [0, 0],
  "parameters": {
    "method": "POST",
    "url": "=https://SUBDOMINIO.kommo.com/api/v4/leads",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify([{ name: $json.nombre, price: 0, pipeline_id: 1234567, status_id: 12345678, custom_fields_values: [{ field_id: 987654, values: [{ value: $json.comentario }] }] }]) }}",
    "options": {}
  }
}
```

Usar `JSON.stringify(...)` sobre un objeto JS —en vez de escribir el JSON a mano con `{{ }}` interpolados dentro— evita que un apóstrofo o un salto de línea del dato rompa el JSON.

Máximo 250 entidades por request; **la doc recomienda ≤50** para no comerse un `504`.

## PATCH — mover un lead de etapa

```json
{
  "name": "Kommo · Mover etapa",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [0, 0],
  "parameters": {
    "method": "PATCH",
    "url": "=https://SUBDOMINIO.kommo.com/api/v4/leads/{{ $json.lead_id }}",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify({ status_id: 12345678, pipeline_id: 1234567 }) }}",
    "options": {}
  }
}
```

El PATCH de **un** lead (`/leads/{id}`) lleva objeto; el de **varios** (`/leads`) lleva array con el `id` dentro de cada elemento.

`142` (Cerrado ganado) y `143` (Cerrado perdido) son etapas del sistema presentes en todos los pipelines. Al mover a `143` se puede mandar `loss_reason_id`.

## Custom fields — la forma cambia según el tipo

```jsonc
"custom_fields_values": [
  { "field_id": 111, "values": [{ "value": "texto libre" }] },          // text / textarea
  { "field_id": 222, "values": [{ "value": 1500 }] },                    // numeric → número, no string
  { "field_id": 333, "values": [{ "value": true }] },                    // checkbox
  { "field_id": 444, "values": [{ "enum_id": 5550001 }] },               // select → por enum_id (robusto)
  { "field_id": 444, "values": [{ "value": "Opción exacta" }] },         // select → por texto EXACTO
  { "field_id": 555, "values": [{ "value": "+51999888777", "enum_code": "WORK" }] } // teléfono/email
]
```

Reglas:

- **`select` por `enum_id` siempre que puedas.** Por texto, un acento o una mayúscula de diferencia devuelve `400`.
- Un `select` que vaya a rellenar un LLM es un problema: parafrasea. Mejor pedir al usuario que el campo sea de texto.
- `numeric` con string → `400`.
- Teléfonos y emails en contactos son multivalor: `enum_code` es `WORK`, `MOB`, `HOME`, `OTHER`, `WORKDD`, `PRIV`.
- Los IDs de campo salen de `GET /api/v4/leads/custom_fields` (o `/contacts/custom_fields`), nunca de la intuición.

## POST — lead + contacto en una sola llamada

`/api/v4/leads/complex` crea todo junto y devuelve los IDs generados. Evita la carrera de "crear contacto, esperar ID, crear lead".

```json
"jsonBody": "={{ JSON.stringify([{ name: 'Lead web — ' + $json.nombre, pipeline_id: 1234567, status_id: 12345678, _embedded: { contacts: [{ first_name: $json.nombre, custom_fields_values: [{ field_id: 555, values: [{ value: $json.telefono, enum_code: 'WORK' }] }] }] } }]) }}"
```

Límite: 40 valores de campo personalizado por entidad añadida.

## Paginación — traer todos los leads

Kommo devuelve `_links.next.href` mientras queden páginas, y lo omite en la última. El propio nodo pagina:

```json
"parameters": {
  "method": "GET",
  "url": "=https://SUBDOMINIO.kommo.com/api/v4/leads",
  "authentication": "genericCredentialType",
  "genericAuthType": "httpHeaderAuth",
  "sendQuery": true,
  "queryParameters": {
    "parameters": [
      { "name": "limit", "value": "250" },
      { "name": "with", "value": "contacts" }
    ]
  },
  "options": {
    "pagination": {
      "pagination": {
        "paginationMode": "responseContainsNextURL",
        "nextURL": "={{ $response.body._links.next.href }}",
        "paginationCompleteWhen": "other",
        "completeExpression": "={{ $response.body._links.next === undefined }}"
      }
    }
  }
}
```

`limit` máximo 250. Si la primera página ya viene `204`, no hay `body` y `completeExpression` corta sola.

## Throttling — 7 req/s y bloqueo de IP

Superar 7 req/s devuelve `429`; insistir **bloquea la IP** con `403` para toda la cuenta. Cuando un nodo HTTP recibe muchos ítems del nodo anterior, hay que frenarlo:

```json
"options": {
  "batching": {
    "batch": { "batchSize": 1, "batchInterval": 200 }
  }
}
```

`batchSize: 1` + `batchInterval: 200` ≈ 5 req/s, con margen. Para cargas grandes, mejor agrupar de a 50 en el body (Kommo acepta lote) que mandar 50 requests.

Complementa con reintento en el propio nodo:

```json
"retryOnFail": true,
"maxTries": 3,
"waitBetweenTries": 2000
```

## Salesbot — API v2, no v4

```json
{
  "name": "Kommo · Lanzar Salesbot",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [0, 0],
  "parameters": {
    "method": "POST",
    "url": "=https://SUBDOMINIO.kommo.com/api/v2/salesbot/run",
    "authentication": "genericCredentialType",
    "genericAuthType": "httpHeaderAuth",
    "sendBody": true,
    "specifyBody": "json",
    "jsonBody": "={{ JSON.stringify([{ bot_id: 123456, entity_id: $json.lead_id, entity_type: 2 }]) }}",
    "options": {}
  }
}
```

`entity_type: 2` = lead. Éxito es `200` **o** `202`.

Patrón habitual del bot con IA: el flujo **no** envía el mensaje; escribe la respuesta en un custom field del lead con un PATCH y después lanza el Salesbot, que lee ese campo y lo manda al canal. **Encadena en ese orden y comprueba que el PATCH devolvió 200 antes de lanzar el bot**, o el cliente recibirá la respuesta anterior.

## Notas — dejar rastro en el lead

```json
"url": "=https://SUBDOMINIO.kommo.com/api/v4/leads/{{ $json.lead_id }}/notes",
"jsonBody": "={{ JSON.stringify([{ note_type: 'common', params: { text: $json.resumen } }]) }}"
```

Tipos válidos en `reference/notes-types` (`common`, `call_in`, `call_out`, `service_message`…). Un `note_type` inventado devuelve `400`.

## Diagnóstico cuando algo devuelve 400

Kommo describe el campo culpable en el cuerpo del error, pero n8n por defecto solo muestra el status. Para verlo:

```json
"options": {
  "response": {
    "response": { "fullResponse": true, "neverError": true }
  }
}
```

Con `neverError` el nodo no aborta y puedes inspeccionar `statusCode` y `body` (que trae `validation-errors` con el `field_name` exacto). Quítalo cuando el flujo ya funcione: enmascara fallos reales en producción.
