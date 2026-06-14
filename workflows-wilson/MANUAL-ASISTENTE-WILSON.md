# 🤖 Asistente Wilson — Manual del Proyecto

> Bot de WhatsApp con IA (Claude) para gestionar pendientes de la familia.
> Documento de referencia / runbook. Última actualización: junio 2026.

---

## 1. ¿Qué es y qué hace?

Un asistente personal por **WhatsApp** que:
- Entiende **texto, imágenes (las ve), PDFs (los lee) y notas de voz (las transcribe)**.
- Guarda, lista y completa **pendientes/tareas/evaluaciones** en una libreta permanente.
- Clasifica por categoría (Hogar, Trabajo, Colegio, Otros) y por persona.
- Responde solo a **números autorizados** (familia).

**Flujo general:**
```
📱 WhatsApp → ☁️ Servidor (n8n) → 🧠 Claude → 📒 Libreta (tabla) → 📱 Respuesta
```

---

## 2. Infraestructura (dónde vive todo)

| Pieza | Detalle |
|---|---|
| **Servidor** | Contabo **Cloud VPS 20** · Ubuntu · backup diario activado |
| **Acceso al servidor** | `ssh root@n8n.mamutagencia.com` (contraseña root en tu gestor) |
| **Carpeta de n8n** | `/opt/n8n` (Docker Compose: n8n + Caddy) |
| **Dominio del panel** | `https://n8n.mamutagencia.com` (HTTPS automático vía Caddy) |
| **DNS** | Subdominio `n8n` (registro A) apunta al VPS — gestionado en Webempresa |
| **Código respaldado** | GitHub repo `aperez1983/n8n`, rama `claude/daily-reminders-mobile-jsisf8`, carpeta `workflows-wilson/` |

> ⚠️ El panel `n8n.mamutagencia.com` puede mostrar una alerta de "sitio peligroso" en Chrome: es un **falso positivo** de Google con subdominios nuevos. Entra con "Detalles → visitar de todos modos". No afecta al bot.

---

## 3. Cuentas y credenciales (sin secretos aquí)

| Servicio | Para qué | Dónde está la llave |
|---|---|---|
| **Anthropic (Claude)** | Cerebro del bot (modelo **Claude Sonnet 4.6**) | Credencial `Anthropic account` en n8n · prepago en console.anthropic.com |
| **WhatsApp Cloud API** | Recibir/enviar mensajes | Credenciales `WhatsApp account` (enviar) y `WhatsApp OAuth account` (recibir) en n8n |
| **OpenAI (Whisper)** | Transcribir notas de voz | Credencial `OpenAI account` en n8n · prepago en platform.openai.com |

**Token permanente de WhatsApp:** generado con el **usuario del sistema `n8n-whatsapp-bot`** (caducidad "Nunca"). Si alguna vez falla la conexión de envío, se regenera ahí y se pega en `WhatsApp account`.

---

## 4. Datos clave de Meta / WhatsApp

| Dato | Valor |
|---|---|
| **App** | Asistente Wilson |
| **App ID (Client ID)** | `3243505939165635` |
| **Portafolio comercial** | Mamut Agencia Digital II (`business_id 1370436766914450`) |
| **Número del bot (prueba)** | `+1 555 199 9974` |
| **Phone Number ID** | `1195007940358395` |
| **WhatsApp Business Account ID (Test)** | `1995327248016431` |
| **Modo de la app** | Desarrollo (correcto para uso familiar) |

> El número de prueba permite **hasta 5 destinatarios verificados**. Para más gente o uso público, hay que registrar un **número propio** (chip dedicado).

---

## 5. Workflows en n8n

| Workflow | ID | Función |
|---|---|---|
| **Asistente Wilson — WhatsApp (multimedia)** | `WilsonWAMultimedia` | **EL PRINCIPAL** (debe estar *Published* 🟢). Texto + imagen + PDF + voz |
| Asistente Wilson — WhatsApp | `WilsonWhatsAppBot1` | Versión anterior (solo texto). Mantener *Unpublished* |
| Asistente Wilson — Bot de Pendientes | `NpkEkgvbyG0hJ3t9` | Chat de prueba en el navegador (sin WhatsApp) |
| Setup — Crear tabla pendientes | `WilsonSetupTabla001` | Se usó una vez para crear la tabla |

> ⚠️ **Solo UN workflow de WhatsApp puede estar publicado a la vez** (Meta permite un webhook por app). El activo es el **multimedia**.

