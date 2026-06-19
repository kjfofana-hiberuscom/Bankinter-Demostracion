---
name: aem-cf-creation
description: Crea Content Fragments (CF) en AEM vía MCP. Cubre el ciclo completo: verificar/crear el modelo CF en /conf/global/settings/dam/cfm/models/, crear la carpeta en /content/dam/, y crear la instancia del CF con sus campos. Incluye el gotcha crítico de campos duplicados con manageContentFragmentModel. Usar SIEMPRE que el agente necesite crear, actualizar o verificar un Content Fragment en AEM.
---

# Creación de Content Fragments en AEM

Los Content Fragments (CF) son fragmentos de contenido estructurado. Viven en `/content/dam/` junto con otros assets DAM. Su estructura la define un **modelo CF** que declara qué campos tiene el fragmento.

## Estructura de un CF en AEM

```
/conf/global/settings/dam/cfm/models/
  └── {model_name}/                   ← modelo (define los campos)
        └── jcr:content/model/cq:dialog/content/items/
              ├── campo1              ← nodo de campo (field node)
              └── campo2

/content/dam/
  └── {domain}/                       ← agrupación por dominio
        └── {categoria}/              ← carpeta de categoría (ej: diccionario)
              └── {cf_name}           ← el CF concreto (nodo JCR)
```

---

## Flujo de creación — Paso a Paso

### Paso 1: Verificar/Crear el Modelo CF

El modelo define los campos del fragmento. **Solo crea el modelo si el JSON incluye `aem.model_fields`** (array no vacío). Si `model_fields` no está en el JSON, el modelo ya existe en AEM → salta al Paso 2.

#### GOTCHA CRÍTICO — Campos duplicados

`manageContentFragmentModel` con `action: "update"` hace **APPEND** de campos, no replace. Llamarlo dos veces sobre el mismo modelo crea campos duplicados. Nunca uses `update` para añadir o modificar campos de un modelo.

**a) Verificar si el modelo ya existe con los campos correctos:**

```
getNodeContent(
  path: "{aem.model_path}/jcr:content/model/cq:dialog/content/items",
  depth: 2
)
```

Interpreta el resultado:

- Nodo existe + **N hijos == model_fields.length** → modelo OK, **salta al Paso 2**.
- Nodo existe + **hijos = múltiplo de model_fields.length** → campos duplicados → borra y recrea (ver más abajo).
- Nodo no existe o `items` vacío → procede a crear el modelo.

**b) Crear el modelo:**

```
manageContentFragmentModel(
  action: "create",
  configPath: "{parte del model_path antes de /settings}",
                ← ej: "/conf/global" para "/conf/global/settings/dam/cfm/models/termino-financiero"
  modelName: "{última parte del model_path}",
                ← ej: "termino-financiero"
  title: "{aem.model_title}",
  fields: [
    { name: "campo1", label: "Campo 1", type: "text-single", required: true },
    { name: "campo2", label: "Campo 2", type: "text-multi",  required: false }
  ]
)
```

**c) Verificar que los campos se escribieron:**

Tras `create`, vuelve a llamar `getNodeContent` en `.../items`. Cuenta los nodos hijo:

- Count == `model_fields.length` → éxito.
- Count no coincide → el MCP no escribió los campos → ACK `FAIL | failure_class:AEM_DOWN`.

**d) Si hay campos duplicados — borrar y recrear:**

```
manageContentFragmentModel(action: "delete", modelPath: "{aem.model_path}")
```

Luego vuelve al paso (b) para crear desde cero.

### Paso 2: Crear la jerarquía de carpetas DAM y configurar la política CF

#### 2a — Crear las carpetas (mkdir -p automático)

```
createDamFolder(folderPath: "{aem.parent_path}")
```

`createDamFolder` crea toda la jerarquía de una vez: si `parent_path` es `/content/dam/bankinter.com/diccionario-economia`, crea tanto `/content/dam/bankinter.com` como el subpath completo. Carpetas ya existentes no dan error.

#### 2b — Configurar "Allowed Content Fragment Models" en la raíz del dominio

Esta política es **obligatoria**: sin ella, AEM no permite crear CFs en la carpeta aunque el modelo exista.

Se aplica en la carpeta raíz del dominio (no en la subcarpeta), y **todas las subcarpetas heredan** automáticamente.

Deriva las rutas necesarias:

