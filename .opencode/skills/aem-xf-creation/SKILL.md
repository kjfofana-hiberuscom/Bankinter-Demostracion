---
name: aem-xf-creation
description: Crea Experience Fragments (XF) en AEM vía MCP. Cubre el ciclo completo desde verificar el template disponible, crear la estructura de carpetas en /content/experience-fragments/ con sus propiedades, crear el XF y añadir variaciones (master, web). Incluye los gotchas críticos de AEM para XFs. Usar SIEMPRE que el agente necesite crear, actualizar o verificar un Experience Fragment en AEM.
---

# Creación de Experience Fragments en AEM

Los Experience Fragments (XF) son fragmentos de experiencia reutilizables — combinan contenido y layout para ser embebidos en páginas. Viven en `/content/experience-fragments/`, **no en el DAM**.

## Estructura de un XF en AEM

```
/content/experience-fragments/
  └── {domain}/                       ← agrupación por dominio (ej: bankinter.com)
        └── {content_type}/           ← categoría (ej: diccionario, blog, productos)
              └── {xf_name}/          ← el XF concreto
                    ├── jcr:content   ← propiedades del XF (title, description, template)
                    └── master/       ← variación master (siempre existe)
                          └── jcr:content/root/...  ← componentes del contenido
```

Los **templates** de XF están en `/conf/global/settings/wcm/templates/`. El template estándar para la mayoría de XFs es:

```
/conf/global/settings/wcm/templates/experience-fragment-web-variation
```

---

## Flujo de creación — Paso a Paso

### Paso 1: Verificar que el template existe

Antes de crear el XF, confirma que el template está disponible en la instancia:

```
getNodeContent(
  path: "/conf/global/settings/wcm/templates/experience-fragment-web-variation",
  depth: 1
)
```

- Si el nodo **existe** → continúa con el Paso 2.
- Si **no existe** → usa `getTemplates` para listar los templates disponibles y elegir el más apropiado.
- Si AEM no responde → ACK `FAIL | failure_class:AEM_DOWN`.

### Paso 2: Preparar la carpeta padre

La ruta padre del XF (`/content/experience-fragments/{domain}/{content_type}`) debe existir antes de crear el XF. AEM **no crea la jerarquía automáticamente** al crear un XF.

Verifica si existe:

```
getNodeContent(
  path: "/content/experience-fragments/{domain}/{content_type}",
  depth: 1
)
```

**Si no existe**, crea cada nivel que falte con `manageExperienceFragment` usando `action: "create"` con un template de carpeta, o usa `createPage` si el MCP lo soporta para rutas de experience-fragments. En la práctica, intenta crear el XF directamente — si AEM crea la ruta automáticamente, perfecto; si falla con "parent not found", entonces crea los niveles intermedios primero.

### Paso 3: Crear el Experience Fragment

```
manageExperienceFragment(
  action: "create",
  parentPath: "/content/experience-fragments/{domain}/{content_type}",
  name: "{xf_name}",
  title: "{xf_title}",
  description: "{xf_description}",        ← opcional
  template: "/conf/global/settings/wcm/templates/experience-fragment-web-variation"
)
```

**Si el XF ya existe** → usa `action: "update"` con el mismo `parentPath` + `name`.

**Verificar existencia inmediata** — no uses `getExperienceFragment` (puede dar 404 aunque el XF exista). Usa en cambio:

```
getNodeContent(
  path: "/content/experience-fragments/{domain}/{content_type}/{xf_name}",
  depth: 1
)
```

### Paso 4: Actualizar la variación master

Cuando creas un XF, AEM **crea automáticamente** la variación `master`. Por tanto:

- Llamar `manageExperienceFragmentVariation` con `action: "create"` + `variationName: "master"` → **error "ya existe"**.
- Siempre usa `action: "update"` para `master` en un XF recién creado.

```
manageExperienceFragmentVariation(
  action: "update",
  xfPath: "/content/experience-fragments/{domain}/{content_type}/{xf_name}",
  variationName: "master",
  title: "Master",
  content: "{contenido construido desde los campos del JSON}"
)
```

