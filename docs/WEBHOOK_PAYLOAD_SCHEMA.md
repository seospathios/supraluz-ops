# Supraluz-Ops: Schema de Webhook y Payloads

Especificación completa del endpoint webhook y estructura de payloads para todos los eventos del sistema.

---

## Endpoint Principal

```
Protocolo: HTTPS (POST)
Host: n8n.supraluz.local (o similar)
Path: /webhook/supraluz-events
Timeout: 30 segundos
Reintentos: 3 (exponential backoff)
```

---

## Header Requeridos

```
Content-Type: application/json
X-Supraluz-Signature: {{HMAC-SHA256}} (opcional, para seguridad)
X-Supraluz-Version: 1.0
```

---

## Estructura Base de Todo Payload

```json
{
  "event_type": "string (tipo de evento)",
  "timestamp": "ISO 8601 datetime",
  "source": "string (origen: whatsapp, email, scheduled, manual, webhook)",
  "correlation_id": "string (UUID para trazabilidad)",
  "user_id": "string (quién dispara: automático, domin, sistema)",
  "data": {
    "...campos específicos del evento..."
  }
}
```

---

## Event Types y Schemas

### 1. EVENT: factura_recibida

**Trigger:** Recepción de PDF de factura por WhatsApp o email

**Campos Requeridos:**
```json
{
  "event_type": "factura_recibida",
  "timestamp": "2026-04-05T14:30:00Z",
  "source": "whatsapp|email|manual",
  "correlation_id": "f47ac10b-58cc-4372-a567-0e02b2c3d479",
  "user_id": "auto|domin|admin",
  "data": {
    "factura_url": "string (URL del PDF descargado o base64)",
    "factura_filename": "string (nombre original del archivo)",
    "factura_bytes": "number (tamaño en bytes)",
    "cliente_id": "string (UUID cliente en Notion, opcional si es nuevo)",
    "cliente_nombre": "string (nombre cliente si es nuevo)",
    "cliente_telefono": "string (número WhatsApp o email)",
    "ps_id": "string (UUID Punto Suministro en Notion, opcional si es nuevo)",
    "ps_cups": "string (CUPS si conocido)",
    "whatsapp_from": "string (îmémero WhatsApp origen, ej: +34612345678)",
    "email_from": "string (email origen, si aplica)",
    "messaje_texto": "string (texto que acompañó al PDF, si hay)",
    "metadata": {
      "date_received": "ISO 8601",
      "mime_type": "application/pdf",
      "platform": "whatsapp|email"
    }
  }
}
```

**Campos Opcionales:**
- `cliente_id`
- `ps_id`
- `ps_cups`
- `email_from`
- `messaje_texto`

**Ejemplo Real:**
```json
{
  "event_type": "factura_recibida",
  "timestamp": "2026-04-05T14:32:15Z",
  "source": "whatsapp",
  "correlation_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "user_id": "auto",
  "data": {
    "factura_url": "https://drive.google.com/uc?id=1abc123...",
    "factura_filename": "factura_marzo_2026.pdf",
    "factura_bytes": 524288,
    "cliente_nombre": "Juan García López",
    "cliente_telefono": "+34612345678",
    "whatsapp_from": "+34612345678",
    "messaje_texto": "Hola, aquí va la factura de este mes",
    "metadata": {
      "date_received": "2026-04-05T14:32:15Z",
      "mime_type": "application/pdf",
      "platform": "whatsapp"
    }
  }
}
```

**Workflow Destino:** WF-01 (análisis_factura)

---

### 2. EVENT: comparativa_lista

**Trigger:** Comparativa generada y lista para aprobación comercial

**Campos Requeridos:**
```json
{
  "event_type": "comparativa_lista",
  "timestamp": "2026-04-05T15:45:00Z",
  "source": "sistema",
  "correlation_id": "b2c3d4e5-f6a7-8901-bcde-f1234567890a",
  "user_id": "auto",
  "data": {
    "comparativa_id": "string (UUID de Comparativa en Notion)",
    "ps_id": "string (UUID de Punto de Suministro)",
    "cliente_id": "string (UUID de Cliente)",
    "cliente_nombre": "string",
    "nombre_titular": "string",
    "cups": "string",
    "coste_actual_anual": "number (€)",
    "coste_nuevo_anual": "number (€)",
    "ahorro_estimado": "number (€)",
    "porcentaje_ahorro": "number (%)",
    "tarifa_recomendada": {
      "id": "string (UUID tarifa en Notion)",
      "nombre": "string",
      "comercializadora": "string",
      "precio_energia": "number (€/kWh)",
      "precio_potencia": "number (€/kW/día)"
    },
    "tarifas_alternativas": [
      {
        "id": "string",
        "nombre": "string",
        "ahorro_anual": "number"
      }
    ],
    "pdf_url": "string (URL en Google Drive)",
    "pdf_nombre": "string (Supraluz_C00001_Juan_Garcia_Lopez.pdf)",
    "resumen_analisis": "string (párrafos de análisis)",
    "validacion": {
      "ahorro_minimo_recomendado": "number (100)",
      "cumple_criterios": "boolean"
    },
    "metadata": {
      "origen_factura_fecha": "ISO 8601 (cuándo llegó factura)"
    }
  }
}
```

