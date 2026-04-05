# WF-00: Dispatcher Central de Supraluz-Ops

## Objetivo

WF-00 es el **orquestador central** del sistema. Recibe todos los eventos webhook y los enruta a los workflows específicos correspondientes (WF-01 a WF-05) basándose en el `event_type`.

Actúa como punto único de entrada que garantiza:
- Validación de payloads
- Enrutamiento correcto
- Trazabilidad de todas las operaciones
- Configuración centralizada de variables de entorno

---

## Trigger

**Webhook Supraluz Events**
- **Endpoint:** `POST /webhook/supraluz-events`
- **Método:** HTTP POST
- **Response Mode:** onReceived (responde inmediatamente)
- **Response Data:** JSON de confirmación con `correlation_id` y `event_id`

**Aceptar requests de:**
- WhatsApp (webhook de Twilio o similar)
- Email (webhook de servicio de email)
- Scheduler interno (triggers de n8n)
- Manual (test desde UI de n8n)

---

## Nodos

### 1. Webhook Supraluz Events
**Tipo:** Webhook
**Posición:** (100, 200)

Escucha en `/webhook/supraluz-events` y captura todo el payload JSON.

**Payload esperado:** Ver `supraluz_WEBHOOK_PAYLOAD_SCHEMA.md`

**Output:** Todo el payload JSON original

---

### 2. Set Config Variables
**Tipo:** Set
**Posición:** (350, 200)

Establece variables de entorno globales que serán usadas por los sub-workflows.

**Variables a configurar (desde environment o credenciales):**

```
NOTION_TOKEN        = "noti_***" (Secret)
CLAUDE_API_KEY      = "sk-***" (Secret)
GDRIVE_FOLDER_ID    = "1a2b3c4d..." (ID de carpeta Google Drive)
WHATSAPP_DOMIN_NUMBER = "+34612345678" (Número de Domin)
WHATSAPP_API_TOKEN  = "***" (Token WhatsApp Business API)
PYTHON_PDF_ENDPOINT = "https://internal.supraluz.local/generate-pdf"
WF01_ID             = "123456" (ID de WF-01 en n8n)
WF02_ID             = "123457" (ID de WF-02 en n8n)
WF03_ID             = "123458" (ID de WF-03 en n8n)
WF04_ID             = "123459" (ID de WF-04 en n8n)
WF05_ID             = "123460" (ID de WF-05 en n8n)
```

*Nota:** Se recomienda usar credenciales de n8n (tipo "Secret") para valores sensibles.

**Output:** Payload original + variables inyectadas

---

### 3. Route por Event Type
**Tipo:** Switch
**Posición:** (600, 200)

Nodo condicional que evalúa el campo `event_type` del payload y decide hacia qué salida dirigir el flujo.

**Reglas de Routing:**

| Condition | Output | Workflow |
|-----------|--------|----------|
| `event_type == "factura_recibida"` | Salida 1 | WF-01 |
| `event_type == "comparativa_lista"` | Salida 2 | WF-02 |
| `event_type == "renovacion_proxima"` | Salida 3 | WF-03 |
| `event_type == "whatsapp_domin_input"` | Salida 4 | WF-04 |
| `event_type == "propuesta_aprobada"` | Salida 5 | WF-05 |
| Ninguna de las anteriores (fallback) | Salida 6 | Fallback |

**Output:** Mismo payload, dirigido a una de las 6 salidas según su tipo.

---

### 4. Execute WF-01: Análisis Factura
**Tipo:** Execute Workflow
**Posición:** (850, 50)
**Trigger en salida:** 1 (factura_recibida)

Ejecuta WE-01 pasando todo el payload del webhook.

**Parámetros:**
- `workflowId`: `{{env.WF01_ID}}`
- `args`: `{{$json}}` (todo el payload)

**Espera:** Ejecución completa de WF-01 y su resultado

**Output:** Resultado de WF-01 (contexto enriquecido del análisis)

---

### 5. Execute WF-02: Comparativa PDF
**Tipo:** Execute Workflow
**Posición:** (850, 150)
**Trigger en salida:** 2 (comparativa_lista)

Ejecuta WF-02 para generar y procesar la comparativa PDF.

**Parámetros:**
- `workflowId`: `{{env.WF02_ID}}`
- `args`: `{{$json}}`

**Output:** Resultado de WF-02 (URL PDF, ID Notion, notificación a Domin)

---

### 6. Execute WF-03: Alarmas Renovación
**Tipo:** Execute Workflow
**Posición:** (850, 250)
**Trigger en salida:** 3 (renovacion_proxima)

Ejecuta WE-03 para procesar alertas de renovación próxima.

**Parámetros:**
- `workflowId`: `{{env.WF03_ID}}`
- `args`: `{{$json}}`

**Output:** Resultado de WF-03 (lista formateada de renovaciones, mensaje a Domin)

---

### 7. Execute WF-04: Input WhatsApp
**Tipo:** Execute Workflow
**Posición:** (850, 350)
**Trigger en salida:** 4 (whatsapp_domin_input)

Ejecuta WF-04 para procesar mensaje de WhatsApp de Domin.

**Parámetros:**
- `workflowId`: `{{env.WF04_ID}}`
- `args`: `{{$json}}`

**Output:** Resultado de WF-04 (Comunicación creada, Clientes actualizado)

---

### 8. Execute WF-05: Envío Propuesta
**Tipo:** Execute Workflow
**Posición:** (850, 450)
**Trigger en salida:** 5 (propuesta_aprobada)

Ejecuta WF-05 para enviar PDF comparativa al cliente.

**Parámetros:**
- `workflowId`: `{{env.WF05_ID}}`
- `args`: `{{$json}}`

**Output:** Resultado de WF-05 (PDF enviado al cliente, Notion actualizado)

---

### 9. Log Trazabilidad Final
**Tipo:** Code (JavaScript)
**Posición:** (1150, 200)

Nodo que consolida la trazabilidad final de cualquier path ejecutado.

**Código:**
```javascript
const event_type = $input.first().json.event_type;
const timestamp = new Date().toISOString();
const correlation_id = $input.first().json.correlation_id;
const source = $input.first().json.source;
const user_id = $input.first().json.user_id;

return [{
  json: {\n    dispatcher_log: {
      timestamp: timestamp,
      correlation_id: correlation_id,
      event_type: event_type,
      source: source,
      user_id: user_id,
      status: 'processed',
      workflow_routed: true,
      message: `Evento ${event_type} procesado correctamente`,
      duration_ms: $input.first().json.process_duration || 'N/A'
    }
  }
}];
```

**Output:** Log estructurado con trazabilidad completa

---

### 10. Fallback - Unknown Event Type
**Tipo:** Code (JavaScript)
**Posición:** (850, 550)
**Trigger en salida:** 6 (fallback)

Maneja eventos con `event_type` no reconocido.

**Código:**
```javascript
const event_type = $input.first().json.event_type;
const correlation_id = $input.first().json.correlation_id;

return [{
  json: {
    error_log: {
      timestamp: new Date().toISOString(),
      correlation_id: correlation_id,
      event_type: event_type,
      status: 'error',
      message: 'Event type no reconocido. Evento descartado.',
      accepted_types: ['factura_recibida', 'comparativa_lista', 'renovacion_proxima', 'whatsapp_domin_input', 'propuesta_aprobada']
    }
  }
}];
```

**Output:** Log de error con tipos aceptados