- `domain_root`: primeros 3 segmentos del path → `/content/dam/{domain}` (ej: `/content/dam/bankinter.com`)
- `configPath`: parte del `model_path` antes de `/settings` → ej: `/conf/global`

```
setDamFolderCfPolicy(
  folderPath: "{domain_root}",          ← /content/dam/{domain}
  allowedByPath: ["{configPath}"],      ← ["/conf/global"]
  mode: "merge"                         ← no sobreescribe políticas existentes
)
```

`mode: "merge"` es seguro: si ya hay una política definida, la completa en lugar de borrarla.

**¿Cuándo necesitas esto?** Siempre que crees una carpeta nueva para un dominio. Las carpetas existentes con política ya configurada no necesitan este paso.

### Paso 3: Crear el Content Fragment

```
manageContentFragment(
  action: "create",
  parentPath: "{aem.parent_path}",
  model: "{aem.model_path}",
  title: "{aem.cf_title}",
  name: "{aem.cf_name}",           ← opcional: si no se pasa, AEM genera el name del título
  description: "{aem.cf_description}",  ← opcional
  fields: {
    "campo1": "valor1",
    "campo2": "valor2"
  }
)
```

Los valores de `fields` vienen **tal cual** del JSON de entrada — no los interpretes ni modifiques.

**Si el CF ya existe:**

- Usa `action: "update"` con los mismos parámetros.
- Verifica existencia primero con `getNodeContent` (ver más abajo).

### Paso 4: Verificar existencia del CF

**No uses `getContentFragment` para verificar existencia inmediata** — puede dar 404 aunque el CF exista, porque la indexación de assets DAM es asíncrona.

Usa en cambio:

```
getNodeContent(
  path: "{aem.parent_path}/{aem.cf_name}",
  depth: 1
)
```

Si el nodo existe → CF creado correctamente.

---

## Tipos de campos en modelos CF

| `type`               | Descripción                  |
| -------------------- | ---------------------------- |
| `text-single`        | Texto de una línea           |
| `text-multi`         | Texto multilínea o rich text |
| `number`             | Número entero o decimal      |
| `boolean`            | Verdadero/Falso              |
| `date`               | Fecha (ISO 8601)             |
| `enum`               | Lista de valores cerrada     |
| `fragment-reference` | Referencia a otro CF         |

---

## Gotchas críticos

| Problema                                           | Causa                                   | Solución                                                                |
| -------------------------------------------------- | --------------------------------------- | ----------------------------------------------------------------------- |
| `manageContentFragmentModel update` duplica campos | `update` hace append, no replace        | Nunca uses `update` para campos; borra y recrea el modelo               |
| `getContentFragment` da 404                        | Indexación DAM asíncrona                | Usa `getNodeContent` para verificar existencia inmediata                |
| CF creado sin campos                               | `fields` vacío o con claves incorrectas | Verifica que las claves de `fields` coinciden con los `name` del modelo |
| Modelo sin nodos hijo tras `create`                | Bug del MCP o AEM inestable             | Verifica con `getNodeContent`, reporta AEM_DOWN si persiste             |
| `createDamFolder` error                            | Ruta padre no existe                    | Crea los niveles intermedios primero                                    |

---

## Referencia de herramientas MCP

| Operación                               | Tool MCP                                                                     |
| --------------------------------------- | ---------------------------------------------------------------------------- |
| Verificar si modelo existe/tiene campos | `getNodeContent` en `{model_path}/jcr:content/model/cq:dialog/content/items` |
| Crear modelo CF                         | `manageContentFragmentModel` `action: "create"`                              |
| Borrar modelo (antes de recrear)        | `manageContentFragmentModel` `action: "delete"`                              |
| Crear carpeta DAM (mkdir -p)            | `createDamFolder`                                                            |
| Política CF en carpeta dominio          | `setDamFolderCfPolicy` `allowedByPath` + `mode: "merge"`                     |
| Crear CF                                | `manageContentFragment` `action: "create"`                                   |
| Actualizar CF                           | `manageContentFragment` `action: "update"`                                   |
| Verificar existencia CF                 | `getNodeContent` con `depth: 1`                                              |
| Leer CF con campos                      | `getContentFragment` (puede dar 404 si recién creado)                        |
| Listar CFs en carpeta                   | `listContentFragments`                                                       |
