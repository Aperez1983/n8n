# 🚨 Alarma de casa conectada al celular (ESP32 + MQTT + n8n + Telegram)

Proyecto para modernizar una alarma antigua (~20-25 años, tipo Prosegur Chile)
y poder **armarla, desarmarla y recibir alertas desde el celular**, sin
reemplazar el panel existente. La alarma sigue funcionando de forma 100%
autónoma: esto es un agregado, no un reemplazo.

## ⚠️ Antes de empezar (importante)

1. **¿Sigue activo el contrato de monitoreo con Prosegur?**
   - Si **SÍ**: no abras el panel. Intervenir el equipo puede violar el
     contrato y generar falsas alarmas a la central. Pide a Prosegur su app
     oficial (Prosegur SMART) o da de baja el servicio primero.
   - Si **NO** (servicio dado de baja hace años): el panel quedó en tu casa y
     normalmente sigue funcionando en modo local. Puedes intervenirlo.
2. **Clave de instalador**: para programar una zona como *keyswitch* vas a
   necesitar la clave de instalador del panel. Los paneles que Prosegur
   instalaba en Chile en los 2000 eran casi siempre paneles estándar
   re-etiquetados: **DSC (PC585 / PC1565 / PC1832)**, **Paradox (Esprit 728)**
   o **Honeywell/Ademco (Vista-10/20)**. Las claves de instalador de fábrica
   (si nunca las cambiaron) son conocidas: DSC `5555`, Paradox `757575` /
   `000000`, Ademco `4112`. Si Prosegur la cambió y no hay forma de
   recuperarla, casi todos los paneles tienen un procedimiento de *reset* de
   hardware (jumper o cortocircuito de pads) documentado en su manual.
3. **Identifica tu panel**: abre la caja metálica (con el sistema desarmado y
   idealmente con la batería y transformador desconectados) y busca la
   serigrafía de la placa (ej: "PC585", "728 ULTRA", "Vista-10SE"). Con eso se
   ajusta el cableado y la programación exacta.

## 🏗️ Arquitectura

```
┌─────────────┐  cables   ┌────────┐  WiFi/MQTT  ┌───────────┐        ┌──────────┐
│ Panel alarma│◄─────────►│ ESP32  │◄───────────►│ Mosquitto │◄──────►│   n8n    │
│ (DSC/Paradox│           │(ESPHome)│            │  (broker) │        │(workflows)│
│  /Ademco)   │           └────────┘             └───────────┘        └────┬─────┘
└─────────────┘                                                            │
   • zona keyswitch ← relé                                                 ▼
   • salida PGM "armado" → GPIO                                      ┌──────────┐
   • salida sirena → GPIO                                            │ Telegram │
                                                                     │ (celular)│
                                                                     └──────────┘
```

- **ESP32 con ESPHome**: un relé da un pulso a una zona programada como
  *keyswitch* (armar/desarmar), y dos entradas leen el estado "armada" (salida
  PGM del panel) y la sirena.
- **Mosquitto**: broker MQTT local (corre en el mismo servidor/Raspberry que n8n).
- **n8n**: dos workflows — uno recibe comandos por Telegram (`/armar`,
  `/desarmar`, `/estado`) y otro te manda notificaciones push cuando la
  alarma suena o cambia de estado.
- **Telegram**: hace de "app" en el celular — gratis, con notificaciones push,
  cifrado y sin publicar nada en tiendas de apps.

## 🛒 Lista de materiales (~$15.000–25.000 CLP)

| Ítem | Detalle | Precio aprox. |
|------|---------|---------------|
| ESP32 DevKit | ESP32-WROOM-32, 38 pines | $5.000–8.000 CLP |
| Módulo relé 1 canal | 5V con optoacoplador | $2.000–3.000 CLP |
| 2× optoacoplador PC817 | Para leer PGM y sirena de forma aislada | $1.000 CLP |
| Conversor buck 12V→5V | Para alimentar el ESP32 desde el AUX del panel | $2.000–3.000 CLP |
| Resistencias | 2× 1kΩ, 2× 10kΩ (limitadoras para los PC817) | $500 CLP |
| Cables y protoboard/placa perforada | — | $3.000 CLP |

