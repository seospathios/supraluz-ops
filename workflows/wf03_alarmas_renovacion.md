# WF-03: Alarmas de Renovación Próxima

## Objetivo

WF-03 ejecuta diariamente a las 8:00 AM para identificar todos los Puntos de Suministro cuyo contrato vence en los próximos 30 días. Agrupa por cliente y notifica a Domin con lista detallada para que genere comparativas proactivas.

---

## Trigger

**Schedule Trigger**
- **Frequency:** Diariamente
- **Time:** 08:00 AM (timezone local)
- **No trigger de WF-00:** Este workflow es independiente y se ejecuta por schedule

---

## Nodos

### 1. Schedule Trigger (Diario 8:00 AM)
**Tipo:** Schedule Trigger
**Posición:** (100, 200)

Ejecuta automáticamente cada día a las 8:00 AM.

---

### 2. HTTP POST: Query PS - Renovación en 30 días
**Tipo:** HTTP Request
**Posición:** (350, 200)

Consulta todos los PS cuya Fecha Renovación está entre hoy y hoy+30 días.

**Endpoint:** `https://api.notion.com/v1/databases/7648879b-8d9d-4a52-b669-20f485b07ddf/query`

**Sorts:** Por `Fecha Renovación` ASC

**Output:** Lista de PS con renovación próxima

---

### 3. Code: Procesar y Agrupar Renovaciones
**Tipo:** Code (JavaScript)
**Posición:** (600, 200)

Extrae datos relevantes de cada PS y agrupa por cliente.

**Output:** Renovaciones individuales + agrupadas por cliente

---

### 4. Code: Formatear Mensaje WhatsApp
**Tipo:** Code (JavaScript)
**Posición:** (850, 200)

Genera mensaje legible para WhatsApp con urgencia visual.

**Urgencia Visual:**
- URGENTE: 1-7 dias
- PROXIMO: 8-14 dias
- NORMAL: 15-30 dias

---

### 5. HTTP POST: WhatsApp - Enviar a Domin
**Tipo:** HTTP Request
**Posición:** (1100, 200)

Envía mensaje formateado a Domin por WhatsApp.

**Endpoint:** `https://api.whatsapp.com/send`

**To:** `{{env.WHATSAPP_DOMIN_NUMBER}}`

---

### 6. Code: Log Final
**Tipo:** Code (JavaScript)
**Posición:** (1350, 200)

Registra resultado de la ejecución.

---

## Datos que Extrae de Cada PS

| Campo | Origen Notion | Descripcion |
|-------|---|---|
| ps_id | ID de pagina | Identificador unico |
| cups | CUPS (title) | Codigo de suministro |
| nombre_titular | Nombre Titular | Para mensaje |
| fecha_renovacion | Fecha Renovacion | Cuando vence |
| dias_para_vencer | Calculo | Urgencia |
| tarifa_actual | Tarifa Actual | Tarifa vigente |
| comercializadora_actual | Comercializadora Actual | Proveedor actual |

---

## Scheduling

- **Frecuencia:** Diaria
- **Hora:** 08:00 AM
- **Timezone:** Configuracion local de n8n (debe ser Espana)

---

## Variables de Entorno Requeridas

```
NOTION_TOKEN              = "noti_***"
WHATSAPP_API_TOKEN        = "***"
WHATSAPP_DOMIN_NUMBER     = "+34612345678"
```
