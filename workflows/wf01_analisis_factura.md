# WF-01: Análisis de Factura Eléctrica

## Objetivo

WF-01 recibe una factura PDF y extrae automáticamente todos los datos técnicos y contractuales necesarios usando Claude API. Los datos extraídos se utilizan para:
1. Crear o actualizar el Punto de Suministro en Notion
2. Preparar el contexto para generación de comparativas (WF-02)
3. Mantener histórico actualizado de facturas procesadas

---

## Triggers

### 1. Execute Workflow Trigger (Principal)
- Llamado por B*WF-00 Dispatcher** cuando `event_type == "factura_recibida"`
- Recibe el payload completo del webhook
- Es la forma de producción

### 2. Manual Trigger (Test)
- Permite ejecutar manualmente WF-01 desde UI de n8n
- Útil para debugging y pruebas
- Ignora el payload de WF-00

---

## Nodos del Workflow

### 1. Execute Workflow Trigger
**Tipo:** Trigger
**Posición:** (100, 200)

Escucha cuando WF-00 lo ejecuta directamente.

**Input esperado:** Payload webhook con estructura:
```json
{
  "event_type": "factura_recibida",
  "data": {
    "factura_url": "https://...",
    "cliente_nombre": "...",
    "whatsapp_from": "+34..."
  },
  "correlation_id": "UUID",
  "timestamp": "ISO 8601"
}
```

---

### 2. Manual Trigger (Test)
**Tipo:** Manual Trigger
**Posición:** (100, 350)

Permite testear sin pasar por WF-00.

---

### 3. HTTP GET: Descargar PDF
**Tipo:** HTTP Request
**Posición:** (350, 200)
**Trigger:** Ambos triggers (1 y 2)

Descarga el PDF desde la URL proporcionada.

**Parámetros:**
- **URL:** `{{$json.data.factura_url}}`
- **Method:** GET
- **Response Format:** arraybuffer (para manejo binario)

**Output:** Buffer binario del PDF

**Error Handling:**
- Si URL no válida: HTTP 400
- Si PDF corrupto: falla silenciosa, se continúa

---

### 4. HTTP POST: Claude API - Extracción
**Tipo:** HTTP Request
**Posición:** (600, 200)

Envía el PDF a Claude Sonnet 4.6 para extracción estructurada de datos.

**Endpoint:** `https://api.anthropic.com/v1/messages`
**Method:** POST

**Headers:**
```
x-api-key: {{env.CLAUDE_API_KEY}}
anthropic-version: 2023-06-01
Content-Type: application/json
```

**Body Payload:**
```json
{
  "model": "claude-sonnet-4-6-20250514",
  "max_tokens": 2000,
  "system": "Eres un experto en facturas eléctricas. Tu tarea es extraer información estructurada de una factura eléctrica. Responde SIEMPRE en JSON válido.",
  "messages": [
    {
      "role": "user",
      "content": "Extrae los siguientes datos de la factura:\n1. CUPS (código de punto de suministro)\n2. Nombre del titular\n3. DNI/CIF del titular\n4. Tipo de titular (Propietario/Arrendatario/Autorizado/Otro)\n5. Comercializadora (ej: Endesa, Iberdrola)\n6. Tarifa actual (ej: 2.0A, 3.0A)\n7. Potencia contratada (kW)\n8. Consumo (kWh)\n9. Precio de la energía (€/kWh)\n10. Precio de la potencia (€/kW/día)\n11. Importe total factura (€)\n12. Fecha de la factura (YYYY-MM-DD)\n13. Fecha de renovación del contrato (YYYY-MM-DD si aparece)\n14. Meses de permanencia restantes\n\nRespuesta en formato JSON con estructura: {\"cups\":\"\",\"nombre_titular\":\"\",\"dni_cif_titular\":\"\",\"tipo_titular\":\"\",\"comercializadora\":\"\",\"tarifa_actual\":\"\",\"potencia_kw\":0,\"consumo_kwh\":0,\"precio_energia\":0,\"precio_potencia\":0,\"importe_factura\":0,\"fecha_factura\":\"\",\"fecha_renovacion\":\"\",\"permanencia_meses\":0}"
    }
  ]
}
```

**Prompt Detallado:**
El prompt instruye a Claude a:
1. Extraer 14 campos específicos
2. Retornar JSON válido siempre
3. Usar tipos de datos correctos (string, number)
4. Usar formato ISO para fechas

