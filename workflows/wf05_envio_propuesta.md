# WF-05: Envío de Propuesta al Cliente

## Objetivo

WF-05 es el último paso del flujo. Una vez que Domin aprueba una comparativa, WF-05 envía automáticamente el PDF al cliente por WhatsApp y actualiza el estado de Notion (Comparativa: Enviada, PS: Procesando).

---

## Trigger

**Execute Workflow Trigger** - Llamado por WF-00 cuando `event_type == "propuesta_aprobada"`

Input: Evento de aprobación de Domin
```json
{
  "event_type": "propuesta_aprobada",
  "data": {
    "comparativa_id": "UUID",
    "ps_id": "UUID",
    "aprobado_por": "domin",
    "fecha_aprobacion": "2026-04-05T17:18:45Z",
    "canal_envio": "whatsapp",
    "numero_envio": "+34612345678"
  }
}
```

---

## Nodos

### 1. Execute Workflow Trigger
**Tipo:** Trigger
**Posición:** (100, 200)

---

### 2. HTTP POST: Query Comparativa
**Tipo:** HTTP Request
**Posición:** (350, 200)

Busca los datos de la comparativa aprobada en Notion.

**Endpoint:** `https://api.notion.com/v1/databases/d4917696-feb6-44fc-be55-9038ca6cbcb4/query`

**Filter:**
```json
{
  "property": "ID Comparativa",
  "title": { "contains": "{{$json.data.comparativa_id}}" }
}
```

**Output:** Datos completos de la Comparativa (incluyendo URL del PDF)

---

### 3. Code: Extraer Datos Comparativa
**Tipo:** Code (JavaScript)
**Posición:** (600, 200)

Extrae campos relevantes de la comparativa.

**Código:**
```javascript
const comp_result = $input.first().json.results[0];
const props = comp_result.properties;

const comparativa_data = {
  id: comp_result.id,
  titulo: props['ID Comparativa']?.title[0]?.plain_text || '',
  pdf_url: props['PDF Comparativa']?.url || '',
  ps_id: props['Punto de Suministro']?.relation[0]?.id,
  cliente_id: props['Cliente']?.relation[0]?.id,
  estado: props['Estado']?.select?.name || ''
};

return [{
  json: {
    comparativa: comparativa_data,
    timestamp: new Date().toISOString()
  }
}];
```

**Output:** Datos de comparativa (incluyendo PDF URL)

---

### 4. HTTP POST: Query PS - Teléfono Cliente
**Tipo:** HTTP Request
**Posición:** (850, 200)

Obtiene el Punto de Suministro para extraer teléfono del cliente.

**Endpoint:** `https://api.notion.com/v1/databases/7648879b-8d9d-4a52-b669-20f485b07ddf/query`

**Filter:** Buscar por ID de PS (de la comparativa)

**Output:** PS con sus propiedades (teléfono, email, nombre titular)

---

### 5. Code: Extraer Datos PS
**Tipo:** Code (JavaScript)
**Posición:** (1100, 200)

Extrae teléfono y datos del cliente del PS.

**Código:**
```javascript
const ps_result = $input.first().json.results[0];
const props = ps_result?.properties || {};

const ps_data = {
  id: ps_result?.id,
  cups: props['CUPS']?.title[0]?.plain_text || '',
  nombre_titular: props['Nombre Titular']?.rich_text[0]?.plain_text || '',
  telefono_guest: props['Teléfono Guest']?.phone_number || '',
  email_guest: props['Email Guest']?.email || ''
};

return [{
  json: {
    ps: ps_data,
    comparativa: $input.all()[1]?.json?.comparativa,
    timestamp: new Date().toISOString()
  }
}];
```

**Output:** Datos del cliente (teléfono principal para envío)

---

### 6. HTTP GET: Descargar PDF desde Drive
**Tipo:** HTTP Request
**Posición:** (1350, 200)

Descarga el PDF desde Google Drive (en caso de necesitarlo para envío por otros canales).

**Endpoint:** `https://www.googleapis.com/drive/v3/files/{{file_id}}?alt=media`

**Note:** Para WhatsApp, pasamos directamente la URL del archivo.

---

### 7. HTTP POST: WhatsApp - Enviar PDF Cliente
**Tipo:** HTTP Request
**Posición:** (1600, 200)

Envía el PDF al cliente por WhatsApp con mensaje personalizado.

**Endpoint:** `https://api.whatsapp.com/send`

**Body:**
```json
{
  "to": "+34612345678",
  "type": "document",
  "document": {
    "link": "https://drive.google.com/uc?id=..."
  },
  "caption": "Estimado Juan García López,\n\nAdjuntamos la comparativa personalizada de tarifas eléctricas para su CUPS ES0001234567890123456789.\n\nEsta propuesta incluye un análisis detallado del ahorro que podría conseguir cambiando de tarifa.\n\nQuedamos a su disposición para cualquier duda.\n\n¡Saludos!\nEquipo Supraluz"
}
```

**Output:** Confirmación de envío (message_id)

---

### 8. HTTP PATCH: Actualizar Comparativa
**Tipo:** HTTP Request
**Posición:** (1850, 200)

Actualiza el estado de la Comparativa en Notion.

**Endpoint:** `https://api.notion.com/v1/pages/{{comparativa_id}}`

**Method:** PATCH

