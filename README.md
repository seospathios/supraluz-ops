# supraluz-ops

Repositorio de automatizaciones n8n para **Supraluz** — CRM de comparación de tarifas eléctricas.

## Descripción

Sistema automatizado que gestiona el ciclo completo de comparativa de tarifas: desde la recepción de facturas por WhatsApp hasta el envío de propuestas personalizadas a clientes, pasando por análisis con IA (Claude API), generación de PDFs y aprobación por parte de Domin.

## Arquitectura

```
WhatsApp / Webhook
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│          WF-00  Dispatcher Central (supraluz-events)     │
│  event_type → Switch → executeWorkflow                   │
└──────┬───────┬───────┬───────┬───────┬───────────────────┘
       │       │       │       │       │
       ▼       ▼       ▼       ▼       ▼
    WF-01   WF-02   WF-03   WF-04   WF-05
  Análisis Compa-  Alarmas  Input   Envío
  Factura  rativa  Renov.  WhatsApp Propuesta
           PDF
```

## Índice de Workflows

| ID    | Nombre                   | Trigger              | Descripción |
|-------|--------------------------|----------------------|-------------|
| WF-00 | Dispatcher Central       | Webhook POST         | Enruta eventos a sub-workflows |
| WF-01 | Análisis Factura         | executeWorkflow      | Claude API extrae 14 campos de la factura |
| WF-02 | Comparativa PDF          | executeWorkflow      | Calcula ahorro, genera PDF, sube a Drive |
| WF-03 | Alarmas Renovación       | Schedule 08:00       | Alerta a Domin por WhatsApp contratos próximos |
| WF-04 | Input WhatsApp Domin     | executeWorkflow      | Claude interpreta texto libre de Domin |
| WF-05 | Envío Propuesta          | executeWorkflow      | Envía PDF aprobado al cliente por WhatsApp |

## Estructura del Repositorio

```
supraluz-ops/
├── README.md
├── workflows/
│   ├── wf00_dispatcher_supraluz.json
│   ├── wf00_dispatcher_supraluz.md
│   ├── wf01_analisis_factura.json
│   ├── wf01_analisis_factura.md
│   ├── wf02_comparativa_pdf.json
│   ├── wf02_comparativa_pdf.md
│   ├── wf03_alarmas_renovacion.json
│   ├── wf03_alarmas_renovacion.md
│   ├── wf04_input_whatsapp.json
│   ├── wf04_input_whatsapp.md
│   ├── wf05_envio_propuesta.json
│   └── wf05_envio_propuesta.md
└── docs/
    ├── ARCHITECTURE.md
    ├── WEBHOOK_PAYLOAD_SCHEMA.md
    └── NOTION_IDS.md
```

## Bases de Datos Notion

| Base de Datos        | Data Source ID |
|----------------------|----------------|
| Clientes             | `9db380cf-6310-4a9f-a5a7-e97d18db9e75` |
| Puntos de Suministro | `7648879b-8d9d-4a52-b669-20f485b07ddf` |
| Historial Comparativas | `d4917696-feb6-44fc-be55-9038ca6cbcb4` |
| Tarifas del Mercado  | `5b881746-bd78-403e-84e5-24a5924da649` |
| Comunicaciones       | `9ff3c108-0907-4267-b49d-89254ef20b58` |

## Evento Webhook (WF-00)

Endpoint: `POST /webhook/supraluz-events`

| `event_type`          | Sub-workflow |
|-----------------------|-------------|
| `factura_recibida`    | WF-01 |
| `comparativa_lista`   | WF-02 |
| `renovacion_proxima`  | WF-03 |
| `whatsapp_domin_input`| WF-04 |
| `propuesta_aprobada`  | WF-05 |

## Reglas de Negocio Críticas

1. **Nombre en PDF**: SIEMPRE usar `Nombre titular` del Punto de Suministro. NUNCA el alias del Cliente.
2. **Retrocomisión**: PROHIBIDA en cualquier documento enviado al cliente.
3. **Nomenclatura PDF**: `Supraluz_C[N]_[Nombre_Titular_completo].pdf`
4. **Footer PDF**: `Tel: 611 015 050 · www.supraluz.com · Instagram: @supraluz_esp`

## Fórmulas de Cálculo

```
coste_actual_anual = (consumo_kwh × precio_energia_actual) + (potencia_kw × 365 × precio_potencia_actual)
coste_nuevo_anual  = (consumo_kwh × precio_energia_nueva)  + (potencia_kw × 365 × precio_potencia_nueva)
ahorro_estimado    = coste_actual_anual − coste_nuevo_anual
```

## Integraciones

- **Claude API** (claude-sonnet-4-6): análisis de facturas e interpretación de mensajes
- **WhatsApp Business** (360dialog): recepción y envío de mensajes
- **Google Drive**: almacenamiento de PDFs generados
- **Notion**: base de datos central (Clientes, PS, Comparativas, Tarifas, Comunicaciones)
- **ReportLab** (Python): generación de PDFs de propuesta

## Stack

- n8n (self-hosted)
- Notion API
- Claude API (Anthropic)
- 360dialog WhatsApp Business API
- Google Drive API
- Python + ReportLab (via n8n Code node)