**Campos Opcionales:**
- `tarifas_alternativas` (puede estar&�acío)

**Ejemplo Real:**
```json
{
  "event_type": "comparativa_lista",
  "timestamp": "2026-04-05T15:47:32Z",
  "source": "sistema",
  "correlation_id": "b2c3d4e5-f6a7-8901-bcde-f1234567890a",
  "user_id": "auto",
  "data": {
    "comparativa_id": "c1d2e3f4-g5h6-i789-jklm-n0o1p2q3r4s5",
    "ps_id": "d2e3f4g5-h6i7-j8k9-lmno-p1q2r3s4t5u6",
    "cliente_id": "e3f4g5h6-i7j8-k9l0-mnop-q2r3s4t5u6v7",
    "cliente_nombre": "García López SL",
    "nombre_titular": "Juan García López",
    "cups": "ES0001234567890123456789",
    "coste_actual_anual": 3600,
    "coste_nuevo_anual": 3150,
    "ahorro_estimado": 450,
    "porcentaje_ahorro": 12.5,
    "tarifa_recomendada": {
      "id": "f4g5h6i7-j8k9-l0m1-nopq-r3s4t5u6v7w8",
      "nombre": "ENDESA TARIFA FIJA 2026",
      "comercializadora": "Endesa",
      "precio_energia": 0.175,
      "precio_potencia": 0.42
    },
    "tarifas_alternativas": [
      {
        "id": "g5h6i7j8-k9l0-m1n2-opqr-s4t5u6v7w8x9",
        "nombre": "IBERDROLA FLEX PYME",
        "ahorro_anual": 380
      }
    ],
    "pdf_url": "https://drive.google.com/file/d/1xyz789...",
    "pdf_nombre": "Supraluz_C00001_Juan_Garcia_Lopez.pdf",
    "resumen_analisis": "Análisis confidencial para aprobación interna...",
    "validacion": {
      "ahorro_minimo_recomendado": 100,
      "cumple_criterios": true
    },
    "metadata": {
      "origen_factura_fecha": "2026-04-05T14:32:15Z"
    }
  }
}
```

**Workflow Destino:** WF-02 (notificación a Domin)
**También Dispara:** Notificación WhatsApp a `WHATSAPP_DOMIN_NUMBER`

---

### 3. EVENT: renovacion_proxima

**Trigger:** Generado por WF-03 (schedule diario)

**Campos Requeridos:**
```json
{
  "event_type": "renovacion_proxima",
  "timestamp": "2026-04-05T08:00:00Z",
  "source": "scheduled",
  "correlation_id": "c3d4e5f6-g7h8-i9j0-klmn-o5p6q7r8s9t0",
  "user_id": "auto",
  "data": {
    "renovaciones": [
      {
        "ps_id": "string (UUID)",
        "cliente_id": "string (UUID)",
        "cliente_nombre": "string",
        "cups": "string",
        "fecha_renovacion": "ISO 8601 (fecha vencimiento)",
        "dias_para_vencer": "number (0-30)",
        "tarifa_actual": "string (nombre)",
        "comercializadora_actual": "string"
      }
    ],
    "total_renovaciones": "number",
    "rango_fechas": {
      "desde": "ISO 8601",
      "hasta": "ISO 8601"
    }
  }
}
```

