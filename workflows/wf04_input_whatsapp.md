# WF-04: Procesamiento Input WhatsApp de Domin

## Objetivo

WF-04 recibe mensajes de texto libre de Domin (comercial) por WhatsApp y los convierte en datos estructurados que se guardan en la BD de Comunicaciones de Notion. Esto permite auditar y hacer seguimiento de todas las interacciones comerciales.

---

## Trigger

**Execute Workflow Trigger** - Llamado por WF-00 cuando `event_type == "whatsapp_domin_input"`

Input: Mensaje de WhatsApp
```json
{
  "event_type": "whatsapp_domin_input",
  "data": {
    "message_text": "Llamé a García López, muy interesado, dijo que llamo el lunes",
    "whatsapp_from": "+34612345678",
    "timestamp_mensaje": "2026-04-05T16:25:10Z"
  }
}
```

---

## Nodos

### 1. Execute Workflow Trigger
**Tipo:** Trigger
**Posición:** (100, 200)

---

### 2. Code: Extraer Menciones
**Tipo:** Code (JavaScript)
**Posición:** (350, 200)

Extrae información útil del mensaje de texto libre (números de teléfono, palabras clave).

**Código:**
```javascript
const input = $input.first().json;
const message_text = input.data?.message_text || '';

// Extrae números de teléfono si están en el texto
const phone_pattern = /\+?\d{10,15}/g;
const telefonos = message_text.match(phone_pattern) || [];

// Buscar nombres de clientes mencionados (heurística simple)
const palabras = message_text.split(/\s+/);

return [{
  json: {
    message_text: message_text,
    telefonos_encontrados: telefonos,
    palabras_clave: palabras,
    timestamp: new Date().toISOString()
  }
}];
```

**Output:** Datos pre-procesados del mensaje

---

### 3. HTTP POST: Claude API - Estructuración
**Tipo:** HTTP Request
**Posición:** (600, 200)

Envía el mensaje a Claude para extraer estructura JSON.

**Endpoint:** `https://api.anthropic.com/v1/messages`

**Prompt:**
```
Analiza este mensaje de WhatsApp de un comercial y extrae estructura:

Mensaje: "Llamé a García López, muy interesado, dijo que llama el lunes"

Devuelve JSON con:
- canal: 'Llamada' / 'WhatsApp' / 'Email' / 'Presencial' / 'SMS'
- resultado: 'Positivo' / 'Neutral' / 'Negativo' / 'Pendiente'
- resumen: 1 línea del contenido
- proxima_accion: Qué hacer después (ej: llamar, enviar comparativa, seguimiento)
- fecha_proxima_accion: Fecha (YYYY-MM-DD) estimada para próxima acción
- notas: Observaciones relevantes
- cliente_nombre: Si menciona nombre de cliente
- telefono: Si menciona teléfono

Respuesta formato JSON valido.
```

**Output:** JSON estructurado desde Claude

---

### 4. Code: Validar Estructura
**Tipo:** Code (JavaScript)
**Posición:** (850, 200)

Valida y normaliza la respuesta de Claude.

**Código:**
```javascript
const claude_response = $input.first().json;
const content = claude_response.content[0].text;

let structured = {};
try {
  structured = JSON.parse(content);
} catch (e) {
  const jsonMatch = content.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    structured = JSON.parse(jsonMatch[0]);
  }
}

return [{
  json: {
    structured_input: {
      canal: structured.canal || 'Presencial',
      resultado: structured.resultado || 'Pendiente',
      resumen: structured.resumen || '',
      proxima_accion: structured.proxima_accion || '',
      fecha_proxima_accion: structured.fecha_proxima_accion || new Date(Date.now() + 7*24*60*60*1000).toISOString().split('T')[0],
      notas: structured.notas || '',
      cliente_nombre: structured.cliente_nombre || '',
      telefono: structured.telefono || ''
    },
    timestamp: new Date().toISOString()
  }
}];
```

**Validaciones:**
- Si canal no está en SELECT válido, usa default "Presencial"
- Si resultado no está en SELECT válido, usa default "Pendiente"
- Si fecha_proxima_accion no existe, calcula +7 días

**Output:** Datos estructurados y validados

---

### 5. HTTP POST: Crear Comunicación en Notion
**Tipo:** HTTP Request
**Posición:** (1100, 200)

Crea nuevo registro en la database Comunicaciones.

**Endpoint:** `https://api.notion.com/v1/pages`

**Database:** `9ff3c108-0907-4267-b49d-89254ef20b58` (Comunicaciones)

**Properties:**
```json
{
  "ID Comunicación": { "title": "Domin - {{timestamp}}" },
  "Tipo": { "select": "{{canal}}" },
  "Dirección": { "select": "Saliente" },
  "Asunto": { "rich_text": "{{resumen}}" },
  "Contenido": { "rich_text": "{{message_text}}" },
  "Resultado": { "select": "{{resultado}}" },
  "Proxima Accion": { "rich_text": "{{proxima_accion}}" },
  "Fecha Proxima Accion": { "date": { "start": "{{fecha_proxima_accion}}" } },
  "Realizado Por": { "rich_text": "Domin" },
  "Notas Internas": { "rich_text": "{{notas}}" }
}
```

**Output:** ID de comunicación hcreada

---

### 6. Code: Confirmar Creación
**Tipo:** Code (JavaScript)
**Posición:** (1350, 200)

Prepara respuesta de confirmación a Domin.

