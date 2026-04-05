# Supraluz-Ops: IDs de Notion para Uso en n8n

Este archivo contiene todos los IDs necesarios para conectar n8n con las bases de datos de Notion de Supraluz-Ops.

---

## Data Sources (Colecciones)

```
CLIENTES
├ Database ID: (obtener del workspace)
├ Data Source ID: 9db380cf-6310-4a9f-a5a7-e97d18db9e75
├ URL Notion: https://www.notion.so/[workspace]/Clientes-9db380cf6310...
└ Descripción: Base de datos de clientes y contactos principales

PUNTOS DE SUMINISTRO
├ Database ID: (obtener del workspace)
├ Data Source ID: 7648879b-8d9d-4a52-b669-20f485b07ddf
├ URL Notion: https://www.notion.so/[workspace]/Puntos-de-Suministro-7648879b...
└ Descripción: Suministros eléctricos con datos técnicos y contractuales

HISTORIAL COMPARATIVAS
├ Database ID: (obtener del workspace)
├ Data Source ID: d4917696-feb6-44fc-be55-9038ca6cbcb4
├ URL Notion: https://www.notion.so/[workspace]/Historial-Comparativas-d491769...
└ Descripción: Registro de comparativas generadas y resultados

TARIFAS DEL MERCADO
├ Database ID: (obtener del workspace)
├ Data Source ID: 5b881746-bd78-403e-84e5-24a5924da649
├ URL Notion: https://www.notion.so/[workspace]/Tarifas-del-Mercado-5b88174...
└ Descripción: Portfolio de tarifas disponibles para ofertas

COMUNICACIONES
├ Database ID: (obtener del workspace)
├ Data Source ID: 9ff3c108-0907-4267-b49d-89254ef20b58
├ URL Notion: https://www.notion.so/[workspace]/Comunicaciones-9ff3c108...
└ Descripción: Log de contactos y comunicaciones con clientes
```

---

## Campos Select - Valores Válidos

### CLIENTES - Tipo
```json
{
  "Individual": "Individual",
  "Empresa": "Empresa"
}
```

### CLIENTES - Industria
```json
{
  "Residencial": "Residencial",
  "PYME": "PYME",
  "Comercial": "Comercial",
  "Industrial": "Industrial",
  "Otros": "Otros"
}
```

### CLIENTES - Último Canal
```json
{
  "Llamada": "Llamada",
  "WhatsApp": "WhatsApp",
  "Email": "Email",
  "Presencial": "Presencial"
}
```

### CLIENTES - Estado
```json
{
  "Activo": "Activo",
  "Prospecto": "Prospecto",
  "Inactivo": "Inactivo",
  "Ganado": "Ganado",
  "Perdido": "Perdido"
}
```

### PUNTOS DE SUMINISTRO - Tipo Titular
```json
{
  "Propietario": "Propietario",
  "Arrendatario": "Arrendatario",
  "Autorizado": "Autorizado",
  "Otro": "Otro"
}
```

### PUNTOS DE SUMINISTRO - Tipo Suministro
```json
{
  "Electricidad": "Electricidad",
  "Gas": "Gas",
  "Ambos": "Ambos"
}
```

### PUNTOS DE SUMINISTRO - Estado
```json
{
  "Activo": "Activo",
  "En Revisión": "En Revisión",
  "Procesando": "Procesando",
  "Inactivo": "Inactivo"
}
```

### TARIFAS DEL MERCADO - Tipo Tarifa
```json
{
  "2.0A": "2.0A",
  "3.0A": "3.0A",
  "3.1A": "3.1A",
  "6.1": "6.1",
  "6.2": "6.2",
  "6.3": "6.3",
  "6.4": "6.4",
  "Otra": "Otra"
}
```

### TARIFAS DEL MERCADO - Sector Apto (Multi-Select)
```json
{
  "Individual": "Individual",
  "PYME": "PYME",
  "Comercial": "Comercial",
  "Industrial": "Industrial"
}
```

### HISTORIAL COMPARATIVAS - Estado
```json
{
  "Generada": "Generada",
  "Aprobada": "Aprobada",
  "Enviada": "Enviada",
  "Aceptada": "Aceptada",
  "Rechazada": "Rechazada"
}
```

### HISTORIAL COMPARATIVAS - Canal Envío
```json
{
  "WhatsApp": "WhatsApp",
  "Email": "Email",
  "Presencial": "Presencial",
  "SMS": "SMS"
}
```

### COMUNICACIONES - Tipo
```json
{
  "Llamada": "Llamada",
  "WhatsApp": "WhatsApp",
  "Email": "Email",
  "Presencial": "Presencial",
  "SMS": "SMS"
}
```

### COMUNICACIONES - Dirección
```json
{
  "Entrante": "Entrante",
  "Saliente": "Saliente"
}
```

### COMUNICACIONES - Resultado
```json
{
  "Positivo": "Positivo",
  "Neutral": "Neutral",
  "Negativo": "Negativo",
  "Pendiente": "Pendiente"
}
```

---

## Estructura para HTTP Requests a Notion API

### Headers Estándar
```json
{
  "Authorization": "Bearer {{env.NOTION_TOKEN}}",
  "Notion-Version": "2022-06-28",
  "Content-Type": "application/json"
}
```

### Base URL
```
https://api.notion.com/v1
```

### Endpoints Utilizados

#### Lectura de Registros
```
GET /databases/{database_id}/query
POST /databases/{database_id}/query
```

#### Crear Registro
```
POST /pages
```