**Ejemplo Real:**
```json
{
  "event_type": "renovacion_proxima",
  "timestamp": "2026-04-05T08:00:00Z",
  "source": "scheduled",
  "correlation_id": "c3d4e5f6-g7h8-i9j0-klmn-o5p6q7r8s9t0",
  "user_id": "auto",
  "data": {
    "renovaciones": [
      {
        "ps_id": "d2e3f4g5-h6i7-j8k9-lmno-p1q2r3s4t5u6",
        "cliente_id": "e3f4g5h6-i7j8-k9l0-mnop-q2r3s4t5u6v7",
        "cliente_nombre": "García López SL",
        "cups": "ES0001234567890123456789",
        "fecha_renovacion": "2026-04-25",
        "dias_para_vencer": 20,
        "tarifa_actual": "2.0A Endesa",
        "comercializadora_actual": "Endesa"
      },
      {
        "ps_id": "f5g6h7i8-j9k0-l1m2-nopq-r5s6t7u8v9w0",
        "cliente_id": "g6h7i8j9-kl1-m2n3-opqr-s6t7u8v9w0x1",
        "cliente_nombre": "Empresa Acme SL",
        "cups": "ES0002345678901234567890",
        "fecha_renovacion": "2026-04-30",
        "dias_para_vencer": 25,
        "tarifa_actual": "3.0A Iberdrola",
        "comercializadora_actual": "Iberdrola"
      }
    ],
    "total_renovaciones": 2,
    "rango_fechas": {
      "desde": "2026-04-05",
      "hasta": "2026-05-05"
    }
  }
}
```

**Workflow Destino:** WF-03 (envi�o WhatsApp a Domin)

---

### 4. EVENT: whatsapp_domin_input

**Trigger:** Mensaje de WhatsApp de Domin (comercial)

**Campos Requeridos:**
```json
{
  "event_type": "whatsapp_domin_input",
  "timestamp": "2026-04-05T16:20:00Z",
  "source": "whatsapp",
  "correlation_id": "d4e5f6g7-h8i9-j0k1-lmno-p7q8r9s0t1u2",
  "user_id": "domin",
  "data": {
    "message_id": "string (ID único en WhatsApp)",
    "message_text": "string (texto libre del mensaje)",
    "whatsapp_from": "string (+34xxxxxxxxx)",
    "timestamp_mensaje": "ISO 8601",
    "tipo_mensaje": "text|media",
    "media_url": "string (si es imagen/audio, opcional)",
    "referenced_message": {
      "message_id": "string (si es reply, opcional)",
      "text_preview": "string"
    }
  }
}
```

**Campos Opcionales:**
- `media_url`
- `referenced_message`

**Ejemplo Real:**
```json
{
  "event_type": "whatsapp_domin_input",
  "timestamp": "2026-04-05T16:25:10Z",
  "source": "whatsapp",
  "correlation_id": "d4e5f6g7-h8i9-j0k1-lmno-p7q8r9s0t1u2",
  "user_id": "domin",
  "data": {
    "message_id": "wamid.HBEUHUFDFDUFDJKFHDJkdf==",
    "message_text": "Llamé a García López, muy interesado, dijo que llamo el lunes",
    "whatsapp_from": "+34612345678",
    "timestamp_mensaje": "2026-04-05T16:25:10Z",
    "tipo_mensaje": "text"
  }
}
```

**Workflow Destino:** WF-04 (procesamiento con Claude)

---

### 5. EVENT: propuesta_aprobada

**Trigger:** Domin aprueba comparativa en WhatsApp y da orden de envío

**Campos Requeridos:**
```json
{
  "event_type": "propuesta_aprobada",
  "timestamp": "2026-04-05T17:15:00Z",
  "source": "whatsapp|manual",
  "correlation_id": "e5f6g7h8-i9j0-k1l2-mnop-q8r9s0t1u2v3",
  "user_id": "domin",
  "data": {
    "comparativa_id": "string (UUID en Notion)",
    "ps_id": "string (UUID)",
    "cliente_id": "string (UUID)",
    "aprobado_por": "string (nombre usuario, ej: domin)",
    "fecha_aprobacion": "ISO 8601",
    "canal_envio": "whatsapp|email|presencial",
    "numero_envio": "string (teléfono o email destino)",
    "observaciones_domin": "string (si hay comentarios)",
    "condiciones_especiales": "string (descuentos, bonos, etc, opcional)"
  }
}
```

**Campos Opcionales:**
- `observaciones_domin`
- `condiciones_especiales`