**Output:** JSON con respuesta de Claude
```json
{
  "content": [
    {
      "type": "text",
      "text": "{\"cups\":\"ES0001234567890123456789\",\"nombre_titular\":\"Juan García López\",\"dni_cif_titular\":\"12345678A\",\"tipo_titular\":\"Propietario\",\"comercializadora\":\"Endesa\",\"tarifa_actual\":\"2.0A\",\"potencia_kw\":5.5,\"consumo_kwh\":12000,\"precio_energia\":0.18,\"precio_potencia\":0.42,\"importe_factura\":2150,\"fecha_factura\":\"2026-04-05\",\"fecha_renovacion\":\"2026-07-05\",\"permanencia_meses\":3}"
    }
  ]
}
```

---

### 5. Code: Estructura Extracción
**Tipo:** Code (JavaScript)
**Posición:** (850, 200)

Valida y estructura la respuesta de Claude en JSON limpio y tipado.

**Código:**
```javascript
const response = $input.first().json;
const content = response.content[0].text;

let extracted_data = {};
try {
  extracted_data = JSON.parse(content);
} catch (e) {
  // Si no es JSON válido, intentar extraer JSON del texto
  const jsonMatch = content.match(/\{[\s\S]*\}/);
  if (jsonMatch) {
    extracted_data = JSON.parse(jsonMatch[0]);
  } else {
    throw new Error('No se pudo extraer datos válidos del PDF');
  }
}

// Validar campos requeridos
const required_fields = ['cups', 'nombre_titular', 'comercializadora', 'tarifa_actual', 'potencia_kw', 'consumo_kwh', 'precio_energia', 'precio_potencia', 'importe_factura', 'fecha_factura'];

const missing = required_fields.filter(f => !extracted_data[f]);
if (missing.length > 0) {
  throw new Error(`Campos faltantes: ${missing.join(', ')}`);
}

// Normalizar tipos de datos
return [{
  json: {
    data: {
      cups: extracted_data.cups.toUpperCase(),
      nombre_titular: extracted_data.nombre_titular.trim(),
      dni_cif_titular: extracted_data.dni_cif_titular?.trim() || '',
      tipo_titular: extracted_data.tipo_titular || 'Propietario',
      comercializadora: extracted_data.comercializadora.trim(),
      tarifa_actual: extracted_data.tarifa_actual.trim(),
      potencia_kw: parseFloat(extracted_data.potencia_kw),
      consumo_kwh: parseFloat(extracted_data.consumo_kwh),
      precio_energia: parseFloat(extracted_data.precio_energia),
      precio_potencia: parseFloat(extracted_data.precio_potencia),
      importe_factura: parseFloat(extracted_data.importe_factura),
      fecha_factura: extracted_data.fecha_factura,
      fecha_renovacion: extracted_data.fecha_renovacion || null,
      permanencia_meses: parseInt(extracted_data.permanencia_meses) || 0
    },
    extraction_timestamp: new Date().toISOString()
  }
}];
```

**Validaciones:**
1. Parsea JSON (con fallback a regex si es necesario)
2. Verifica que existan todos los campos requeridos
3. Normaliza strings (trim, uppercase CUPS)
4. Convierte a tipos correctos (parseFloat, parseInt)

**Output:** Datos limpios y validados

---

### 6. HTTP POST: Notion Query PS por CUPS
**Tipo:** HTTP Request
**Posición:** (1100, 200)

Busca si el Punto de Suministro ya existe en Notion.

**Endpoint:** `https://api.notion.com/v1/databases/7648879b-8d9d-4a52-b669-20f485b07ddf/query`

**Method:** POST

**Headers:**
```
Authorization: Bearer {{env.NOTION_TOKEN}}
Notion-Version: 2022-06-28
```

**Body:**
```json
{
  "filter": {
    "property": "CUPS",
    "title": {
      "equals": "{{$json.data.cups}}"
    }
  }
}
```

**Output:** Resultado de búsqueda
- Si no existe: `results: []`
- Si existe: `results: [{ id: "...", properties: {...} }]`

---

### 7. Code: Evaluar PS Existente
**Tipo:** Code (JavaScript)
**Posición:** (1350, 200)

Decide si crear nuevo PS o actualizar existente.

**Lógica:**
```javascript
if (results.length === 0) {
  return { action: 'create_new_ps', ps_exists: false }
} else if (results.length === 1) {
  return { action: 'update_existing_ps', ps_id: results[0].id, ps_exists: true }
} else {
  return { error: 'Multiples PS con mismo CUPS' }
}
```

**Output:**
- Si no existe: señal hacia HTTP POST Crear
- Si existe: señal hacia HTTP PATCH Actualizar

---

