---
name: bankinter-workflow
description: Guía completa de este proyecto AEM + opencode. Explica los dos pipelines (PDF→DAM y WebPage→ContentFragment+XF), los agentes, los commands disponibles, la estructura de carpetas context/, y los gotchas conocidos (Cloudflare, MCP model creation). Usa esta skill cuando el usuario pregunte cómo funciona el flujo, qué command usar, qué hace un agente, cómo está organizado el proyecto, o cuando haya dudas sobre qué pipeline corresponde a cada tarea.
allowed-tools: Read
---

# Guía de Flujo — AEM + opencode

Este proyecto tiene **dos pipelines independientes** gestionados por agentes de opencode. Cada pipeline tiene su propio orquestador, scraper y contribuidor.

---

## Pipelines

### Pipeline 1 — PDF→DAM

**Objetivo**: Extraer PDFs enlazados en una URL y subirlos al AEM DAM.

```
[Usuario] → /pdf-pipeline <URL>
               ↓
         pdf-orchestrator
               ↓
         pdf-scraper          → context/pdf/json/{domain}/{page_slug}.json
               ↓ (si pdfs_found > 0)
         pdf-contributor      → /content/dam/{domain}/{subcarpeta}/
```

**Agentes:**
- `pdf-orchestrator` — orquestador primario, gestiona STATUS en `context/pdf/pipeline/`
- `pdf-scraper` — navega la URL con Playwright (--headed), extrae URLs de PDFs, escribe JSON
- `pdf-contributor` — descarga PDFs via fetch-en-browser (bypass Cloudflare), sube al DAM via MCP

---

### Pipeline 2 — WebPage→ContentFragment

**Objetivo**: Extraer contenido estructurado de una página web y crear un Content Fragment en AEM DAM.

```
[Usuario] → /cf-pipeline <URL>
               ↓
         cf-orchestrator
               ↓
         cf-scraper           → context/cf/{domain}/{page_slug}.json
               ↓
         cf-contributor       → /content/dam/{domain}/{content_type}/{cf_name}
                                /conf/global/settings/dam/cfm/models/{model_name}
```

**Agentes:**
- `cf-orchestrator` — orquestador primario, gestiona STATUS en `context/cf/pipeline/`
- `cf-scraper` — navega con Playwright (--headed), extrae título + contenido + campos etiquetados, deriva `content_type` de la URL, escribe JSON con sección `aem` completa
- `cf-contributor` — lee el JSON, crea el modelo CF si no existe (verificando fields en JCR), crea carpeta DAM, crea el CF via MCP

**Variante independiente:**
```
/cf-scrape <URL>           → cf-scraper → JSON
/xf-contribute <json_path> → xf-contributor → /content/experience-fragments/...
```

---

## Commands disponibles

| Command | Agente | Descripción |
|---------|--------|-------------|
| `/pdf-pipeline <URL>` | pdf-orchestrator | Pipeline completo PDF→DAM |
| `/scrape <URL>` | pdf-scraper | Solo extrae PDFs → JSON (sin subida) |
| `/upload <json_path>` | pdf-contributor | Solo sube PDFs al DAM desde JSON existente |
| `/cf-pipeline <URL>` | cf-orchestrator | Pipeline completo WebPage→CF en AEM |
| `/cf-scrape <URL>` | cf-scraper | Solo extrae contenido → JSON CF (sin crear en AEM) |
| `/xf-contribute <json_path>` | xf-contributor | Solo crea XF en AEM desde JSON existente |

---

## Estructura de carpetas `context/`

```
context/
  pdf/
    json/{domain}/
      {page_slug}.json          ← PDFs encontrados en la página
      scrape-log.md             ← log del pdf-scraper
      dam-log.md                ← log del pdf-contributor
    pipeline/{domain}/
      {page_slug}.md            ← STATUS del pipeline (IN_PROGRESS/COMPLETED/FAILED)
  cf/
    {domain}/
      {page_slug}.json          ← contenido + definición AEM completa
      scrape-log.md             ← log del cf-scraper
      cf-log.md                 ← log del cf-contributor
      xf-log.md                 ← log del xf-contributor
    pipeline/{domain}/
      {page_slug}.md            ← STATUS del pipeline CF
```

