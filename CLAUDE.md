# CLAUDE.md — Proyecto Asistente Wilson

> Contexto del proyecto para Claude. Léelo al iniciar cualquier sesión sobre este repo.
> Documentación extendida: `workflows-wilson/MANUAL-ASISTENTE-WILSON.md`.

## Qué es este proyecto

Este fork de n8n se usa como **repositorio de trabajo de Wilson (Andrés Pérez)** para su
**bot de WhatsApp con IA** que gestiona pendientes familiares. Los entregables reales viven en
`workflows-wilson/` (workflows JSON de n8n + manual). **No** se desarrolla el core de n8n aquí.

- Usuario: Wilson (Andrés Pérez), Chile (zona horaria America/Santiago). Hablarle en español, tono cercano, paso a paso (perfil no-técnico pero aprende rápido).
- Rama de trabajo: `claude/daily-reminders-mobile-jsisf8`.

## Arquitectura

```
WhatsApp (Cloud API, número de prueba +1 555 199 9974)
  → n8n autoalojado (https://n8n.mamutagencia.com, VPS Contabo, /opt/n8n, Docker: n8n + Caddy)
  → AI Agent (Claude Haiku 4.5) + Simple Memory (ventana 6)
  → Data Table "pendientes" (tarea, fecha AAAA-MM-DD, estado, categoria, persona)
  → Responde por WhatsApp
```

Workflow **principal** (debe estar Published; solo UNO de WhatsApp a la vez):
`Asistente Wilson — WhatsApp (multimedia)` — id `WilsonWAMultimedia` — maneja texto, imagen
(visión de Claude), PDF (Extract From File) y voz (Whisper de OpenAI). Fuente:
`workflows-wilson/asistente-wilson-whatsapp-multimedia.json`.

## Datos clave

- Meta App "Asistente Wilson": App ID `3243505939165635`, business `1370436766914450` (Mamut Agencia Digital II)
- Phone Number ID: `1195007940358395` · WABA (Test): `1995327248016431`
- Token de WhatsApp: PERMANENTE, vía system user `n8n-whatsapp-bot` (no usar el de 24h)
- Autorizados (filtro "Solo autorizados"): Wilson `56956392228`, señora `56979882647` (hijas pendientes)
- Datos fijos en el system prompt: Maite → 7° Básico B, Isidora → 3° Básico D
- Fechas: n8n inyecta fecha de Chile + calendario de 21 días en cada prompt; Claude NUNCA debe calcular días de la semana; `tarea` sin fechas, la fecha va en columna `fecha`
- Categorías: Hogar/Trabajo/Colegio/Otros + personalizadas (reutilizar nombre exacto)

## Flujo de despliegue (importante)

Los workflows se editan EN ESTE REPO y se despliegan al servidor por SSH. No hay acceso directo
al n8n del usuario desde Claude; el usuario ejecuta los comandos (guiarlo: prompt `PS C:\...` = su
PC, `root@vmi3364873` = servidor; conectar con `ssh root@n8n.mamutagencia.com`).

Script de actualización en el servidor (si existe): `/opt/n8n/actualizar-bot.sh`. Patrón manual:
descargar el JSON crudo desde esta rama de GitHub → `n8n export:workflow` del actual para extraer
credenciales → reinyectarlas con python al JSON nuevo → `docker compose cp` → `n8n import:workflow`
→ `docker compose restart n8n` → el usuario re-publica en la UI (el import desactiva el workflow).

Gotchas conocidos:
- `n8n execute` por CLI falla ("Task Broker port 5679 in use") → ejecutar workflows de setup desde la UI
- El import exige campo `id` en el JSON y sobreescribe por id; las credenciales NO viajan en el JSON
- NUNCA usar "Execute workflow" para probar WhatsApp (rompe el webhook); probar desde el celular
- Si el bot deja de responder: re-publicar (Unpublish → Publish) para re-registrar el webhook
- El campo de token en la consola de Meta siempre se ve vacío al recargar: es normal
- Costos: usar Haiku; "Buscar pendientes" filtra estado=pendiente; no reenviar PDFs repetidos

## Pendientes del proyecto

Ver checklist en `workflows-wilson/MANUAL-ASISTENTE-WILSON.md` (sección 11): agregar números de
las hijas (Meta + filtro), limpiar duplicados de la tabla, auto-recarga Anthropic/OpenAI,
número propio (chip ya comprado), memoria de datos editable, espejo Obsidian.