**Construye el contenido** concatenando los campos del JSON con formato legible:

```
{label_campo1}: {valor1}

{label_campo2}: {valor2}

...
```

### Paso 5: Crear variaciones adicionales (opcional)

Si el JSON especifica variaciones adicionales (ej: `"web"`, `"email"`):

```
manageExperienceFragmentVariation(
  action: "create",
  xfPath: "/content/experience-fragments/{domain}/{content_type}/{xf_name}",
  variationName: "web",
  title: "Web",
  content: "{mismo contenido o adaptado}"
)
```

Si la variación ya existe → usa `action: "update"`.

---

## Propiedades de carpeta (Folder Properties)

Las carpetas de XF pueden tener restricciones de templates (qué templates están permitidos crear dentro de ellas). Si AEM rechaza la creación del XF con un error de template no permitido:

1. Verifica las propiedades actuales de la carpeta:

   ```
   getNodeContent(
     path: "/content/experience-fragments/{domain}",
     depth: 2
   )
   ```

2. Busca el nodo `jcr:content` con `cq:allowedTemplates`. Si no existe o no incluye el template necesario, el XF no se puede crear sin modificar las propiedades de la carpeta.

3. En ese caso, usa `updateComponent` o `getNodeContent` + una operación CRUD sobre el nodo JCR para añadir el template a `cq:allowedTemplates`.

> **Nota**: Las instancias AEM de desarrollo suelen tener `cq:allowedTemplates: [".*"]` (todos permitidos). Si estás en una instancia con políticas restrictivas, esto será un bloqueador.

---

## Gotchas críticos

| Problema                                 | Causa                                      | Solución                                             |
| ---------------------------------------- | ------------------------------------------ | ---------------------------------------------------- |
| `getExperienceFragment` da 404           | Indexación pendiente tras creación         | Usa `getNodeContent` para verificar existencia       |
| `create` en variación `master` falla     | AEM la crea automáticamente con el XF      | Usa `action: "update"` siempre para `master`         |
| XF no se crea por "template not allowed" | Carpeta padre tiene templates restrictivos | Verifica `cq:allowedTemplates` en la carpeta         |
| Path no encontrado al crear              | Ruta padre no existe en AEM                | Crea primero los niveles de carpeta intermedios      |
| Contenido no aparece en la variación     | `content` vacío o mal formado              | Construye el contenido concatenando todos los campos |

---

## Referencia de herramientas MCP

| Operación                           | Tool MCP                                                |
| ----------------------------------- | ------------------------------------------------------- |
| Verificar template disponible       | `getNodeContent` en `/conf/global/.../templates/{name}` |
| Listar templates disponibles        | `getTemplates`                                          |
| Verificar si carpeta/XF existe      | `getNodeContent` con `depth: 1`                         |
| Crear XF                            | `manageExperienceFragment` `action: "create"`           |
| Actualizar XF                       | `manageExperienceFragment` `action: "update"`           |
| Crear variación nueva               | `manageExperienceFragmentVariation` `action: "create"`  |
| Actualizar variación (incl. master) | `manageExperienceFragmentVariation` `action: "update"`  |
| Listar XFs en una ruta              | `listExperienceFragments`                               |
| Leer XF completo (puede dar 404)    | `getExperienceFragment`                                 |
| Ver árbol de XFs                    | `getExperienceFragmentTree`                             |

---

## Valores por defecto si el JSON no especifica `xf`

Si la sección `xf` no está en el JSON de entrada, derívala así:

| Campo            | Derivación                                                              |
| ---------------- | ----------------------------------------------------------------------- |
| `parent_path`    | `/content/experience-fragments/{domain}/{content_type}`                 |
| `xf_name`        | Último segmento de `page_slug` o el `page_slug` completo si es corto    |
| `xf_title`       | Primer valor en `fields`, o `page_slug` humanizado                      |
| `xf_description` | `"Importado automáticamente desde {source_url}"`                        |
| `template`       | `/conf/global/settings/wcm/templates/experience-fragment-web-variation` |
| `variations`     | `["master"]`                                                            |