**Ejemplo Real:**
```json
{
  "event_type": "propuesta_aprobada",
  "timestamp": "2026-04-05T17:18:45Z",
  "source": "whatsapp",
  "correlation_id": "e5f6g7h8-i9j0-kk1l2-mnop-q8r9s0t1u2v3",
  "user_id": "domin",
  "data": {
    "comparativa_id": "c1d2e3f4-g5h6-i789-jklm-n0o1p2q3r4s5",
    "ps_id": "d2e3f4g5-h6i7-j8k9-lmno-p1q2r3s4t5u6",
    "cliente_id": "e3f4g5h6-i7j8-k9l0-mnop-q2r3s4t5u6v7",
    "aprobado_por": "domin",
    "fecha_aprobacion": "2026-04-05T17:18:45Z",
    "canal_envio": "whatsapp",
    "numero_envio": "+34612345678",
    "observaciones_domin": "Cliente muy interesado, mejor con Endesa"
  }
}
```

**Workflow Destino:** WF-05 (envi�o PDF al cliente)

---

## Tabla de Campos Requeridos por Event Type

| Campo | factura_recibida | comparativa_lista | renovacion_proxima | whatsapp_domin_input | propuesta_aprobada |
|-------|:-:|:-:|:-:|:-:|:-:|
| event_type | ✓ | ✓ | ✓ | ✓ | ✓ |
| timestamp | ✓ | ✓ | ✓ | ✓ | ✓ |
| source | ✓ | ✓ | ✓ | ✓ | ✓ |
| correlation_id | ✓ | ✓ | ✓ | ✓ | ✓ |
| user_id | ✓ | ✓ | ✓ | ✓ | ✓ |
| factura_url | ✓ |  |  |  |  |
| cliente_nombre | ✓ |  |  |  |  |
| ps_cups | ✓ |  |  |  |  |
| whatsapp_from | ✓ |  |  | ✓ |  |
| comparativa_id |  | ✓ |  |  | ✓ |
| ahorro_estimado |  | ✓ |  |  |  |
| renovaciones |  |  | ✓ |  |  |
| message_text |  |  |  | ✓ |  |
| aprobado_por |  |  |  |  | ✓ |
| canal_envio |  |  |  |  | ✓ |
| numero_envio |  |  |  |  | ✓ |

---

## Respuestas HTTP Esperadas

### Success (2xx)
```json
HTTP 200 OK
{
  "status": "received",
  "event_id": "{{correlation_id}}",
  "workflow_triggered": "WF-XX",
  "message": "Payload procesado exitosamente"
}
```

### Error (4xx)
```json
HTTP 400 Bad Request
{
  "status": "error",
  "error_code": "MISSING_REQUIRED_FIELDS",
  "message": "Faltan campos requeridos9�[lista]",
  "details": {
    "missing_fields": ["comparativa_id", "canal_envio"]
  }
}
```

### Error (5xx)
```json
HTTP 500 Internal Server Error
{
  "status": "error",
  "error_code": "WORKFLOW_EXECUTION_FAILED",
  "message": "Error ejecutando workflow",
  "correlation_id": "{{correlation_id}}"
}
```

---

## Validaciones

### Todas las Requests

1. `event_type` debe ser uno de los 5 tipos válidos
2. `timestamp` debe ser ISO 8601 válido (no futuro)
3. `correlation_id` debe ser UUID válido
4. `source` debe ser uno de: `whatsapp`, `email`, `scheduled`, `manual`, `webhook`
5. `user_id` debe ser no-vacío

### Específicas por Event

**factura_recibida:**
- `factura_url` debe ser URL o base64 válido
- `whatsapp_from` o `email_from` obligatorio (al menos uno)
- Si `cliente_id` no existe, `cliente_nombre` es requerido

**comparativa_lista:**
- `ahorro_estimado` debe ser > 0
- `pdf_url` debe ser URL válida en Google Drive
- `tarifa_recomendada.id` debe existir en Notion

**whatsapp_domin_input:**
- `message_text` no puede estar vacío
- `whatsapp_from` debe ser número formato internacional

**propuesta_aprobada:**
- `canal_envio` debe ser uno de: `whatsapp`, `email`, `presencial`
- Si `canal_envio` es `whatsapp` o `email`, `numero_envio` obligatorio

---

## Retry Policy

- **Max Retries:** 3
- **Initial Backoff:** 2 segundos
- **Exponential Factor:** 2x
- **Max Backoff:** 32 segundos
- **Retry Conditions:** 408, 429, 500, 502, 503, 504

---

## Trazabilidad

Todos los eventos deben mantener el `correlation_id` a través de toda la cadena de workflows:
```
webhook → WF-00 > WF-0X > logs finales
```

El `correlation_id` permite auditar el flujo completo de un cliente desde factura hasta resultado final.
