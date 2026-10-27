---
name: n8n-install-community-mcp
description: >-
  PASO 0 del taller de n8n. Instala en el Claude Code del alumno el MCP de
  comunidad `n8n-mcp` (de czlonkowski) para que Claude pueda diseñar y subir
  workflows a la instancia de n8n del alumno. Verifica Node/npx, pide la URL y
  la API Key de n8n, registra el MCP (modo completo o solo documentación),
  precalienta el paquete con npx y comprueba la conexión. Úsala cuando el
  alumno diga "instalar el MCP de n8n", "conectar Claude Code con n8n",
  "no aparecen las tools de n8n", "n8n-mcp Failed to connect", "paso 0", o
  ejecute /n8n-install-community-mcp.
---

# Skill 0 — Instalar el MCP de comunidad de n8n (`n8n-mcp`)

Esta skill es el **paso previo** a todas las demás del taller (`n8n-01` … `n8n-05`).
Instala el MCP `n8n-mcp` (comunidad, de czlonkowski) en el Claude Code del alumno.
Es el mismo procedimiento que `INSTALAR-N8N-MCP.md`, en formato skill.

Sigue los pasos **en orden** y no te saltes ninguno.

## Paso 1 — Verificar requisitos

```bash
node --version && npx --version
```

Si `node` no existe, dile al alumno que instale **Node.js 18+** desde https://nodejs.org
y que vuelva a intentarlo. **No continúes.**

## Paso 2 — Pedir los datos al alumno

Pregúntale estos **2 datos** (no continúes hasta tenerlos):

1. **URL de su n8n**, por ejemplo `https://n8n.midominio.com` (sin `/` al final).
2. **API Key de n8n**: en n8n → **Settings → n8n API → Create an API key**. Es un texto largo.

> Si el alumno aún no tiene instancia o API key, instala igualmente en **modo documentación**
> (salta el Paso 3 y usa el Paso 3-B). Podrá diseñar workflows, pero no subirlos automáticamente.

La API key es un secreto: úsala solo dentro del comando del Paso 3. **No la repitas en tus
respuestas ni la guardes en archivos del proyecto.**

## Paso 3 — Instalar el MCP (modo completo, con la instancia del alumno)

Sustituye `URL_DEL_ALUMNO` y `API_KEY_DEL_ALUMNO` por lo que te dio y ejecuta:

```bash
claude mcp remove n8n-mcp 2>/dev/null; \
claude mcp add n8n-mcp \
  --env MCP_MODE=stdio \
  --env LOG_LEVEL=error \
  --env DISABLE_CONSOLE_OUTPUT=true \
  --env N8N_API_URL=URL_DEL_ALUMNO \
  --env N8N_API_KEY=API_KEY_DEL_ALUMNO \
  -- npx -y n8n-mcp
```

## Paso 3-B — Instalar SOLO en modo documentación (si no hay API key)

```bash
claude mcp remove n8n-mcp 2>/dev/null; \
claude mcp add n8n-mcp \
  --env MCP_MODE=stdio \
  --env LOG_LEVEL=error \
  --env DISABLE_CONSOLE_OUTPUT=true \
  -- npx -y n8n-mcp
```

## Paso 4 — "Calentar" el paquete (evita el falso error la primera vez)

La primera vez npx descarga el paquete y el chequeo puede marcar "Failed to connect".
Fuerza la descarga:

```bash
echo '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"t","version":"1"}}}' | MCP_MODE=stdio npx -y n8n-mcp 2>&1 | head -3
```

Debe aparecer una línea con `"name":"n8n-documentation-mcp"`. Si sale eso, todo bien.

## Paso 5 — Verificar conexión

```bash
claude mcp list 2>&1 | grep -i n8n
```

Debe decir `n8n-mcp: ... - ✔ Connected`. Si dice "Failed", repite el Paso 4 y vuelve a verificar.

## Paso 6 — Avisar al alumno (importante)

Dile al alumno textualmente:

> ✅ Listo. **Cierra y vuelve a abrir Claude Code** (o reinicia la sesión) para que cargue el MCP de n8n.
> Después ya puedes pedirme cosas como: *"créame un workflow que reciba un webhook y mande un email"*
> y lo diseñaré (y si diste API key, lo subo directo a tu n8n).
> El siguiente paso del taller es `/n8n-01-crear-flujo-base`.

## Notas

- El MCP es `n8n-mcp` (comunidad, de czlonkowski). Da a Claude el conocimiento de los 500+ nodos
  de n8n y, con API key, permite crear/actualizar/activar workflows en la instancia del alumno.
- Requiere **n8n ≥ 1.88** aprox. para la parte de la API de gestión de workflows.
- La API key es un secreto personal de cada alumno: **no se comparte ni se sube a ningún repositorio**.
