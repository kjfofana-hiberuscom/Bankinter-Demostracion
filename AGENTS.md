# Sistema de Agentes — Bankinter PDF→DAM Pipeline

## Arquitectura

```
Usuario
  └── orchestrator (primary)
        ├── web-scraper    (subagent, hidden)  — Playwright → context/json/
        └── dam-contributor (subagent, hidden) — curl + MCP → AEM DAM
```

## Agentes

| Agente | Modo | Propósito |
|---|---|---|
| `orchestrator` | primary | Interlocutor único. Recibe URL, coordina pipeline, gestiona ACKs |
| `web-scraper` | subagent | Playwright CLI headless → extrae URLs de PDFs → JSON |
| `dam-contributor` | subagent | curl descarga PDFs → MCP `uploadDamAsset` → AEM DAM |

## Protocolo ACK

Todos los subagentes retornan una línea estructurada al orchestrator:

```
{agent} | {STATUS} | {key:value}... | questions:{N} | failure_class:{CLASS}
```

### web-scraper ACK

```
web-scraper | COMPLETE | pdfs_found:5 | json_path:context/json/bankinter.com/banca-nav.json | questions:0 | failure_class:-
web-scraper | FAIL    | pdfs_found:0 | json_path:-                                          | questions:0 | failure_class:NETWORK
```

failure_class: `NETWORK` | `AUTH` | `PLAYWRIGHT`

### dam-contributor ACK

```
dam-contributor | COMPLETE | uploaded:5 | failed:0 | dam_path:/content/dam/bankinter.com/atencion-cliente/ | questions:0 | failure_class:-
dam-contributor | FAIL    | uploaded:0 | failed:5 | dam_path:-                                            | questions:0 | failure_class:AEM_DOWN
```

failure_class: `AEM_DOWN` | `DOWNLOAD`

## Routing del Orchestrator

### ACK web-scraper → dam-contributor
| Condición | Acción |
|---|---|
| `COMPLETE` + `pdfs_found:0` | Informa usuario. Detente. |
| `COMPLETE` + `pdfs_found:>0` | Lanza dam-contributor con `json_path` |
| `FAIL` + `failure_class:*` | Reporta al usuario. No relanzas. |
| `questions:>0` | Bloquea. Presenta pregunta al usuario. Relanza con respuesta. |

### ACK dam-contributor → cierre
| Condición | Acción |
|---|---|
| `COMPLETE` + `failed:0` | Éxito total. Informa ruta DAM. |
| `COMPLETE` + `failed:>0` | Éxito parcial. Lista PDFs fallidos. |
| `FAIL` + `failure_class:AEM_DOWN` | Reporta: verificar AEM en `http://localhost:4502`. |
| `FAIL` + `failure_class:DOWNLOAD` | Reporta: URLs inaccesibles. |
| `questions:>0` | Bloquea. Presenta pregunta al usuario. |

## Artefactos persistentes

| Artefacto | Quién escribe | Propósito |
|---|---|---|
| `context/json/{domain}/{page}.json` | web-scraper | Contrato entre agentes. Input para dam-contributor |
| `context/json/{domain}/scrape-log.md` | web-scraper | Worker log de extracción |
| `context/json/{domain}/dam-log.md` | dam-contributor | Worker log de subida |
| `context/pipeline/{domain}/{page_slug}.md` | orchestrator | STATUS por ejecución — espeja la clave del JSON. N URLs = N ficheros independientes |

## Estructura DAM generada

```
/content/dam/{domain}/
  └── {subcarpeta-del-path}/
        ├── documento1.pdf
        └── documento2.pdf
```

Ejemplo: URL `https://bankinter.com/banca/nav/atencion-cliente/elevar-reclamacion`
→ DAM: `/content/dam/bankinter.com/atencion-cliente/`

## Uso

```
# En opencode TUI o CLI — el orchestrator es el agente por defecto
Sube los PDFs de https://www.bankinter.com/banca/nav/atencion-cliente/elevar-reclamacion al DAM
```

## Reglas universales de los subagentes

- `task: deny` — no anidan subagentes propios
- `hidden: true` — no aparecen en autocompletado `@`
- Git mutativo nunca
- Preguntas al usuario: solo via ACK `questions:>0`, nunca directamente
- Máximo 1 relanzamiento automático por error antes de escalar al usuario