Todo se consigue en AliExpress, o en Chile en tiendas como MCI Electronics,
Casa Royal o Vitel.

## 🔌 Cableado (genérico, se afina cuando sepamos el modelo del panel)

Alimentación: la salida **AUX+ / AUX− (12 V)** del panel → conversor buck →
5 V al pin `VIN` del ESP32. Así el ESP32 queda respaldado por la batería de la
alarma.

| Función | Panel | ESP32 |
|---------|-------|-------|
| Armar/desarmar | Zona programada como **keyswitch momentáneo**, en serie con su resistencia de fin de línea | Relé en `GPIO23` (contactos NO del relé en paralelo con la zona) |
| Estado "armada" | Salida **PGM** programada como *Armed Status* (conmuta a negativo al armar) | PGM → R 1kΩ → LED del PC817; fototransistor a `GPIO32` (con pull-up interno) |
| Sirena sonando | Salida **BELL+/BELL−** | BELL+ → R 1kΩ → LED del PC817; fototransistor a `GPIO33` |

Programación del panel (ej. DSC PC585): sección de definición de zonas →
tipo **22 (keyswitch momentáneo)** para la zona elegida; sección PGM → opción
**05 (Armed Status)**. En Paradox y Ademco existen opciones equivalentes.

> 💡 Con keyswitch *momentáneo*, un pulso del relé alterna el estado
> (armada ↔ desarmada). El firmware del ESP32 ya incluye la lógica para que
> `ARMAR` solo pulse si está desarmada y `DESARMAR` solo si está armada, así
> nunca se invierte por error.

## 🚀 Puesta en marcha

### 1. Broker MQTT + n8n

```bash
cd home-alarm
# crear usuario/clave para MQTT
docker run --rm -v $(pwd)/mosquitto:/mosquitto/config eclipse-mosquitto \
  mosquitto_passwd -c -b /mosquitto/config/passwd alarma UNA_CLAVE_SEGURA
docker compose up -d
```

n8n queda en `http://IP_DEL_SERVIDOR:5678`.

### 2. Firmware del ESP32

```bash
pip install esphome
cd esp32
cp secrets.yaml.example secrets.yaml   # editar con tu WiFi y credenciales MQTT
esphome run alarma.yaml                # conectar el ESP32 por USB la primera vez
```

Las siguientes actualizaciones se hacen por WiFi (OTA).

### 3. Bot de Telegram

1. Habla con [@BotFather](https://t.me/BotFather) → `/newbot` → guarda el token.
2. Habla con [@userinfobot](https://t.me/userinfobot) → anota tu **chat ID**.
3. En n8n crea la credencial **Telegram** (con el token) y la credencial
   **MQTT** (host `mosquitto`, puerto `1883`, usuario `alarma` y la clave que
   creaste).

### 4. Workflows

Importa en n8n los dos archivos de `n8n-workflows/`:

- `alarma-control-telegram.json`: comandos `/armar`, `/desarmar`, `/estado`.
  **Edita el nodo "¿Usuario autorizado?" y pon tu chat ID** — es el filtro de
  seguridad que impide que cualquier persona controle tu alarma.
- `alarma-notificaciones.json`: avisos automáticos. **Pon tu chat ID en los
  nodos de Telegram.**

Asigna las credenciales MQTT/Telegram en cada nodo y **activa** ambos workflows.

## 🔐 Seguridad

- El filtro por **chat ID** en el workflow es obligatorio: sin él, cualquiera
  que encuentre tu bot podría desarmar tu casa.
- No expongas Mosquitto (1883) ni n8n (5678) a internet. Telegram funciona con
  conexiones salientes, así que **no necesitas abrir ningún puerto**.
- Usa claves fuertes en MQTT y en el fallback AP del ESP32.
- La alarma sigue siendo autónoma: si se cae WiFi/servidor, el panel, la
  sirena y el teclado original funcionan igual que siempre.

## 📋 Próximos pasos

1. Confirmar si el contrato con Prosegur está dado de baja.
2. Abrir el panel y **fotografiar la placa** para identificar el modelo exacto.
3. Con el modelo, afinar la programación del keyswitch/PGM y el cableado.
4. Montar, flashear y probar primero con la sirena desconectada.
