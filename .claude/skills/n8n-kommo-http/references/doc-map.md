# Mapa: recurso → página de documentación

Toda URL admite `.md` para obtener el markdown:
`curl -sL https://developers.kommo.com/reference/adding-leads.md`

Índice completo y siempre actualizado: `curl -sL https://developers.kommo.com/llms.txt`
(úsalo si el recurso que buscas no está en esta tabla — la doc crece).

Base de todas las llamadas: `https://<subdominio>.kommo.com` — **nunca** `www.kommo.com`.

## Cuenta y autenticación

| Necesito | Página |
|---|---|
| Datos de la cuenta, `amojo_id` | `reference/account` |
| Parámetros extra de la cuenta | `reference/account-parameters` |
| Token de larga duración (integración privada) | `docs/long-lived-token`, `docs/private-integration` |
| OAuth 2.0 / refresh token | `reference/oauth20`, `reference/get-token` |
| Límites, rate limit, tamaños máximos | `docs/limitations` |
| Códigos HTTP y qué significan en Kommo | `docs/http-codes` |

## Leads

| Necesito | Página |
|---|---|
| Listar/filtrar leads | `reference/leads-list` |
| Un lead por ID | `reference/getting-a-lead-by-its-id` |
| Crear leads (lote) | `reference/adding-leads` |
| Actualizar varios leads | `reference/updating-leads` |
| Actualizar un lead (mover de etapa) | `reference/updating-single-lead` |
| Crear lead + contacto + empresa en una llamada | `reference/complex-leads` |
| Razones de pérdida | `reference/loss-reasons` |

## Pipelines y etapas

| Necesito | Página |
|---|---|
| Listar pipelines con sus etapas | `reference/pipelines-list` |
| Un pipeline por ID | `reference/get-pipeline-by-id` |
| Etapas de un pipeline | `reference/stages-list` |
| Crear / editar / borrar etapas | `reference/add-stages`, `reference/edit-stage`, `reference/deleting-stage` |

## Contactos y empresas

| Necesito | Página |
|---|---|
| Listar/buscar contactos | `reference/contacts-list` |
| Contacto por ID | `reference/get-contact` |
| Crear contactos | `reference/add-contacts` |
| Actualizar contacto(s) | `reference/update-contact`, `reference/update-contacts` |
| Empresas (mismo juego) | `reference/companies-list`, `reference/get-company`, `reference/add-companies`, `reference/update-company` |
| Buscar contacto por teléfono (receta) | `recipes/find-a-contact-by-a-phone-number` |

## Campos personalizados

| Necesito | Página |
|---|---|
| Listar campos de una entidad | `reference/custom-field-by-entity` |
| Campo por ID | `reference/custom-fields-by-id` |
| Crear / actualizar campos | `reference/add-custom-fields`, `reference/update-custom-fields` |
| Grupos de campos | `reference/list-of-entity-field-groups` |

## Notas, tareas, eventos, tags

| Necesito | Página |
|---|---|
| Notas de una entidad | `reference/notes-by-entity-id` |
| Notas por tipo de entidad | `reference/notes-list-entity` |
| Crear notas | `reference/add-notes` |
| Tipos de nota válidos | `reference/notes-types` |
| Tareas: listar / crear / editar | `reference/tasks-list`, `reference/add-tasks`, `reference/edit-task` |
| Eventos (historial) y su filtrado | `reference/events-list`, `reference/events-filtration`, `reference/events-types` |
| Tags: listar y asignar | `reference/list-of-entity-tags`, `reference/update-tags` |
| Vincular / desvincular entidades | `reference/linking-entities`, `reference/unlinking-entities` |

## Salesbot (API v2, no v4)

| Necesito | Página |
|---|---|
| Listar bots | `reference/salesbots-list` |
| Lanzar un bot sobre una entidad | `reference/launch-a-salesbot`, `reference/launch-salesbots` |
| Detener un bot | `reference/stop-salesbot` |

## Conversaciones y Chats API (autenticación distinta)

| Necesito | Página |
|---|---|
| Listar / cerrar conversaciones | `reference/get-talks`, `reference/close-conversation` |
| Mensajes de una conversación | `reference/get-conversation-messages`, `reference/send-message-to-conversation` |
| Guía completa de canal de chat propio | `reference/send-message-guide` |
| Registrar y conectar canal | `reference/register-channel`, `reference/connect-channel` |
| Enviar / importar mensajes | `reference/send-import-messages` |
| **Firma HMAC de las cabeceras** | `reference/chats-api-authorization-and-headers`, `recipes/calculate-headers-for-chats-api-requests` |
| Webhooks de chat | `reference/receiving-chat-webhooks` |

> La Chats API vive en `amojo.kommo.com`, usa `amojo_id` y firma **HMAC** en las cabeceras — **no** `Bearer`. Si el flujo toca mensajería propia, lee `reference/chats-api-authorization-and-headers` completa antes de armar nada.

## Webhooks

| Necesito | Página |
|---|---|
| Cómo funcionan y formato del payload | `docs/webhooks-general` |
| Alta / baja por API | `reference/add-webhooks`, `reference/delete-webhook`, `reference/list-webhooks` |
| Catálogo de eventos suscribibles | `reference/webhook-events` |

## Otros

| Necesito | Página |
|---|---|
| Listas / catálogos (productos) | `reference/get-lists`, `reference/get-list-elements`, `reference/add-list-elements` |
| Usuarios y roles | `reference/users-list`, `reference/user-roles-list` |
| Fuentes (sources) | `reference/get-sources`, `reference/add-sources` |
| Plantillas de mensaje / WhatsApp | `reference/get-templates`, `reference/submit-a-whatsapp-template-for-moderation` |
| Archivos (subida por sesiones) | `reference/files-methods`, `reference/create-session`, `reference/attached-to-entity` |
| Leads entrantes (incoming leads) | `reference/incoming-leads-list`, `reference/accepting-incoming-leads` |

## Recetas oficiales útiles

Van directas al caso de uso, con el body ya montado:

- `recipes/move-a-lead-to-another-stage`
- `recipes/create-a-lead-with-a-contact-with-duplicate-control-1`
- `recipes/creating-a-contact-with-a-phone-number-and-email`
- `recipes/find-a-contact-by-a-phone-number`
- `recipes/getting-leads-with-pagination`
- `recipes/add-a-lead-with-utms`
- `recipes/lead-with-tag`
- `recipes/creating-a-product-and-adding-it-to-a-lead`
- `recipes/account-with-amojo_id`
