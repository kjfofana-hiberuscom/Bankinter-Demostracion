---
name: cloudflare-image-download-dam
description: >
  Descarga archivos (PDFs, imágenes) desde dominios protegidos por Cloudflare Turnstile
  y los sube al AEM DAM. Úsala SIEMPRE que necesites descargar assets desde bankinter.com
  u otro dominio con Cloudflare antes de subirlos a DAM. La técnica correcta es
  fetch-in-browser + blob download via playwright-cli — NO uses response-body, curl,
  wget, ni route.fetch() con Cloudflare.
---

# Cloudflare Asset Download → AEM DAM

## Por qué las técnicas habituales fallan con Cloudflare

| Técnica | Resultado | Motivo |
|---|---|---|
| `playwright-cli response-body` | HTML del challenge | Chrome PDF viewer intercepta los bytes antes de playwright |
| `curl` / `wget` | HTML 403 o challenge | Sin fingerprint de navegador real |
| `route.fetch()` / `context.request.get()` | Bloqueado | Petición standalone sin sesión Cloudflare |
| `page.goto()` directo | Puede lanzar "Download is starting" | Content-Disposition fuerza descarga |

**La única técnica que funciona sin instalar playwright npm:**
`fetch` dentro de `page.evaluate` (contexto browser con sesión Cloudflare válida) → Blob → URL objeto → `<a download>` click → `page.waitForEvent('download')` → `download.saveAs()`

## Proceso completo

### Paso 0 — Establecer sesión Cloudflare (una vez por dominio)

Abre la homepage del dominio en modo headed para que Cloudflare valide el navegador:

```bash
playwright-cli open --headed "https://www.{dominio}/" 2>&1
```

Espera a que cargue completamente antes de continuar.

### Paso 1 — Descargar cada archivo

```bash
playwright-cli run-code "async page => {
  const url = '{url_del_archivo}';
  const [download] = await Promise.all([
    page.waitForEvent('download'),
    page.evaluate(async (u) => {
      const r = await fetch(u);
      const blob = await r.blob();
      const blobUrl = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = blobUrl;
      a.download = '{nombre_archivo}';
      document.body.appendChild(a);
      a.click();
      URL.revokeObjectURL(blobUrl);
    }, url)
  ]);
  await download.saveAs('/tmp/dam-upload/{dominio}/{nombre_archivo}');
  return download.suggestedFilename() + ' saved';
}" 2>&1
```

### Paso 2 — Validar cabecera binaria (OBLIGATORIO)

Nunca confíes solo en el tamaño — Cloudflare puede devolver HTML con 200 OK y >1000 bytes.

```bash
head -c 4 /tmp/dam-upload/{dominio}/{nombre_archivo}
ls -lh /tmp/dam-upload/{dominio}/{nombre_archivo}
```

| Tipo | Cabecera esperada | Cabecera Cloudflare |
|---|---|---|
| PDF | `%PDF` | `<!DO` o `<htm` |
| PNG | `\x89PNG` | `<!DO` o `<htm` |
| JPG | `\xFF\xD8` | `<!DO` o `<htm` |

- Cabecera correcta **Y** tamaño > 1000 bytes → OK
- Cabecera `<!DO` o `<htm` → Cloudflare bloqueó, marca como fallido

### Paso 3 — Subir al AEM DAM

```
uploadDamAsset({
  localPath: "/tmp/dam-upload/{dominio}/{nombre_archivo}",
  damPath: "/content/dam/{dominio}/{subcarpeta}/{nombre_archivo}",
  mimeType: "application/pdf"
})
```

### Paso 4 — Cerrar browser (tras procesar todos los archivos)

```bash
playwright-cli close 2>&1
```

Cierra solo al final, no entre archivos del mismo dominio — la sesión Cloudflare se comparte.

## Notas importantes

- El `Promise.all` es crítico: `waitForEvent('download')` debe registrarse **antes** de que el click dispare la descarga, si no la promesa no se resuelve.
- `import`, `require` y `module.exports` no funcionan en `run-code` — usa solo la API de Playwright (`download.saveAs`).
- Si el dominio tiene subdominios distintos (ej: `docs.bankinter.com` vs `www.bankinter.com`), abre sesión para cada uno.