**Output:**
```json
{
  "comunicacion_id": "UUID",
  "cliente_nombre": "García López",
  "proxima_accion": "Seguimiento el lunes",
  "fecha_proxima": "2026-04-07",
  "message": "✅ Registrado. Seguimiento el lunes"
}
```

---

### 7. HTTP POST: WhatsApp - Confirmar a Domin
**Tipo:** HTTP Request
**Posición:** (1600, 200)

Envía confirmación a Domin por WhatsApp.

**To:** `{{env.WHATSAPP_DOMIN_NUMBER}}`

**Message:** `✅ Registrado. [próxima acción]`

---

## Flujo de Ejecución

```
┌─────────────────────────────────┐
│ Execute Workflow Trigger       │
│ Message: "Llamé a García López" │
└──────────────┬──────────────────┘
               │
               ▼
┌────────────────────────────┐
│ Code: Extraer Menciones     │
│ Busca teléfonos, palabras   │
└──────────────┬─────────────┘
               │
               ▼
┌──────────────────────────────┐
│ HTTP: Claude API             │
│ Extrae: canal, resultado,   │
│ acciój, notas, etc.         │
└──────────────┬─────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Code: Validar Estructura     │
│ Asegura que campos tengan   │
│ valores válidos             │
└──────────────┬───────────────┘
               │
               ▼
┌────────────────────────────┐
│ HTTP: Crear Comunicación    │
│ Database: Comunicaciones    │
│ Notion                      │
└──────────────┬─────────────┘
               │
               ▼
┌──────────────────────────────┐
│ Code: Confirmar Creación     │
│ Prepara respuesta a Domin   │
└──────────────┬───────────────┘
               │
               ▼
┌──────────────────────────────┐
│ HTTP: WhatsApp Confirmar      │
│ "✅ Registrado. [acción]"    │
└──────────────────────────────┘
```

---

## Ejemplos de Conversión

### Ejemplo 1: Llamada exitosa
**Input:**
```
"Llamé a García López, muy interesado, preguntó por tarifas de gas también"
```

**Estructura extraída:**
```json
{
  "canal": "Llamada",
  "resultado": "Positivo",
  "resumen": "García López muy interesado, pregunta sobre tarifas de gas",
  "proxima_accion": "Enviar comparativa de gas",
  "fecha_proxima_accion": "2026-04-06",
  "notas": "Cliente interesado en gastos combinados",
  "cliente_nombre": "García López"
}
```

**Confirmación a Domin:**
```
✅ Registrado. Enviar comparativa de gas
```

---

### Ejemplo 2: Mensaje sin resultado claro
**Input:**
```
"Dejé mensaje en buzón de voz a la empresa Acme"
```

**Estructura extraída:**
```json
{
  "canal": "Presencial",
  "resultado": "Pendiente",
  "resumen": "Mensaje dejado en buzón de voz a Acme",
  "proxima_accion": "Seguimiento si no hay respuesta",
  "fecha_proxima_accion": "2026-04-12",
  "notas": "Intento sin respuesta aún",
  "cliente_nombre": "Empresa Acme"
}
```

---

### Ejemplo 3: Email enviado
**Input:**
```
"Envié email con comparativa a jgarcia@miempresa.es, está estudiando"
```

**Estructura extraída:**
```json
{
  "canal": "Email",
  "resultado": "Neutral",
  "resumen": "Comparativa enviada a jgarcia@miempresa.es",
  "proxima_accion": "Seguimiento en 3 días",
  "fecha_proxima_accion": "2026-04-08",
  "cliente_nombre": "García",
  "telefono": ""
}
```

---

## Campos de Comunicaciones Actualizados

| Campo Notion | Valor | Tipo |
|---|---|---|
| ID Comunicación | Domin - 2026-04-05T16:25:10Z | Title |
| Tipo | Llamada / WhatsApp / Email / Presencial / SMS | Select |
| Dirección | Saliente | Select |
| Asunto | Resumen de 1 línea | Rich Text |
| Contenido | Mensaje original de Domin | Rich Text |
| Resultado | Positivo / Neutral / Negativo / Pendiente | Select |
| Próxima Acción | Qué hacer después | Rich Text |
| Fecha Próxima Acción | Fecha estimada | Date |
| Realizado Por | Domin | Rich Text |
| Notas Internas | Observaciones | Rich Text |

---

## Valores Válidos de SELECT

### Canal
- Llamada
- WhatsApp
- Email
- Presencial
- SMS

### Dirección
- Entrante
- Saliente

### Resultado
- Positivo
- Neutral
- Negativo
- Pendiente

---

## Manejo de Errores

| Error | Comportamiento |
|-------|---|
| Claude no retorna JSON | Intenta regex, si falla, valida con defaults |
| Comunicación no se crea en Notion | Workflow falla, no se envía confirmación |
| WhatsApp falla al confirmar | Se registra en logs, pero no bloquea |

---

## Variables de Entorno

```
CLAUDE_API_KEY          = "sk-***"
NOTION_TOKEN            = "noti_***"
WHATSAPP_API_TOKEN      = "***"
WHATSAPP_DOMIN_NUMBER   = "+34612345678"
```

---

## Auditoría

Cada mensaje de Domin crea un registro inmutable en Notion. Permite:
- Rastrear historial completo de contactos
- Auditar acciones comerciales
- Planificar follow-ups automaticos
- Analizar patrones de venta

---

## Próximas Mejoras (Future)

1. Integrar con Clientes para auto-actualizar "Ú|timo contacto"
2. Crear tareas automatizadas basadas en "próxima acción"
3. Machine Learning para clasificar resultado automáticamente
4. Análisis de sentimiento en mensajes

