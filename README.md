# 🤖 LinkedIn Post Automation — RSS + IA Multimodal + Cloudflare FLUX.1 + Backups

Automatización avanzada en **n8n** que publica posts profesionales en LinkedIn de forma semanal o a demanda, generados por IA a partir de noticias reales de ciberseguridad, infraestructura, IA y hardware/hardening — con generación de imágenes de alta fidelidad mediante **Cloudflare Workers AI (FLUX.1)** y sistema redundante de respaldo (*fallback*) en cada etapa.

---

## 📋 Tabla de contenidos

- [¿Qué hace este workflow?](#qué-hace-este-workflow)
- [Arquitectura](#arquitectura)
- [Novedades de la última versión](#novedades-de-la-última-versión)
- [Requerimientos](#requerimientos)
- [Instalación](#instalación)
- [Configuración de credenciales](#configuración-de-credenciales)
- [Estructura de Google Sheets](#estructura-de-google-sheets)
- [Fuentes RSS integradas](#fuentes-rss-integradas)
- [Uso](#uso)
- [Troubleshooting](#troubleshooting)
- [Roadmap / mejoras futuras](#roadmap--mejoras-futuras)

---

## ¿Qué hace este workflow?

Cada **lunes a las 8:00 AM** (o mediante disparo manual por Webhook):

1. **Monitorea 23 feeds RSS** especializados en Ciberseguridad, Infraestructura, Inteligencia Artificial y Hardening/Hardware.
2. **Selecciona una noticia nueva** no publicada previamente, consultando la base histórica en Google Sheets.
3. **Genera un post bilingüe estructurado (Español + Inglés)** utilizando **Google Gemini** como motor principal (con **Groq / Llama** como respaldo automático).
4. **Crea un prompt visual técnico y contextual** sin texto superpuesto defectuoso, enfocado en diagramas de arquitectura, esquemas de sistemas o conceptos de seguridad limpios.
5. **Genera la imagen técnica en alta resolución** mediante **Cloudflare Workers AI (`@cf/black-forest-labs/flux-1-schnell`)** como motor primario (con **HuggingFace Inference Router** como backup secundario).
6. **Publica el post en LinkedIn** con la imagen adjunta (o solo texto si fallan todos los motores de imagen).
7. **Envía notificación por Gmail** con el resultado de la publicación.
8. **Registra la publicación en Google Sheets** y genera un backup en CSV respaldado en **Google Drive**.

---

## Arquitectura

```
Schedule Trigger (Lunes 8AM) / Webhook Manual
│
├── RSS Ciberseguridad (6 feeds) ─→ Merge ─┐
├── RSS Infraestructura (5 feeds) ─→ Merge ─┤
├── RSS IA (5 feeds) ───────────────→ Merge ─┼─→ Select New Topic ─→ IF: ¿Hay contenido nuevo?
├── RSS Hardening (7 feeds) ────────→ Merge ─┤   (filtra duplicados)     │
└── Read Posted Log (Sheets) ───────────────┘                           ▼
                                                                Prepare Prompt
                                                                       │
                                      ┌────────────────────────────────┴────────────────────────────────┐
                                      ▼                                                                 ▼
                        Generar Post - Gemini (Primario)                              Generar Prompt Imagen (Contextual)
                                      │ (error)                                                         │
                                      ▼                                                                 ▼
                         Generar Post - Groq (Backup)                                Generar Imagen - Cloudflare FLUX.1 (Primario)
                                      │                                                                 │ (error)
                                      ▼                                                                 ▼
                               Final Post Text                                      Generar Imagen - HuggingFace Router (Backup)
                                      │                                                                 │
                                      └────────────────────────────────┬────────────────────────────────┘
                                                                       ▼
                                                          IF: ¿Hay imagen disponible?
                                                                 │           │
                                                                Sí           No
                                                                 ▼           ▼
                                                     Registrar Upload     Publicar Solo Texto
                                                             │           (Fallback)
                                                       Subir Binario             │
                                                             │                   │
                                                     Publicar con Imagen         │
                                                             │                   │
                                                             └─────────┬─────────┘
                                                                       ▼
                                                           IF: ¿Publicación exitosa?
                                                                 │           │
                                                                Sí           No
                                                                 ▼           ▼
                                                           Success Mail  Failure Mail
                                                                 │
                                                                 ▼
                                                        Log en Google Sheets
                                                                 │
                                                                 ▼
                                                        Export CSV a Google Drive
```

---

## Novedades de la última versión

- **Migración a Cloudflare Workers AI (`Flux-1-schnell`)**: Se eliminó la dependencia previa de Pollinations.ai y Cloudinary (que provocaba errores `400 public_id too long` y URLs truncadas). Ahora la imagen se genera directamente en base64 y se inyecta como binario nativo en n8n.
- **Ampliación a 23 fuentes RSS**: Monitoreo de medios técnicos líderes globales e hispanohablantes (The Hacker News, Krebs on Security, CISA, SANS ISC, AWS Security, Kubernetes, OpenAI, MIT Tech Review, Genbeta, MuyLinux, etc.).
- **Prompts visuales contextuales y limpios**: Se rediseñó la generación del prompt visual para producir ilustraciones técnicas, diagramas abstractos y minimalismo corporativo sin texto ilegible incrustado.
- **Triggers duales**: Soporte simultáneo para ejecución programada semanal y disparo manual vía webhook de prueba.
- **Resiliencia de red y aislamiento**: Conexiones RSS con reintentos automáticos (`retryOnFail`) y nodos `Merge` dedicados por categoría para prevenir disparos duplicados.

---

## Requerimientos

| Servicio | Uso | Modalidad / Costo |
|---|---|---|
| **n8n** (v1.x o v2.x) | Orquestador de la automatización | Self-hosted o Cloud |
| **Cloudflare Workers AI** | Generación de imagen primaria con FLUX.1 | Free tier / Pay-as-you-go |
| **Google Cloud Project** | Hojas de cálculo (Sheets), Drive y Gmail | Gratuito |
| **Google Gemini API** | Generación de post primario (`gemini-2.5-flash` / PaLM) | Gratuito (Google AI Studio) |
| **Groq API** | Backup de generación de post (`llama-3.3-70b`) | Gratuito (Free tier) |
| **HuggingFace API** | Backup de imagen (`router.huggingface.co`) | Gratuito |
| **LinkedIn Developer App** | Publicación de contenido vía API | Gratuito |

---

## Instalación

### 1. Importar el workflow en n8n
1. En n8n, ve a **Workflows** → **Add workflow** → Menú `...` → **Import from File**.
2. Selecciona `linkedin-post-automation.json`.

### 2. Configurar credenciales
Por motivos de seguridad, las credenciales no se incluyen en el archivo exportado y deben configurarse en n8n:
- **Cloudflare Workers AI**: Credencial tipo *Header Auth* (`Authorization: Bearer <TU_CLOUDFLARE_API_TOKEN>`).
- **Google Gemini API**: Credencial *Google Gemini (PaLM) Api*.
- **Groq API**: Credencial *Header Auth* (`Authorization: Bearer <TU_GROQ_API_KEY>`).
- **HuggingFace API**: Credencial *Header Auth* (`Authorization: Bearer <TU_HF_TOKEN>`).
- **LinkedIn OAuth2 API**: Scopes `w_member_social`.
- **Gmail OAuth2**: Aplicación web en Google Cloud Console para notificaciones.
- **Google Service Account**: Con permisos de editor sobre tu Google Sheet de registro.
- **Google Drive OAuth2**: Cuenta de usuario para almacenar los respaldos CSV.

### 3. Ajustar variables de usuario
En el workflow, actualiza los nodos que contienen placeholders `YOUR_*`:
- `documentId` y `sheetName`: En **Read Posted Log** y **Log Posted Item**.
- `url` en **Generar Imagen - Cloudflare FLUX.1**: Reemplaza `YOUR_CLOUDFLARE_ACCOUNT_ID` con el ID de tu cuenta de Cloudflare.
- `owner` y `author`: En los nodos de publicación de LinkedIn con tu URN (`urn:li:person:YOUR_LINKEDIN_PERSON_URN`).
- `sendTo`: En **Success Mail** y **Failure Mail** con tu correo personal.

---

## Estructura de Google Sheets

Crea una hoja de cálculo en Google Sheets con una pestaña llamada **`PostedLog`** y los siguientes encabezados en la primera fila:

| Columna | Nombre de Campo | Descripción |
|---|---|---|
| A | `Title` | Título del artículo o noticia original |
| B | `Link` | Enlace permanente (clave única de deduplicación) |
| C | `Category` | Categoría asignada (`Ciberseguridad`, `Infraestructura`, `IA`, `Hardening`) |
| D | `PostedText` | Texto completo publicado en LinkedIn |
| E | `URN` | Identificador de publicación devuelto por LinkedIn |
| F | `PostedAt` | Fecha y hora de publicación |

---

## Fuentes RSS integradas

### Ciberseguridad
- The Hacker News (`https://feeds.feedburner.com/TheHackersNews`)
- Krebs on Security (`https://krebsonsecurity.com/feed/`)
- Dark Reading (`https://www.darkreading.com/rss.xml`)
- The Register Security (`https://www.theregister.com/security/headlines.atom`)
- Wired Security (`https://www.wired.com/feed/category/security/latest/rss`)
- SecurityWeek (`https://www.securityweek.com/feed/`)

### Infraestructura & Cloud
- SANS Internet Storm Center (`https://isc.sans.edu/rssfeed_full.xml`)
- AWS Security Blog (`https://aws.amazon.com/blogs/security/feed/`)
- Kubernetes Official Blog (`https://kubernetes.io/feed.xml`)
- Docker Official Blog (`https://www.docker.com/blog/feed/`)
- HashiCorp Blog (`https://www.hashicorp.com/blog/feed.xml`)

### Inteligencia Artificial
- Google AI Research (`https://blog.research.google/feeds/posts/default`)
- OpenAI News (`https://openai.com/news/rss.xml`)
- TechCrunch AI (`https://techcrunch.com/category/artificial-intelligence/feed/`)
- MIT Technology Review AI (`https://www.technologyreview.com/topic/artificial-intelligence/feed/`)
- Hugging Face Blog (`https://huggingface.co/blog/feed.xml`)

### Hardening, Hardware & Open Source
- CISA Cyber Advisories (`https://www.cisa.gov/cybersecurity-advisories/all.xml`)
- MuySeguridad (`https://www.muyseguridad.net/feed/`)
- Tom's Hardware (`https://www.tomshardware.com/feeds/all`)
- El Chapuzas Informático (`https://elchapuzasinformatico.com/feed/`)
- HardZone (`https://hardzone.es/feed/`)
- Phoronix Linux/Hardware (`https://www.phoronix.com/phoronix-rss.php`)
- RedesZone (`https://www.redeszone.net/feed/`)
- Genbeta (`https://www.genbeta.com/index.xml`)
- MuyLinux (`https://www.muylinux.com/feed/`)
- Ars Technica (`https://feeds.arstechnica.com/arstechnica/index`)

---

## Troubleshooting

| Problema | Causa Raíz | Solución Implementada |
|---|---|---|
| `400 Bad Request` en generación de imágenes | Limitación de URL de Cloudinary/Pollinations al codificar prompts largos | Migrado a **Cloudflare Workers AI (FLUX.1)**, invocación POST directa con payload JSON sin intermediarios. |
| `getaddrinfo ENOTFOUND api-inference.huggingface.co` | Endpoint legado deprecado por Hugging Face | Actualizado a `https://router.huggingface.co/hf-inference/models/<modelo>`. |
| Publicaciones duplicadas en una sola corrida | Múltiples fuentes conectadas en paralelo sin agregador | Implementados nodos `Merge` (modo *Append*) por categoría antes del selector. |
| Timeouts en feeds RSS | Caídas temporales en servidores de noticias | Configurado `retryOnFail` (3 intentos con intervalo de 2s) y `continueRegularOutput`. |
| Pérdida del archivo binario de imagen | Nodos tipo Set que purgan datos binarios | Manejo de binarios mediante nodo Code (`$input.first().binary`). |

---

## Licencia

Uso personal y educativo bajo MIT License. Adapta las fuentes y las credenciales según tu caso de uso.