---

## Contrato JSON — CF pipeline

El `cf-scraper` produce un JSON que `cf-contributor` y `xf-contributor` consumen. Los campos son genéricos — el scraper los infiere dinámicamente de la URL y el contenido de la página:

```json
{
  "source_url": "https://{domain}/...",
  "domain": "{domain}",
  "page_slug": "{path-con-guiones}",
  "scraped_at": "2026-01-01T00:00:00Z",
  "content_type": "{derivado-del-path}",
  "aem": {
    "model_path": "/conf/global/settings/dam/cfm/models/{content_type}",
    "model_title": "{content_type humanizado}",
    "model_fields": [ ... campos inferidos de la página ... ],
    "parent_path": "/content/dam/{domain}/{content_type}",
    "cf_name": "{slug del item}",
    "cf_title": "{título extraído}",
    "cf_description": "Importado automáticamente desde {URL}"
  },
  "fields": {
    "titulo":    "{título}",
    "contenido": "{cuerpo de texto}",
    "url_origen":"{URL}",
    "{campo_extra_1}": "...",
    "relacionados": "..."
  }
}
```

**Principio clave**: `cf-contributor` y `xf-contributor` son ejecutores puros — toda la lógica de negocio (qué modelo, qué fields, qué rutas DAM) viene del JSON. Ningún agente contribuidor tiene lógica hardcodeada.

---

## Gotchas conocidos

### 1. Cloudflare en sitios protegidos
Algunos dominios tienen Cloudflare Turnstile. En modo headless el challenge falla silenciosamente (devuelve HTML vacío con status 200).

**Regla**: siempre `playwright-cli open --headed "{URL}"` — nunca headless.

### 2. MCP `manageContentFragmentModel` — update hace append, no replace
Si llamas `update` en un modelo que ya tiene fields, **añade fields duplicados** en lugar de reemplazarlos. El `create` inicial a veces escribe el nodo del modelo sin los field nodes.

**Regla en `cf-contributor`**:
1. Tras `create`, verificar con `getNodeContent` en `.../jcr:content/model/cq:dialog/content/items` que el número de nodos hijo == `model_fields.length`
2. Si hay duplicados (múltiplo): borrar modelo y recrear
3. Nunca usar `update` para añadir fields a un modelo existente

### 3. `getContentFragment` puede dar 404 aunque el CF exista
La Assets API (`/api/assets/`) a veces no indexa inmediatamente el CF recién creado. Verificar existencia con `getNodeContent` en la ruta JCR directa en vez de `getContentFragment`.

### 4. Descarga de PDFs con Cloudflare activo
No uses `curl`, `response-body` ni `route.fetch()` — Cloudflare devuelve HTML con status 200. La técnica correcta: `fetch()` desde dentro del browser (que ya tiene la sesión CF válida) + blob URL + evento `download`.

---

## MCP tools principales (mcp-aem-hiberus)

| Tool | Uso |
|------|-----|
| `manageContentFragmentModel` | Crear/borrar modelos CF en `/conf/` |
| `manageContentFragment` | Crear/actualizar/borrar instancias CF en DAM |
| `createDamFolder` | Crear carpetas en `/content/dam/` |
| `uploadDamAsset` | Subir assets (PDFs, imágenes) al DAM |
| `getNodeContent` | Verificar estado JCR crudo (más fiable que las APIs de assets) |
| `getContentFragment` | Leer CF con todos sus fields (puede dar 404 si recién creado) |
| `manageExperienceFragment` | Crear/actualizar XFs en `/content/experience-fragments/` |

---

## Leer el estado actual

Para consultar el estado real del proyecto, lee estos archivos:

- Agentes: `.opencode/agents/*.md`
- Commands: `.opencode/commands/*.md`
- Config opencode: `opencode.json` (contiene `default_agent` y config MCP)
- Logs de ejecución: `context/pdf/` y `context/cf/`