**Nodos del multimedia:** WhatsApp Trigger → filtro "Solo autorizados" → Switch por tipo (texto/imagen/PDF/audio) → descarga y procesa según el tipo → AI Agent (Claude + memoria + 3 herramientas) → Responder por WhatsApp.

---

## 6. La libreta: tabla `pendientes`

Columnas: `tarea` (solo descripción) · `fecha` (AAAA-MM-DD) · `estado` (pendiente/completada) · `categoria` · `persona`.

- **Los datos NO se pierden** — la tabla es permanente (disco + backup diario).
- El bot **lee toda la tabla** cada vez que preguntas (no depende de la memoria de conversación).
- La **memoria simple** solo recuerda el chat reciente (eso sí se olvida; es normal).

---

## 7. Datos fijos del bot (en sus instrucciones)

Grabados "a fuego" en el System Message del AI Agent (nunca se olvidan):
- **Maite → 7° Básico B**
- **Isidora → 3° Básico D**

> Si cambian (ej. cambio de año), hay que **editarlos a mano** en el System Message del nodo "AI Agent".

**Manejo de fechas (importante):** el bot recibe en cada mensaje la **fecha real de Chile** + un **calendario de referencia de 21 días** ya calculado por n8n. Así nunca calcula mal el día de la semana (Claude solo no es confiable para eso).

---

## 8. Cómo agregar a la familia (2 partes obligatorias)

**Parte A — En Meta** (porque es número de prueba): Configuración de la API → "Para" → "Administrar lista de números" → agregar cada número con su código de verificación.

**Parte B — En el bot** (el portero): editar el nodo **"Solo autorizados"** del workflow, en la lista:
```
['56956392228','56979882647','569XXXXXXXX','569XXXXXXXX']
```
(tu número + señora + hijas, formato código país + número, sin `+`).

> Faltan ambas partes para que un número nuevo reciba respuesta.

Autorizados actuales:
- Wilson: `56956392228`
- Señora: `56979882647`
- Maite: *(pendiente)*
- Isidora: *(pendiente)*

---

## 9. Tareas de mantenimiento comunes

```bash
# Conectarse al servidor
ssh root@n8n.mamutagencia.com

# Reiniciar n8n (limpia la memoria temporal del chat)
cd /opt/n8n && docker compose restart n8n

# Ver que los contenedores estén arriba
cd /opt/n8n && docker compose ps
```

- **Re-publicar el webhook** (si el bot deja de responder a mensajes nuevos): abrir el workflow → Unpublish → esperar 3s → Publish.
- **NO usar "Execute workflow"** para probar WhatsApp (rompe el webhook). Probar siempre escribiendo desde el celular.
- **Actualizar un workflow** desde GitHub: se descarga el JSON desde la rama y se importa con `n8n import:workflow` (ver historial de comandos).

---

## 10. Costos aproximados (mensuales)

| Concepto | Costo |
|---|---|
| Contabo Cloud VPS 20 | ~$7,20 USD/mes |
| Backup automático Contabo | ~€2/mes |
| Claude (Anthropic) | Centavos (prepago, según uso) |
| OpenAI Whisper (voz) | ~$0,006/min (centavos) |
| WhatsApp (conversaciones iniciadas por el usuario) | Gratis |

---

## 11. Mejoras pendientes / futuras

- [ ] Agregar números de **Maite e Isidora** (Parte A + B).
- [ ] Registrar **número propio** (chip dedicado) para quitar el límite de 5 destinatarios.
- [ ] **Memoria de datos editable**: tabla de perfiles para que el bot agregue/cambie datos (cursos, cumpleaños) por WhatsApp, sin editar el código.
- [ ] **Espejo en Obsidian** (ver los pendientes en notas).
- [ ] **Filtrado de búsqueda** si la tabla crece a miles de filas.
- [ ] **Recordatorios proactivos** (requiere plantillas aprobadas por Meta).

---

## 12. Lecciones aprendidas (gotchas)

- El campo del **token en la pantalla de Meta siempre se ve vacío** al recargar: es normal, no afecta.
- El **token temporal dura 24h** → usar siempre el **permanente** (usuario del sistema).
- Claude **no calcula bien días de la semana** → se le inyecta calendario ya calculado.
- **No incrustar el día/fecha en el texto** de la tarea → usar la columna `fecha`.
- Restricciones de WhatsApp suelen ser por el **sitio web del perfil**: debe estar accesible y describir el negocio.

---

*Construido en una sesión épica. Todo respaldado en GitHub. 🚀*