### 8. HTTP POST: Crear Nuevo PS
**Tipo:** HTTP Request
**Posición:** (1500, 50)
**Trigger:** Desde nodo 7 si no existe PS

Crea nuevo Punto de Suministro en Notion.

**Endpoint:** `https://api.notion.com/v1/pages`

**Method:** POST

**Headers:**
```
Authorization: Bearer {{env.NOTION_TOKEN}}
Notion-Version: 2022-06-28
```

**Body - Mapeo de Campos:**
```json
{
  "parent": {
    "database_id": "7648879b-8d9d-4a52-b669-20f485b07ddf"
  },
  "properties": {
    "CUPS": {
      "title": [{ "text": { "content": "{{$json.ps_data.cups}}" } }]
    },
    "Nombre Titular": {
      "rich_text": [{ "text": { "content": "{{$json.ps_data.nombre_titular}}" } }]
    },
    "DNI/CIF Titular": {
      "rich_text": [{ "text": { "content": "{{$json.ps_data.dni_cif_titular}}" } }]
    },
    "Tipo Titular": {
      "select": { "name": "{{$json.ps_data.tipo_titular}}" }
    },
    "Comercializadora Actual": {
      "rich_text": [{ "text": { "content": "{{$json.ps_data.comercializadora}}" } }]
    },
    "Tarifa Actual": {
      "rich_text": [{ "text": { "content": "{{$json.ps_data.tarifa_actual}}" } }]
    },
    "Potencia Contratada": {
      "number": "{{$json.ps_data.potencia_kw}}"
    },
    "Consumo Anual": {
      "number": "{{$json.ps_data.consumo_kwh}}"
    },
    "Precio Energia Actual": {
      "number": "{{$json.ps_data.precio_energia}}"
    },
    "Precio Potencia Actual": {
      "number": "{{$json.ps_data.precio_potencia}}"
    },
    "Importe Factura Ultima": {
      "number": "{{$json.ps_data.importe_factura}}"
    },
    "Fecha Factura Ultima": {
      "date": { "start": "{{$json.ps_data.fecha_factura}}" }
    },
    "Fecha Renovacion": {
      "date": { "start": "{{$json.ps_data.fecha_renovacion}}" }
    },
    "Permanencia Meses": {
      "number": "{{$json.ps_data.permanencia_meses}}"
    },
    "Estado": {
      "select": { "name": "En Revision" }
    }
  }
}
```

**Output:** ID de PS creado + confirmacion

---

### 9. HTTP PATCH: Actualizar PS Existente
**Tipo:** HTTP Request
**Posición:** (1500, 350)
**Trigger:** Desde nodo 7 si existe PS

Actualiza un PS existente con nuevos datos de factura.

**Endpoint:** `https://api.notion.com/v1/pages/{{$json.ps_id}}`

**Method:** PATCH

**Headers:** Igual que POST

**Body:** Mismo mapeo que nodo 8, pero sin CUPS (que es Title y no se modifica)

**Output:** PS actualizado confirmado

---

### 10. Code: Contexto Enriquecido para WF-02
**Tipo:** Code (JavaScript)
**Posición:** (1750, 200)

Prepara el contexto que sera pasado a WF-02 para generacion de comparativa.

**Codigo:**
```javascript
const ps_info = $input.first().json;
const from_input = $input.all()[0].json;

const context = {
  correlation_id: from_input.correlation_id || 'no-correlation-id',
  timestamp: new Date().toISOString(),
  event_type: 'factura_analizada',
  ps_id: ps_info.ps_id || ps_info.id,
  ps_exists: ps_info.ps_exists || false,
  ps_data: ps_info.ps_data,
  coste_anual_actual: (ps_info.ps_data?.consumo_kwh * ps_info.ps_data?.precio_energia) + (ps_info.ps_data?.potencia_kw * 365 * ps_info.ps_data?.precio_potencia),
  status: 'success',
  message: 'Factura analizada y Punto de Suministro actualizado',
  ready_for_comparison: true
};

return [{
  json: context
}];
```

**Formula de Coste Anual:**
```
coste_anual_actual = (consumo_kwh * precio_energia) + (potencia_kw * 365 * precio_potencia)
```

**Output:** Contexto enriquecido listo para WF-02

---

## Flujo de Ejecucion

```
┌─────────────────────┐
│ Execute Workflow     │
│ + Manual Trigger     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────┐
│ HTTP GET: Descargar PDF     │
│ URL: $json.data.factura_url │
└──────────┬──────────────────┘
           │
           ▼
       [Claude API + Code Nodes]
            │
           ▼
     [Notion QS + Crear/Actualizar PS]
           │
           ▼
      [Contexto Enriquecido -> WF-02]
```

