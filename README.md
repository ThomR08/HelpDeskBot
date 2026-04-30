# 🤖 HelpDeskBot

Bot de soporte técnico conversacional para **Telegram**, construido con **n8n**, **Google Gemini** y **Google Sheets** como base de datos. Permite a los usuarios crear y consultar solicitudes de soporte directamente desde su chat de Telegram, sin necesidad de aplicaciones adicionales.

---

## ✨ Características

- **Registro automático de usuarios** — Los nuevos usuarios se registran solos al enviar su primer mensaje.
- **Interpretación con IA** — Google Gemini interpreta los mensajes en lenguaje natural y navega los menús por el usuario.
- **Gestión de tickets** — Creación, consulta de estado y listado de solicitudes propias.
- **Tres tipos de solicitud** — Soporte técnico, solicitud administrativa y consulta general.
- **Prioridad automática** — La IA asigna la prioridad del ticket según la descripción del usuario.
- **Reportes** — Resumen de solicitudes por estado y tipo.
- **Prompts configurables** — Los prompts del bot se administran desde una hoja de Google Sheets, sin tocar el flujo.
- **Logging completo** — Cada interacción queda registrada con usuario, acción y timestamp.

---

## 🧱 Stack tecnológico

| Componente | Tecnología |
|---|---|
| Automatización | [n8n](https://n8n.io) |
| Bot de mensajería | Telegram Bot API |
| Modelo de IA | Google Gemini (PaLM API) |
| Base de datos | Google Sheets |

---

## 📋 Requisitos previos

- Instancia de **n8n** activa (self-hosted o cloud)
- **Bot de Telegram** creado con [@BotFather](https://t.me/BotFather)
- Cuenta de **Google** con acceso a Google Sheets
- **API key de Google Gemini** (Google AI Studio)

---

## 🗂️ Estructura de Google Sheets

El flujo utiliza un único spreadsheet con las siguientes hojas:

### `SOLICITUDES`
Tabla principal de tickets.

| Columna | Descripción |
|---|---|
| `id_ticket` | Número incremental del ticket |
| `tipo` | Soporte técnico / Solicitud administrativa / Consulta general |
| `prioridad` | Asignada automáticamente por la IA |
| `descripcion` | Descripción escrita por el usuario |
| `estado` | `Sin resolver`, `En proceso`, `Resuelto` |
| `creado_por` | ID de Telegram del usuario |
| `fecha_creacion` | Timestamp de creación |

### `USUARIOS`
Registro de usuarios del bot.

| Columna | Descripción |
|---|---|
| `telegram_user` | ID numérico de Telegram |
| `nombre` | Nombre completo (first + last name) |
| `rol` | `user` o `admin` |
| `activo` | `true` / `false` |

### `LOGS`
Auditoría de cada interacción.

| Columna | Descripción |
|---|---|
| `timestamp` | Fecha y hora de la acción |
| `telegram_user` | ID de Telegram |
| `pantalla` | Número de pantalla activa |
| `opcion` | Opción seleccionada |
| `resultado` | Descripción de la acción realizada |

### `PROMTS`
Prompts dinámicos del bot. Permite configurar el comportamiento de la IA sin editar el flujo de n8n.

| Columna | Descripción |
|---|---|
| `pantalla` | Número de pantalla (0–5) |
| `opcion` | Número de opción dentro de la pantalla |
| `promt` | Instrucción del sistema para Gemini |

### `HISTORIAL`
Historial de cambios de estado de los tickets.

---

## 🚀 Instalación

### 1. Clonar el flujo en n8n

Importar el archivo `HelpDeskBot.json` en n8n:

```
Menú principal → Import → From File → seleccionar HelpDeskBot.json
```

### 2. Configurar credenciales

En n8n, crear las siguientes credenciales:

**Telegram Bot API**
1. Ir a *Credentials → New → Telegram API*
2. Ingresar el token obtenido desde [@BotFather](https://t.me/BotFather)

**Google Sheets (OAuth2)**
1. Ir a *Credentials → New → Google Sheets OAuth2 API*
2. Autenticar con la cuenta de Google que tiene acceso al spreadsheet

**Google Gemini (PaLM API)**
1. Ir a *Credentials → New → Google PaLM API*
2. Ingresar la API key obtenida desde [Google AI Studio](https://aistudio.google.com)

### 3. Configurar el Spreadsheet

1. Crear un Google Sheets con las cinco hojas descritas en la sección anterior: `SOLICITUDES`, `USUARIOS`, `LOGS`, `PROMTS`, `HISTORIAL`
2. Copiar el ID del spreadsheet desde la URL:
   ```
   https://docs.google.com/spreadsheets/d/[ID_AQUI]/edit
   ```
3. En cada nodo de Google Sheets dentro del flujo, actualizar el `documentId` con el nuevo ID

### 4. Activar el flujo

En n8n, hacer clic en el toggle **Active** del flujo. El webhook de Telegram quedará escuchando automáticamente.

---

## 💬 Menú del bot

Al escribir cualquier mensaje al bot, el usuario recibe el menú principal:

```
Menú principal:
0. Ayuda
1. Crear solicitud
2. Consultar estado de solicitud
3. Mis solicitudes
4. Reportes
5. Configuración
```

### Pantalla 1 — Crear solicitud

El usuario selecciona el tipo:
```
1. Soporte técnico
2. Solicitud administrativa
3. Consulta general
9. Cancelar
```

Luego describe el problema en texto libre. La IA extrae la descripción y asigna la prioridad. El ticket se guarda en la hoja `SOLICITUDES` con un ID incremental y estado `Sin resolver`.

### Pantalla 2 — Consultar estado

El usuario ingresa el número de ticket. El bot retorna el estado actual, la prioridad y la fecha de creación.

### Pantalla 3 — Mis solicitudes

Lista todos los tickets creados por el usuario con su estado actual.

### Pantalla 4 — Reportes

Resumen de solicitudes agrupadas por estado y tipo (disponible según el rol del usuario).

### Pantalla 5 — Configuración

Opciones de administración (en desarrollo).

---

## 🔄 Flujo interno

```
Telegram Trigger
     │
     ▼
Buscar usuario en USUARIOS
     │
 ¿Existe?
  ├─ No → Registrar usuario nuevo → Log: USUARIO REGISTRADO
  └─ Sí ──────────────────────────────────────────────────┐
                                                           │
     ┌─────────────────────────────────────────────────────┘
     ▼
Obtener último LOG del usuario (estado de sesión)
     │
     ▼
Cargar prompt desde PROMTS (por pantalla + opción)
     │
     ▼
Enviar a Google Gemini
     │
     ▼
Parsear respuesta JSON
{ pantalla, opcion, mensaje, done, descripcion_solicitud, prioridad }
     │
 ¿done?
  ├─ No → Enviar mensaje parcial al usuario
  └─ Sí → Switch por pantalla (0–5)
               │
       ┌───────┴────────┐
       ▼                ▼
  Operaciones     Respuesta
  en Sheets    en Telegram
       │
       ▼
   Log en LOGS
```

---

## ⚙️ Cómo funciona la IA

El bot usa Google Gemini como motor de interpretación conversacional. En cada mensaje del usuario, se le envía a Gemini:

- El **prompt del sistema** correspondiente a la pantalla y opción actuales (cargado desde la hoja `PROMTS`)
- El **nombre del usuario**
- La **pantalla activa** y la **opción actual**
- El **texto del mensaje** del usuario

Gemini devuelve siempre un JSON con esta estructura:

```json
{
  "pantalla": 1,
  "opcion": 2,
  "mensaje": "Texto de respuesta para el usuario",
  "done": true,
  "descripcion_solicitud": "El usuario reporta que...",
  "prioridad": "Alta"
}
```

El campo `done` indica si la acción está completa o si el bot debe seguir recolectando información antes de ejecutar la operación.

---

## 📁 Estructura del repositorio

```
HelpDeskBot/
├── HelpDeskBot.json        # Flujo completo de n8n (importable)
└── README.md               # Este archivo
```

---

## 🐛 Problemas conocidos

- El modelo de Gemini configurado en el flujo (`gemini-robotics-er-1.5-preview`) puede no estar disponible. Se recomienda reemplazarlo por `gemini-1.5-flash` o `gemini-1.5-pro`.
- El estado de la conversación se recupera leyendo el **último registro de la hoja LOGS**. Si el spreadsheet tiene muchas filas, esto puede volverse lento. Para alta concurrencia se recomienda migrar a una base de datos real (PostgreSQL, Supabase, etc.).
- El ID de ticket se genera leyendo el último valor de la hoja `SOLICITUDES` y sumando 1. No es atómico, por lo que dos usuarios simultáneos podrían generar IDs duplicados.

---

## 🔧 Personalización

Para modificar el comportamiento del bot sin editar el flujo de n8n, editar directamente la hoja `PROMTS` del spreadsheet. Cada fila corresponde a una combinación de pantalla + opción, y contiene el prompt que se enviará a Gemini.

---

## 📄 Licencia

MIT