**Body:**
```json
{
  "properties": {
    "Propuesta Enviada al Cliente": { "checkbox": true },
    "Fecha Envío Cliente": { "date": { "start": "{{now}}" } },
    "Estado": { "select": { "name": "Enviada" } },
    "Canal Envío": { "select": { "name": "WhatsApp" } }
  }
}
```

---

### 9. HTTP PATCH: Actualizar PS Estado
**Tipo:** HTTP Request
**Posición:** (2100, 200)

Actualiza el estado del Punto de Suministro a "Procesando".

**Endpoint:** `https://api.notion.com/v1/pages/{{ps_id}}`

**Method:** PATCH

**Body:**
```json
{
  "properties": {
    "Estado": { "select": { "name": "Procesando" } }
  }
}
```

---

### 10. HTTP POST: Confirmar a Domin
**Tipo:** HTTP Request
**Posición:** (2350, 200)

Envía confirmación a Domin de que el cliente recibió la propuesta.

**Endpoint:** `https://api.whatsapp.com/send`

**To:** `{{env.WHATSAPP_DOMIN_NUMBER}}`

**Message:**
```
✅ Propuesta enviada al cliente Juan García López
CUPS: ES0001234567890123456789
Canal: WhatsApp (+34612345678)

Ahora esperamos respuesta del cliente.
```

---

## Flujo de Ejecución

```
┌────────────────────────────────┐
│ Execute Workflow Trigger       │
│ (Domin aprueba comparativa)    │
└──────────────┬─────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ HTTP: Query Comparativa        │
│ Database: Historial Comparativas│
└──────────────┬─────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ Code: Extraer Datos Comparativa│
│ PDF URL + relaciones           │
└──────────────┬─────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ HTTP: Query PS                 │
│ Obtener teléfono del cliente   │
└──────────────┬─────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ Code: Extraer Datos PS         │
│ Teléfono, email, nombre        │
└──────────────┬─────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ HTTP: Descargar PDF Drive       │
│ (opcional, para otros canales) │
└──────────────┬──────────────────┘
               │
               ▼
┌────────────────────────────────────┐
│ HTTP: Enviar PDF WhatsApp          │
│ To: Teléfono del cliente           │
│ Document: URL del PDF               │
│ Caption: Mensaje personalizado      │
└──────────────┬─────────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ HTTP PATCH: Actualizar         │
│ Comparativa -> "Enviada"       │
│ + Fecha envío + Canal envío    │
└──────────────┬─────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ HTTP PATCH: Actualizar PS      │
│ Estado -> "Procesando"         │
└──────────────┬─────────────────┘
               │
               ▼
┌────────────────────────────────┐
│ HTTP: Confirmar a Domin        │
│ "✅ Propuesta enviada"          │
└────────────────────────────────┘
```

---

## Mensaje al Cliente

**Formato:**
```
Estimado {{nombre_titular}},

Adjuntamos la comparativa personalizada de tarifas eléctricas para su CUPS {{cups}}.

Esta propuesta incluye un análisis detallado del ahorro que podría conseguir cambiando de tarifa.

Quedamos a su disposición para cualquier duda.

¡Saludos!
Equipo Supraluz
```

**Personalizaciój:**
- `{{nombre_titular}}`: Del Punto de Suministro
- `{{cups}}`: Código de suministro
- Archivo adjunto: PDF desde Google Drive

---

## Estados Actualizados en Notion

### Comparativa
- **Propuesta Enviada al Cliente:** true ✓
- **Fecha Envío Cliente:** Hoy (date)
- **Estado:** Enviada (select)
- **Canal Envío:** WhatsApp / Email / Presencial (select)

### Punto de Suministro
- **Estado:** Procesando (select)

---

## Flujos Alternativos

Si en el futuro se necesita enviar por otros canales (email, presencial):

**Cambiar nodo 7 a:**
- Para Email: HTTP POST a servicio de email
- Para Presencial: Log + manual (notificar a Domin)

El campo `canal_envio` del evento permite routing según el canal elegido.

---

## Manejo de Errores

| Error | Acción |
|-------|--------|
| Comparativa no encontrada | Workflow falla |
| PS no encontrada | Workflow falla |
| Teléfono vacío | Workflow falla (no se puede enviar) |
| WhatsApp falla | Retry 3x. luego falla |
| Notion PATCH falla | Se registra error, pero no bloquea |

---

## Variables de Entorno Requeridas

```
NOTION_TOKEN          = "noti_***"
WHATSAPP_API_TOKEN    = "***"
WHATSAPP_DOMIN_NUMBER = "+34612345678"
GDRIVE_TOKEN          = "ya29.***" (para acceso Google Drive)
```

---

## Tracking Completo

Una propuesta que pasa por WF-05 queda registrada en Notion con:
1. **Fecha generación** (WF-02)
2. **Aprobada por Domin*.* (checkbox)
3. **Fecha aprobación** (fecha)
4. **Propuesta enviada al cliente** (true)
5. **Fecha envío cliente** (WF-05)
6. **Canal envío+* (WhatsApp / Email / etc.)

Esto permite auditar todo el flujo desde factura hasta envío de propuesta.

---

## Próximas Acciones

Después de WF-05, el cliente puede:
1. **Responder positivamente:** Domin lo registra en WF-04 como "Aceptado"
2. **Responder negativamente:** Domin lo registra como "Rechazado"
3. **No responder:** WF-03 alerta 7 días después (follow-up automático)

