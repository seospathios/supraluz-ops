# WF-02: Comparativa PDF y Generación

## Objetivo

WF-02 recibe el contexto enriquecido de WF-01 y:
1. Obtiene todas las tarifas activas del portfolio
2. Calcula ahorro para cada tarifa vs tarifa actual
3. Genera párrafos de análisis con Claude
4. Genera PDF comparativa (internamente con Python)
5. Sube PDF a Google Drive
6. Crea registro en Notion Historial Comparativas
7. Notifica a Domin por WhatsApp

---

## Trigger

**Execute Workflow Trigger** - Llamado por WF-00 cuando `event_type == "comparativa_lista"`

Input: Contexto enriquecido de WF-01
```json
{
  "correlation_id": "UUID",
  "ps_id": "UUID",
  "ps_data": { ... },
  "coste_anual_actual": 2335.5,
  "ready_for_comparison": true
}
```

---

## Nodos

### 1. Execute Workflow Trigger
**Tipo:** Trigger
**Posición:** (100, 200)

---

### 2. HTTP POST: Get Tarifas Activas
**Tipo:** HTTP Request
**Posición:** (350, 200)

Obtiene todas las tarifas del portfolio que cumplen criterios.

**Endpoint:** `https://api.notion.com/v1/databases/5b881746-bd78-403e-84e5-24a5924da649/query`

**Filter:**
```json
{
  "and": [
    { "property": "Activa", "checkbox": { "equals": true } },
    { "property": "Apta para Comparativa", "checkbox": { "equals": true } }
  ]
}
```

**Sorts:** Por `Ranking Recomendación` ASC

**Output:** Lista de tarifas con propiedades Notion

---

### 3. Code: Calcular Ahorros
**Tipo:** Code (JavaScript)
**Posición:** (600, 200)

Calcula ahorro anual para cada tarifa usando la fórmula:

```
coste_anual_nuevo = (consumo_kwh * precio_energia) + (potencia_kw * 365 * precio_potencia)
ahorro = coste_actual - coste_anual_nuevo
porcentaje_ahorro = (ahorro / coste_actual) * 100
```

**Código:**
```javascript
const tarifas_result = $input.first().json.results || [];
const input_context = $input.all()[0].json;\n\nconst consumo_kwh = input_context.ps_data.consumo_kwh;\nconst potencia_kw = input_context.ps_data.potencia_kw;\nconst coste_actual = input_context.coste_anual_actual;\n\nconst tarifas = tarifas_result.map(t => {\n  const props = t.properties;\n  const precio_energia = props['Precio Energia']?.number || 0;\n  const precio_potencia = props['Precio Potencia']?.number || 0;\n\n  const coste_nuevo = (consumo_kwh * precio_energia) + (potencia_kw * 365 * precio_potencia);\n  const ahorro = coste_actual - coste_nuevo;\n  const porcentaje_ahorro = (ahorro / coste_actual * 100).toFixed(2);\n\n  return {\n    id: t.id,\n    nombre: props['Nombre Tarifa']?.title[0]?.plain_text || 'Sin nombre',\n    comercializadora: props['Comercializadora']?.rich_text[0]?.plain_text || '',\n    tipo_tarifa: props['Tipo Tarifa']?.select?.name || '',\n    precio_energia: precio_energia,\n    precio_potencia: precio_potencia,\n    permanencia: props['Permanencia']?.number || 0,\n    comision_supraluz: props['Comision Supraluz']?.number || 0,\n    coste_anual: coste_nuevo,\n    ahorro_estimado: ahorro,\n    porcentaje_ahorro: parseFloat(porcentaje_ahorro),\n    ranking: props['Ranking Recomendacion']?.number || 999\n  };\n});\n\n// Ordenar por ahorro DESC\nconst tarifas_ordenadas = tarifas.sort((a, b) => b.ahorro_estimado - a.ahorro_estimado);\n\nreturn [{\n  json: {\n    tarifas: tarifas_ordenadas,\n    coste_actual_anual: coste_actual,\n    top_3: tarifas_ordenadas.slice(0, 3),\n    tarifa_recomendada: tarifas_ordenadas[0],\n    context: input_context\n  }\n}];
```

**Output:**
- `tarifas`: Lista ordenada por ahorro DESC
- `tarifa_recomendada`: Top 1 (mayor ahorro)
- `top_3`: Top 3 tarifas

---

### 4. HTTP POST: Claude API - Analisis
**Tipo:** HTTP Request
**Posición:** (850, 200)

Genera analisis en dos formatos:

1. **Resumen Confidencial** (5 lineas) - Solo para aprobacion interna de Domin
2. **Analisis Completo** (15-20 lineas) - Para enviar al cliente

**Endpoint:** `https://api.anthropic.com/v1/messages`

**Output:** JSON con ambos textos

---

### 5. HTTP POST: Generar PDF (Python)
**Tipo:** HTTP Request
**Posición:** (1100, 200)

Llama a endpoint interno Python para generar PDF.

**Endpoint:** `{{env.PYTHON_PDF_ENDPOINT}}`

**Output:** URL del PDF generado + tamano

---

### 6. Code: Procesar Respuesta PDF
**Tipo:** Code
**Posición:** (1350, 200)

**Genera:**
- `pdf_filename`: `Supraluz_C[N]_[Nombre_Titular].pdf`

---

### 7. HTTP POST: Google Drive - Upload PDF
**Tipo:** HTTP Request
**Posición:** (1600, 200)

**Endpoint:** `https://www.googleapis.com/upload/drive/v3/files`

**Output:** URL publico del PDF en Drive

---

### 8. HTTP POST: Notion - Crear Comparativa
**Tipo:** HTTP Request
**Posición:** (1850, 200)

Crea registro en Historial Comparativas.

**Endpoint:** `https://api.notion.com/v1/pages`

---

### 9. HTTP POST: WhatsApp - Notificar Domin
**Tipo:** HTTP Request
**Posición:** (2100, 200)

Envia notificacion a Domin con resumen ejecutivo.

**To:** `{{env.WHATSAPP_DOMIN_NUMBER}}`

---

## Reglas de Negocio

3. Nombre Titular en PDF: Siempre usar Nombre Titular del PS
4. Nomenclatura PDF: Supraluz_C[N]_[Nombre_Titular].pdf
5. Filtros de Tarifas: Activa = TRUE y Apta para Comparativa = TRUE

---

## Campos que Crea en Notion (Historial Comparativas)

| Campo | Valor | Notas |
|-------|-------|-------|
| ID Comparativa | Supraluz_C00001_Juan_Garcia.pdf | Title |
| Punto de Suministro | Link a PS | Relation |
| Fecha Generacion | Hoy | Date |
| Tarifa Actual | 2.0A | Copiada de PS |
| Tarifa Recomendada | TARIFA FIJA 2026 | Relation a Tarifas |
| Ahorro Estimado Anual | 450 | Number |
| Coste Actual Anual | 2335.5 | Number |
| Coste Nuevo Anual | 1885.5 | Number |
| PDF Comparativa | https://drive.google... | URL |
| Resumen Analisis | [5 lineas confidencial] | Rich Text |
| Estado | Generada | Select |

---

## Variables de Entorno Requeridas

```
NOTION_TOKEN = "noti_***"
CLAUDE_API_KEY = "sk-***"
GDRIVE_TOKEN = "ya29.***"
GDRIVE_FOLDER_ID = "1a2b3c4d..."
PYTHON_PDF_ENDPOINT = "https://internal.supraluz.local/generate-pdf"
WHATSAPP_API_TOKEN = "***"
WHATSAPP_DOMIN_NUMBER = "+34612345678"
```