---

## Campos que Extrae Claude

| Campo | Tipo | Ejemplo | Requerido |
|-------|------|---------|-----------|
| cups | string | ES0001234567890123456789 | Si |
| nombre_titular | string | Juan Garcia Lopez | Si |
| dni_cif_titular | string | 12345678A | Si |
| tipo_titular | select | Propietario | Si |
| comercializadora | string | Endesa | Si |
| tarifa_actual | string | 2.0A | Si |
| potencia_kw | number | 5.5 | Si |
| consumo_kwh | number | 12000 | Si |
| precio_energia | number | 0.18 | Si |
| precio_potencia | number | 0.42 | Si |
| importe_factura | number | 2150 | Si |
| fecha_factura | date (YYYY-MM-DD) | 2026-04-05 | Si |
| fecha_renovacion | date (YYYY-MM-DD) | 2026-07-05 | No |
| permanencia_meses | number | 3 | No |

---

## Campos que Actualiza en Notion (Puntos de Suministro)

| Campo Extraido | Propiedad Notion | Tipo |
|---|---|---|
| cups | CUPS | Title |
| nombre_titular | Nombre Titular | Rich Text |
| dni_cif_titular | DNH/CIF Titular | Rich Text |
| tipo_titular | Tipo Titular | Select |
| comercializadora | Comercializadora Actual | Rich Text |
| tarifa_actual | Tarifa Actual | Rich Text |
| potencia_kw | Potencia Contratada | Number |
| consumo_kwh | Consumo Anual | Number |
| precio_energia | Precio Energia Actual | Number |
| precio_potencia | Precio Potencia Actual | Number |
| importe_factura | Importe Factura Ultima | Number |
| fecha_factura | Fecha Factura Ultima | Date |
| fecha_renovacion | Fecha Renovacion | Date |
| permanencia_meses | Permanencia Meses | Number |
| (automatico) | Estado | Select: "En Revision" |

---

## Output Hacia WF-02

WF-01 retorna este contexto enriquecido:

```json
{
  "correlation_id": "UUID",
  "timestamp": "2026-04-05T14:35:20Z",
  "event_type": "factura_analizada",
  "ps_id": "d2e3f4g5-h6i7-j8k9-lmno-p1q2r3s4t5u6",
  "ps_exists": true,
  "ps_data": {
    "cups": "ES0001234567890123456789",
    "nombre_titular": "Juan Garcia Lopez",
    "dni_cif_titular": "12345678A",
    "tipo_titular": "Propietario",
    "comercializadora": "Endesa",
    "tarifa_actual": "2.0A",
    "potencia_kw": 5.5,
    "consumo_kwh": 12000,
    "precio_energia": 0.18,
    "precio_potencia": 0.42,
    "importe_factura": 2150,
    "fecha_factura": "2026-04-05",
    "fecha_renovacion": "2026-07-05",
    "permanencia_meses": 3
  },
  "coste_anual_actual": 2335.5,
  "status": "success",
  "message": "Factura analizada y Punto de Suministro actualizado",
  "ready_for_comparison": true
}
```

Este contexto es recibido por WF-02 para calcular comparativas.

---

## Manejo de Errores

| Situacion | Comportamiento |
|-----------|---|
| PDF descarga falla | HTTP Error, workflow falla |
| Claude no retorna JSON valido | Code node intenta regex, si falla, throw error |
| Campos requeridos faltantes | Throw error, workflow falla |
| Multiples PS con mismo CUPS | Error explicito, requiere intervencion manual |
| No existe PS | Crea nuevo PS con estado "En Revision" |
| PS eexiste | Actualiza datos de factura mas reciente |

---

## Notas Tecnicas

1. **Idempotencia:** WF-01 puede ejecutarse multiples veces. Si llega la misma factura 2 veces, actualiza los datos (no crea duplicados).

2. **CUPS como Identificador:** El CUPS es el identificador unico global del Punto de Suministro. Nunca cambia.

3. **Nombre Titular IMPORTANTE:** Es el nombre exacto de quien aparece en la factura. Este es el que se mostrara en el PDF comparativa.

4. **Tipo Titular:** Siempre validar contra los SELECT validos de Notion:
   - Propietario
   - Arrendatario
   - Autorizado
   - Otro

5. **Coste Anual:** Se calcula aqui y se propaga al contexto. WF-02 lo usara para comparar contra otras tarifas.

6. **Permanencia:** Si Claude no puede extraerla, se usa 0 (sin compromiso).
