# Automatización del seguimiento de presupuestos con IA Agéntica

Trabajo Final Integrador — Diplomatura en Diseño de Procesos de Negocios con Plataformas Visuales de Automatización e Inteligencia Artificial (IA) Agéntica · FCE-UBA · Cohorte 2026

**Integrantes:** Vázquez Fernández, Lucas · Palacios, Alejo Martín · Grunow, Lucas

---

## ¿Qué proceso transforma este proyecto?

Un taller de reparación de electrodomésticos hace el diagnóstico de un equipo y arma el presupuesto correspondiente. A partir de ahí, hoy todo es manual: alguien tiene que enviar el mail, revisar la bandeja de entrada para ver si el cliente respondió, actualizar a mano un registro informal del estado de cada presupuesto, y acordarse de mandar recordatorios si el cliente no contesta.

Este proyecto automatiza por completo ese seguimiento con **tres workflows de n8n**, usando una única Google Sheet como fuente de verdad y **Google Gemini** como único punto de inteligencia artificial del proceso, dedicado exclusivamente a interpretar en lenguaje natural la respuesta del cliente cuando contesta el mail directamente.

## ¿Para qué sirve?

- Elimina la revisión manual constante de la bandeja de entrada.
- Envía los presupuestos automáticamente en cuanto Administración los marca como listos.
- Manda recordatorios y marca vencimientos sin que nadie tenga que acordarse.
- Clasifica la respuesta del cliente (aprueba, rechaza, consulta, o algo ambiguo) e interviene una persona solo cuando realmente hace falta.
- Centraliza todo el historial del proceso en un único lugar, trazable y auditable.

## Los tres workflows

### 1. `Workflow_1_Envio_de_Presupuestos.json` — Envío automático

- **Disparador:** Schedule cada 30 minutos.
- **Flujo:** `Schedule Trigger` → `Buscar Listo para Enviar` (Google Sheets, filtra `Estado = Listo para enviar`) → `Loop Over Items` → `Enviar Gmail` (con botones Aceptar / Rechazar) → `Actualizar Estado a Esperando Respuesta`.
- **Qué actualiza:** `Estado → Esperando respuesta`, `Fecha Envío`, `Última Actualización`.

### 2. `Workflow_2_Seguimiento_de_Presupuestos.json` — Seguimiento y recordatorios

- **Disparador:** Schedule diario a las 9:00.
- **Flujo:** `Schedule Diario 09:00` → `Buscar Esperando Respuesta` → `Loop Over Items` → `Calcular Dias Transcurridos` (nodo Code, calcula días desde `Fecha Envío` sin reiniciar nunca el conteo) → `Switch Dias`, con tres ramas:
  - `≥ 3 días` y sin Recordatorio 1 enviado → `Gmail Recordatorio 1` → `Actualizar Recordatorio 1`.
  - `≥ 7 días` y sin Recordatorio 2 enviado → `Gmail Recordatorio 2` → `Telegram Recordatorio 2` → `Actualizar Recordatorio 2`.
  - `≥ 15 días` → `Actualizar Estado Vencido` (`Estado → Presupuesto vencido`) → `Telegram Vencido`.

### 3. `Workflow_3_Procesamiento_de_Respuestas.json` — Procesamiento inteligente

Tiene **dos entradas independientes**:

- **Entrada A — Botones del mail:** `Webhook Botones` (GET, path `respuesta-presupuesto`) → `Actualizar Respuesta Botones` (`Estado → Presupuesto aprobado / rechazado` según el query param `accion`) → `Telegram Botones` → `Respond to Webhook` (página de confirmación para el cliente).
- **Entrada B — Respuesta por mail:** `Gmail Trigger` → `Extraer Orden de Servicio` (nodo Code, regex `OS[_-]?\d+` sobre el asunto) → `Buscar Orden` → `Verificar Esperando Respuesta` (IF, corta el flujo si el presupuesto ya no está pendiente) → `Clasificar con Gemini` (HTTP Request a la API de Gemini, modelo `gemini-2.5-flash`) → `Extraer Clasificacion` → `Switch Clasificacion`, con cuatro ramas: `APROBADO`, `RECHAZADO`, `CONSULTA` e `INDETERMINADO`. Las dos últimas nunca resuelven el caso solas: siempre derivan a `Estado → Pendiente gestión administrativa` para que decida una persona.