#### Actualizar Registro
```
PATCH /pages/{page_id}
```

#### Obtener Propiedades Database
```
GET /databases/{database_id}
```

---

## Ejemplos de Query Filters

### Filtrar Tarifas Activas y Aptas para Comparativa
```json
{
  "filter": {
    "and": [
      {
        "property": "Activa",
        "checkbox": {
          "equals": true
        }
      },
      {
        "property": "Apta para Comparativa",
        "checkbox": {
          "equals": true
        }
      },
      {
        "property": "Fecha Vigencia Inicio",
        "date": {
          "on_or_before": "{{now}}"
        }
      },
      {
        "property": "Fecha Vigencia Fin",
        "date": {
          "on_or_after": "{{now}}"
        }
      }
    ]
  }
}
```

### Filtrar Puntos de Suministro con Renovación Próxima (30 días)
```json
{
  "filter": {
    "and": [
      {
        "property": "Estado",
        "select": {
          "does_not_equal": "Inactivo"
        }
      },
      {
        "property": "Fecha Renovación",
        "date": {
          "on_or_after": "{{today}}"
        }
      },
      {
        "property": "Fecha Renovación",
        "date": {
          "before": "{{dateAdd(today, 30)}}"
        }
      }
    ]
  }
}
```

### Buscar Cliente por Nombre
```json
{
  "filter": {
    "property": "Nombre",
    "title": {
      "contains": "{{nombre_cliente}}"
    }
  }
}
```

### Buscar Punto de Suministro por CUPS
```json
{
  "filter": {
    "property": "CUPS",
    "title": {
      "equals": "{{cups}}"
    }
  }
}
```

---

## Estructura para Crear/Actualizar Registros

### Format de Properties para POST /pages

```json
{
  "parent": {
    "database_id": "{{dataSourceId}}"
  },
  "properties": {
    "Nombre": {
      "title": [
        {
          "text": {
            "content": "{{nombre}}"
          }
        }
      ]
    },
    "Email": {
      "email": "{{email}}"
    },
    "Teléfono": {
      "phone_number": "{{telefono}}"
    },
    "Tipo": {
      "select": {
        "name": "Individual"
      }
    },
    "Industria": {
      "select": {
        "name": "PYME"
      }
    },
    "Estado": {
      "select": {
        "name": "Activo"
      }
    },
    "Potencia Instalada Total": {
      "number": 10.5
    },
    "Consumo Anual Estimado": {
      "number": 15000
    },
    "Gasto Actual Anual Estimado": {
      "number": 2250
    },
    "Último Contacto": {
      "date": {
        "start": "2026-04-05"
      }
    },
    "Próximo Contacto Programado": {
      "date": {
        "start": "2026-04-12"
      }
    },
    "Notas": {
      "rich_text": [
        {
          "text": {
            "content": "{{notas}}"
          }
        }
      ]
    }
  }
}
```

### Format para PATCH /pages/{page_id}

```json
{
  "properties": {
    "Estado": {
      "select": {
        "name": "Procesando"
      }
    },
    "Consumo Anual": {
      "number": 18500
    },
    "Fecha Renovación": {
      "date": {
        "start": "2026-07-05"
      }
    },
    "Último Contacto": {
      "date": {
        "start": "{{now}}"
      }
    }
  }
}
```

---

## Mapeo de Campos en Relaciones

### Relacionar Punto de Suministro con Cliente
En PATCH de PS:
```json
{
  "properties": {
    "Cliente": {
      "relation": [
        {
          "id": "{{clientePageId}}"
        }
      ]
    }
  }
}
```

### Relacionar Comparativa con Tarifa Recomendada
En POST/PATCH de Comparativa:
```json
{
  "properties": {
    "Tarifa Recomendada": {
      "relation": [
        {
          "id": "{{tarifaPageId}}"
        }
      ]
    },
    "Top 3 Tarifas Recomendadas": {
      "relation": [
        {
          "id": "{{tarifa1PageId}}"
        },
        {
          "id": "{{tarifa2PageId}}"
        },
        {
          "id": "{{tarifa3PageId}}"
        }
      ]
    }
  }
}
```

---

## Variables de Entorno Requeridas en n8n

```
NOTION_TOKEN = "noti_***" (Secret de Notion API)
CLAUDE_API_KEY = "sk-***" (API Key Claude)
GDRIVE_FOLDER_ID = "1a2b3c4d..." (Folder ID Google Drive)
WHATSAPP_DOMIN_NUMBER = "+34612345678"
WHATSAPP_API_TOKEN = "***" (Token WhatsApp Business)
PYTHON_PDF_ENDPOINT = "https://internal.supraluz.local/generate-pdf"
WF01_ID = "123456" (ID del workflow WF-01 en n8n)
WF02_ID = "123457" (ID del workflow WF-02 en n8n)
WF03_ID = "123458" (ID del workflow WF-03 en n8n)
WF04_ID = "123459" (ID del workflow WF-04 en n8n)
WF05_ID = "123460" (ID del workflow WF-05 en n8n)
```

---

## Nota Importante

Los IDs de Data Source y Database se usan de forma intercambiable en Notion API v1.
Para operaciones en n8n con Notion nodes predefinidos, usar preferentemente:
- **Database ID**: Para el nodo "Notion" oficial de n8n
- **Data Source ID**: Para HTTP Requests directos a la API

En caso de duda, obtener el ID correcto desde la URL de Notion:
```
https://www.notion.so/[workspace]/[DatabaseName]-[ID]
                                             ↑
                                      Copiar este ID
```

