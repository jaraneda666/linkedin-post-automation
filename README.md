# 🤖 LinkedIn Post Automation — RSS + IA + Backups

Automatización en **n8n** que publica posts profesionales en LinkedIn de forma semanal, generados por IA a partir de noticias reales de ciberseguridad, infraestructura, IA y hardening — con imagen incluida y sistema de respaldo (backup) en cada API externa.

---

## 📋 Tabla de contenidos

- [¿Qué hace este workflow?](#qué-hace-este-workflow)
- [Arquitectura](#arquitectura)
- [Requerimientos](#requerimientos)
- [Instalación](#instalación)
- [Configuración de credenciales](#configuración-de-credenciales)
- [Estructura de Google Sheets](#estructura-de-google-sheets)
- [Uso](#uso)
- [Troubleshooting](#troubleshooting)
- [Roadmap / mejoras futuras](#roadmap--mejoras-futuras)

---

## ¿Qué hace este workflow?

Cada **lunes a las 8:00 AM**, el workflow:

1. **Lee 4 feeds RSS** de fuentes reconocidas de ciberseguridad, infraestructura, IA y hardening
2. **Selecciona una noticia nueva** (que no se haya publicado antes, usando un log en Google Sheets)
3. **Genera un post de LinkedIn** con IA a partir de esa noticia (Gemini como motor principal, Groq como respaldo automático)
4. **Genera una imagen conceptual** relacionada al post (Pollinations.ai como principal, HuggingFace como respaldo)
5. **Publica en LinkedIn** con imagen adjunta (o solo texto si la generación de imagen falla)
6. **Envía notificación por Gmail** confirmando éxito o fallo
7. **Registra el post** en Google Sheets (para no repetir noticias) y guarda un backup CSV en Google Drive

---

## Arquitectura

```
Schedule Trigger (Lunes 8AM)
│
├─→ RSS Ciberseguridad ─┐
├─→ RSS Infraestructura ─┤
├─→ RSS IA ───────────────┼─→ Select New Topic (filtra ya publicados) ─→ IF: ¿Hay contenido nuevo?
├─→ RSS Hardening ────────┤ │
└─→ Read Posted Log ─────┘ ▼
Prepare Prompt
│
▼
Generar Post - Gemini (Primario)
│ (error) │ (ok)
▼ ▼
Generar Post - Groq (Backup) Normalizar salida Gemini
│ │
└──────┬───────┘
▼
Final Post Text
│
▼
Generar Imagen - Pollinations (Primario)
│ (error) │ (ok)
▼ ▼
Generar Imagen - HuggingFace (Backup) Mark Image OK
│ (error) │ (ok)
▼ ▼
Sin Imagen Mark Image OK
│ │
└──────┬───────┘
▼
Final Image
│
▼
IF: ¿Hay imagen disponible?
│ │
Sí No
▼ ▼
Registrar Upload → Combinar → Publicar Post
Subir Binario → Publicar Solo Texto
Post con Imagen (Fallback)
│ │
└───────────┬────────────┘
▼
IF: ¿Publicación exitosa?
│ │
Sí No
▼ ▼
Success Mail Failure Mail
│
▼
Log Posted Item (Sheets)
│
▼
CSV + Upload a Google Drive
```

---

## Requerimientos

| Servicio | Uso | Costo |
|---|---|---|
| **n8n** (self-hosted o cloud) | Motor de la automatización | Gratis (self-hosted) |
| **Google Cloud Project** | Sheets, Drive, Gmail | Gratis |
| **Google Gemini API** | Generación de texto (primario) | Gratis (free tier) |
| **Groq API** | Generación de texto (backup) | Gratis (free tier) |
| **Pollinations.ai** | Generación de imagen (primario) | Gratis, sin API key |
| **HuggingFace Inference API** | Generación de imagen (backup) | Gratis (free tier) |
| **LinkedIn Developer App** | Publicación de posts | Gratis |
| **Gmail** | Notificaciones | Gratis |

---

## Instalación

### 1. Importar el workflow

1. En n8n, ve a **Workflows** → **Add workflow** → menú `...` → **Import from File**
2. Selecciona `linkedin-post-automation.json` (incluido en este repo)

### 2. Configurar credenciales

n8n **no importa credenciales por seguridad** — hay que crearlas manualmente. Ver sección [Configuración de credenciales](#configuración-de-credenciales) abajo.

### 3. Crear la hoja de log en Google Sheets

Ver sección [Estructura de Google Sheets](#estructura-de-google-sheets).

### 4. Actualizar referencias en los nodos

Reemplaza en el workflow (todos marcados con placeholders tipo `YOUR_*`):
- `documentId` / `sheetName` de los nodos **Read Posted Log** y **Log Posted Item** con tu propio spreadsheet
- `person: "YOUR_LINKEDIN_PERSON_URN"` en los nodos de LinkedIn con tu propio Person URN
- `sendTo: "YOUR_EMAIL@example.com"` en los nodos de Gmail con tu correo

---

## Configuración de credenciales

### 🔑 Google Gemini API
- Consíguela en: https://aistudio.google.com/apikey
- Tipo de credencial en n8n: **Google Gemini (PaLM) Api**

### 🔑 Groq API Key (backup de texto)
- Consíguela en: https://console.groq.com/keys
- Tipo de credencial en n8n: **Header Auth**
  - Name: `Authorization`
  - Value: `Bearer <tu_key>`

### 🔑 HuggingFace API Key (backup de imagen)
- Consíguela en: https://huggingface.co/settings/tokens (rol "Read")
- Tipo de credencial en n8n: **Header Auth**
  - Name: `Authorization`
  - Value: `Bearer <tu_token>`

### 🔑 LinkedIn OAuth2
- Crea una app en: https://www.linkedin.com/developers/apps
- Scopes requeridos: `w_member_social`
- Tipo de credencial en n8n: **LinkedIn OAuth2 API**

### 🔑 Gmail OAuth2
1. Crea un **OAuth Client ID tipo "Web application"** (no "Desktop") en Google Cloud Console
2. Agrega como Authorized redirect URI la URL que te muestra n8n en el campo "OAuth Redirect URL"
3. Habilita la Gmail API en tu proyecto de Google Cloud
4. Si tu app está en modo "Testing", agrega tu cuenta como **Test user** en la pantalla de consentimiento OAuth
5. Tipo de credencial en n8n: **Gmail OAuth2 API**

### 🔑 Google Service Account (Sheets + Drive)
1. Crea una Service Account en Google Cloud Console (IAM & Admin → Service Accounts)
2. Genera una key en formato **JSON**
3. Comparte tu Google Sheet con el email de la Service Account (permiso **Editor**)
4. Tipo de credencial en n8n: **Google Service Account API**
   - Service Account Email: el `client_email` del JSON
   - Private Key: el campo `private_key` completo del JSON

> ⚠️ **Nota**: las Service Accounts **no tienen cuota de almacenamiento** en Google Drive personal. Para el nodo de subida a Drive (backup CSV), usa una credencial **OAuth2** (tu cuenta personal) en vez de Service Account.

---

## Estructura de Google Sheets

Crea un spreadsheet con una pestaña llamada exactamente **`PostedLog`**, con estos encabezados en la fila 1:

| Columna | Tipo | Descripción |
|---|---|---|
| `Title` | Texto | Título de la noticia RSS original |
| `Link` | URL | Link único — clave para evitar publicar la misma noticia dos veces |
| `Category` | Texto | Una de: `Ciberseguridad`, `Infraestructura`, `IA`, `Hardening` |
| `PostedText` | Texto largo | El post generado que se publicó |
| `URN` | Texto | ID que LinkedIn devuelve al publicar |
| `PostedAt` | Fecha/hora | Timestamp de publicación |

```csv
Title,Link,Category,PostedText,URN,PostedAt
```

---

## Uso

### Ejecución manual (prueba)
1. Abre el workflow en n8n
2. Click en **"Test workflow"**
3. Revisa cada nodo en busca de errores

### Activación automática
1. Verifica que todos los nodos tengan credenciales asignadas correctamente
2. Activa el toggle **"Active"** en la parte superior del workflow
3. Se ejecutará automáticamente cada **lunes a las 8:00 AM**

### Cambiar la frecuencia o el día
Edita el nodo **"Lunes 8AM"** → parámetro `rule.interval` → ajusta `triggerAtDay` (0=domingo, 1=lunes...) y `triggerAtHour`.

### Agregar/cambiar fuentes RSS
Edita la URL en cualquiera de los nodos `RSS - *`, o duplica uno existente, conéctalo al Schedule Trigger, y súmalo como una entrada más al nodo `Merge - <Categoría>` correspondiente (subiendo su `numberInputs` en 1).

> ⚠️ **No conectes el nuevo nodo RSS directamente al nodo `Tag: <Categoría>` ni a `Select New Topic`.** Eso reintroduce el bug de publicaciones duplicadas — ver tabla de Troubleshooting.

Fuentes incluidas en el workflow (13 en total):

**Ciberseguridad**
- `https://feeds.feedburner.com/TheHackersNews`
- `https://krebsonsecurity.com/feed/`
- `https://www.darkreading.com/rss.xml`
- `https://www.theregister.com/security/headlines.atom`
- `https://www.wired.com/feed/category/security/latest/rss`

**Infraestructura**
- `https://isc.sans.edu/rssfeed_full.xml`
- `https://aws.amazon.com/blogs/security/feed/`
- `https://kubernetes.io/feed.xml`
- `https://www.docker.com/blog/feed/`
- `https://www.hashicorp.com/blog/feed.xml`

**IA**
- `https://blog.research.google/feeds/posts/default`
- `https://openai.com/blog/rss.xml`

**Hardening**
- `https://www.cisa.gov/cybersecurity-advisories/all.xml`

---

## Troubleshooting

| Problema | Causa común | Solución |
|---|---|---|
| `Bad request` en "Subir Binario Imagen" | Uploads a LinkedIn requieren `Authentication: None` y `Body Content Type: n8n Binary File` | Verificar configuración del nodo, no usar credencial predefinida en este PUT específico |
| `The item has no binary field 'data'` | El binario se pierde al pasar por nodos Set | Usar **Code nodes** en vez de Set nodes para pasos donde se debe preservar el binario |
| `redirect_uri_mismatch` en OAuth | Cliente OAuth tipo "Desktop" en vez de "Web application" | Crear un nuevo Client ID tipo "Web application" con el redirect URI de n8n registrado |
| `Service Accounts do not have storage quota` | Service Account no puede escribir en "My Drive" personal | Usar credencial **OAuth2** para el nodo de Google Drive, no Service Account |
| Nunca publica nada (`Select New Topic` siempre devuelve `found: false`, aunque haya feeds con contenido nuevo) | Los nodos `Set` ("Tag: *" y "Tag Log Entries") descartan por defecto todos los campos de entrada salvo el que asignan explícitamente (`includeOtherFields` es `false` por defecto en n8n). Sin ese flag, cada item RSS pierde su `link` y cada entrada del log pierde su `Link` antes de llegar a `Select New Topic` — ambos quedan `undefined`, que siempre "coincide", así que el filtro de deduplicación descarta todo | Agregar `"includeOtherFields": true` en el `parameters` de esos 5 nodos `Set` |
| `Bad request` / modelo no encontrado en Gemini o Groq | Los proveedores deprecan modelos con el tiempo (ej. Gemini 2.0 Flash-Lite y Groq `llama-3.1-8b-instant` fueron dados de baja a mediados de 2026) | Revisar el modelo vigente en la documentación oficial del proveedor y actualizar el `modelId` (Gemini) o el campo `model` del body (Groq) |
| El post sale con `**negrita**` o etiquetas de metadata | El modelo de IA copia literalmente el formato del prompt | Ajustar el system prompt para prohibir markdown y etiquetas explícitamente (ver nodo "Generar Post - Gemini") |
| Se publican 2-3 posts duplicados en LinkedIn en una sola corrida | Varios nodos conectados directamente a la misma entrada de otro nodo (ej. varios `RSS - *` → `Select New Topic`) sin un nodo `Merge`. n8n no combina esas ramas automáticamente: ejecuta el nodo destino una vez *por cada rama*, generando selecciones y publicaciones independientes | Agrupar cada familia de feeds con un nodo `Merge` (modo `Append`, `numberInputs` = cantidad de ramas) antes de unirlas — ver nodos `Merge - Ciberseguridad`, `Merge - Infraestructura`, `Merge - IA` y `Merge - PreSelect` en el workflow |
| `invalid syntax` / `JSON Body is not valid JSON` en nodos HTTP Request | Campo en modo "Fixed" en vez de "Expression" | Cambiar el toggle del campo a "Expression" y usar `JSON.stringify()` |

---

## Roadmap / mejoras futuras

- [ ] Soporte multi-plataforma (Twitter/X, Mastodon)
- [ ] Resumen semanal de posts publicados
- [ ] Selección de tema por IA en vez de aleatoria (priorizar noticias más relevantes/recientes)
- [ ] Dashboard de métricas de engagement por categoría
- [ ] Revisión humana opcional antes de publicar (Google Docs draft, como en versiones anteriores)

---

## Créditos

Workflow desarrollado y depurado iterativamente con asistencia de Claude (Anthropic), corrigiendo bugs originales de conexión, autenticación y manejo de binarios en n8n v2.32.7.

## Licencia

Uso personal / educativo. Ajusta las fuentes RSS y credenciales según tus necesidades.