## Estados del proceso

`Listo para enviar` → `Esperando respuesta` → (`Presupuesto aprobado` | `Presupuesto rechazado` | `Presupuesto vencido` | `Pendiente gestión administrativa`)

## Stack

| Herramienta | Rol |
|---|---|
| **n8n** | Orquesta los tres workflows |
| **Google Sheets** | Fuente única de verdad del estado de cada presupuesto |
| **Gmail** | Envío de presupuestos y recepción de respuestas del cliente |
| **Telegram** | Notificaciones internas a Administración |
| **Google Gemini** | Clasifica en lenguaje natural la respuesta del cliente |

## Estructura del repositorio

```
├── README.md
├── workflows/
│   ├── Workflow_1_Envio_de_Presupuestos.json
│   ├── Workflow_2_Seguimiento_de_Presupuestos.json
│   └── Workflow_3_Procesamiento_de_Respuestas.json
├── presentacion/
│   └── TP_Automatizacion_Presupuestos.pptx
└── informe/
    └── Informe_TrabajoPractico_Final.docx
```

## Cómo usarlo

### 1. Preparar la Google Sheet

Creá una Google Sheet con una pestaña que tenga exactamente estas columnas (en cualquier orden, pero con estos nombres):

`Orden de Servicio` · `Cliente` · `Email` · `Teléfono` · `Equipo` · `Importe Presupuesto` · `Estado` · `Fecha Envío` · `Fecha Respuesta Cliente` · `Recordatorio 1 Enviado` · `Fecha Recordatorio 1` · `Recordatorio 2 Enviado` · `Fecha Recordatorio 2` · `Fecha Vencimiento` · `Motivo de pausa` · `Última Actualización`

### 2. Configurar credenciales en n8n

- **Google Sheets OAuth2** y **Gmail OAuth2**: conectá la cuenta de Google que va a usar el taller.
- **Telegram**: creá un bot con [@BotFather](https://t.me/BotFather) y conseguí el `chat_id` donde quieras recibir los avisos.
- **Google Gemini**: generá una API key gratuita en [aistudio.google.com/apikey](https://aistudio.google.com/apikey) y cargala como credencial de tipo *Query Auth* (parámetro `key`) en el nodo `Clasificar con Gemini` del Workflow 3.

### 3. Importar los workflows

Para cada uno de los 3 archivos `.json`: en n8n, menú (`⋮`) → **Import from File**. Después, en cada workflow:

- Reasigná el `documentId` y el `sheetName` de los nodos de Google Sheets a tu propia planilla.
- Reasigná las credenciales de Gmail, Google Sheets y Telegram a las tuyas.
- Reemplazá el `chat_id` de Telegram en los nodos correspondientes por el tuyo.
- En el Workflow 1, actualizá la URL de los botones del mail (`https://TU-INSTANCIA.app.n8n.cloud/webhook/respuesta-presupuesto...`) para que apunte a tu propia instancia de n8n.

### 4. Activar

Activá los tres workflows (toggle **Active**). El Workflow 1 y el Workflow 3 (que incluye el Gmail Trigger y el Webhook) necesitan estar activos para que las URLs de producción respondan.

## Limitaciones conocidas

- La extracción de la Orden de Servicio depende de que el cliente no borre el asunto original del mail al responder.
- El plan gratuito de n8n tiene un límite bajo de ejecuciones mensuales, algo a tener en cuenta si se usa en producción real.
- La clasificación de Gemini depende de la disponibilidad del servicio (pueden aparecer errores 503 transitorios por alta demanda).
- El sistema asume que los datos de cada Orden de Servicio se cargan de forma consistente en la planilla.

## Cómo se construyó

El diseño del proceso de negocio (estados, disparadores, filosofía de "todo lo indispensable, nada más") fue una decisión del equipo. La implementación técnica en n8n se hizo con asistencia de Claude (Anthropic), de forma iterativa: cada workflow se probó nodo por nodo con datos reales, corrigiendo errores concretos de la plataforma a medida que aparecían (formatos de fecha, versiones de modelos de IA dadas de baja, comportamientos no obvios de ciertos nodos, entre otros). El detalle de esos problemas y sus soluciones está documentado en la sección de Metodología del informe.

## Licencia

MIT — proyecto académico, de uso libre para fines educativos.
