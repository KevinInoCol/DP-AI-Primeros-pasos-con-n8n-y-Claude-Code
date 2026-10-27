# Recibir webhooks de Kommo en n8n

## Configuración en Kommo

**Ajustes → Integraciones → tu integración → Webhooks**, o por API (`reference/add-webhooks`).
Catálogo de eventos suscribibles: `reference/webhook-events`. Máximo 100 webhooks por cuenta.

Kommo considera el envío aceptado si la respuesta está entre **100 y 299**. El nodo Webhook de n8n responde 200 por defecto; si el flujo es lento, pon *Respond: Immediately* para que Kommo no reintente.

## El payload llega form-urlencoded, no JSON

Esta es la causa número uno de "el webhook llega pero `$json` está vacío".

Kommo envía `Content-Type: application/x-www-form-urlencoded` con las claves **aplanadas entre corchetes**:

```
leads[status][0][id]=1234567
leads[status][0][status_id]=12345678
leads[status][0][pipeline_id]=1234567
account[subdomain]=miempresa
```

n8n **no** reconstruye el objeto anidado. Llegan como claves literales:

```js
// ❌ undefined
{{ $json.body.leads.status[0].id }}

// ✅
{{ $json.body['leads[status][0][id]'] }}
```

Si el flujo va a tocar varios campos, normaliza una vez en un nodo Code y trabaja limpio a partir de ahí:

```js
// Code node — reconstruye el objeto anidado desde las claves con corchetes
const out = {};
for (const [k, v] of Object.entries($input.first().json.body ?? {})) {
  const path = k.replace(/\]/g, '').split('[');
  let cur = out;
  path.forEach((seg, i) => {
    if (i === path.length - 1) cur[seg] = v;
    else cur = cur[seg] ??= (/^\d+$/.test(path[i + 1]) ? [] : {});
  });
}
return [{ json: out }];
```

Después: `{{ $json.leads.status[0].id }}`.

## Forma del payload por tipo de evento

```jsonc
{ "leads":    { "add": [ { "id": "..." } ] } }      // creación
{ "leads":    { "update": [ { "id": "..." } ] } }   // actualización
{ "leads":    { "status": [ { "id": "...", "status_id": "...", "pipeline_id": "..." } ] } }
{ "leads":    { "delete": [ { "id": "..." } ] } }
{ "contacts": { "add": [ ... ] } }
{ "message":  { "add": [ ... ] } }                   // mensaje entrante
```

Siempre `entidad → acción → array`. **Itera el array**: un webhook puede traer varias entidades en un mismo POST.

Los valores llegan como **strings**, incluidos los IDs. Si comparas con un ID numérico, convierte (`Number(...)`) o compara con `==`.

## Mensajes entrantes: filtrar antes de hacer nada

Para un bot conversacional, las claves relevantes:

| Clave | Contenido |
|---|---|
| `message[add][0][text]` | El texto del mensaje |
| `message[add][0][element_id]` | ID del **lead** |
| `message[add][0][type]` | `incoming` / `outgoing` |
| `message[add][0][created_by]` | `0` = lo escribió el cliente |
| `message[add][0][author][id]` | ID del **contacto** (estable por persona) |

**Filtra `outgoing` y `created_by != 0` en un IF inmediatamente después del Webhook.** Sin ese filtro, la respuesta que el flujo envía dispara otro webhook y el bot se contesta a sí mismo en bucle — consumiendo tokens y rate limit hasta que Kommo bloquea la IP.

## Clave de conversación: verifica antes de diseñar la memoria

Si el flujo mantiene contexto por conversación, comprueba en la cuenta real qué pasa cuando la misma persona escribe dos veces con días de diferencia. Algunas configuraciones de Kommo **crean un lead nuevo por consulta**: el `element_id` cambia y la memoria se reinicia aunque sea el mismo cliente.

Si es el caso, la clave debe ser `author[id]` (el contacto), no `element_id`.

## Seguridad: Kommo no firma sus webhooks

No hay HMAC ni secreto compartido, así que el endpoint **no puede verificar el origen**. Cualquiera que conozca la URL puede inyectar mensajes falsos, escribir en leads arbitrarios y quemar tokens del LLM.

Mitigación en n8n:

- Path largo y aleatorio en el nodo Webhook (no `/kommo`).
- *Header Auth* en el nodo Webhook con un secreto propio, **si** la integración permite añadir cabeceras (los webhooks nativos de Kommo no; entonces queda el path secreto).
- Allowlist de las IPs de Kommo en el reverse proxy, si hay uno delante.
- Validar que el `account[subdomain]` del payload es el esperado antes de actuar.

## Desarrollo local

La URL de test de n8n solo escucha mientras el editor está abierto con *Listen for test event* activo. Para pruebas reales, activa el workflow y usa la URL de producción.

Con n8n local + ngrok, cada reinicio de ngrok cambia el dominio y hay que volver a registrar el webhook en Kommo — es la causa habitual de "dejó de llegar de un día para otro".
