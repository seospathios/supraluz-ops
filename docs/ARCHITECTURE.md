# Supraluz-Ops: Arquitectura Completa del CRM Energético

## Descripción General

**Supraluz-Ops** es un CRM especializado en comparación de tarifas eléctricas y optimización de contratos de suministro. El sistema automatiza el análisis de facturas, comparación de ofertas y gestión de renovaciones.

**Ciclo principal:**
1. Recepción de PDF de factura (WhatsApp/Email)
2. Extracción de datos vía Claude API
3. Comparación contra portfolio de Tarifas del Mercado
4. Cálculo de ahorro anual
5. Generación de PDF comparativa
6. Envío a Domin para aprobación comercial
7. Envío de propuesta al cliente por WhatsApp

---

## Schemas de Base de Datos (Notion)

### 1. CLIENTES
**Data Source ID:** `9db380cf-6310-4a9f-a5a7-e97d18db9e75`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| **Nombre** | Title | Nombre del cliente/contacto principal |
| **Email** | Email | Email de contacto |
| **Teléfono** | Phone Number | Teléfono principal (formato internacional) |
| **DNI/CIF** | Text | Identificador fiscal |
| **Tipo** | Select | Individual / Empresa |
| **Industria** | Select | Residencial / PYME / Comercial / Industrial / Otros |
| **Potencia Instalada Total** | Number | kW (suma de todos los PS) |
| **Consumo Anual Estimado** | Number | kWh (suma de todos los PS) |
| **Gasto Actual Anual Estimado** | Number | € (suma de todos los PS) |
| **Número PS Activos** | Rollup | Count de Puntos de Suministro con Estado != Inactivo |
| **Últimas Comparativas** | Relation | Link a Historial Comparativas |
| **Puntos de Suministro** | Relation | Link a Puntos de Suministro |
| **Último Contacto** | Date | Fecha último contacto |
| **Último Canal** | Select | Llamada / WhatsApp / Email / Presencial |
| **Próximo Contacto Programado** | Date | Fecha próxima acción |
| **Estado** | Select | Activo / Prospecto / Inactivo / Ganado / Perdido |
| **Notas** | Rich Text | Observaciones comerciales |
| **Fecha Creación** | Created Time | Auto |
| **Última Actualización** | Last Edited Time | Auto |

---

### 2. PUNTOS DE SUMINISTRO
**Data Source ID:** `7648879b-8d9d-4a52-b669-20f485b07ddf`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| **CUPS** | Title | Código de Punto de Suministro (identificador único) |
| **Cliente** | Relation | Link a Clientes |
| **CUPS Empresa** | Text | CUPS en formato empresa (si diferente) |
| **Tipo Titular** | Select | Propietario / Arrendatario / Autorizado / Otro |
| **Nombre Titular** | Text | Nombre exacto del titular (de factura) |
| **DNI/CIF Titular** | Text | Identificador del titular |
| **Comercializadora Actual** | Text | Nombre comercializadora (ej: Endesa, Iberdrola) |
| **Tarifa Actual** | Text | Nombre de la tarifa (ej: 2.0A, 3.0A) |
| **Potencia Contratada** | Number | kW |
| **Consumo Anual** | Number | kWh (del último año) |
| **Precio Energía Actual** | Number | €/kWh |
| **Precio Potencia Actual** | Number | €/kW/día |
| **Importe Factura Última** | Number | € (de PDF más reciente) |
| **Coste Anual Actual** | Formula | consumo_kwh * precio_energia + potencia_kw * 365 * precio_potencia |
| **Fecha Factura Última** | Date | Fecha de factura más reciente procesada |
| **Fecha Renovación** | Date | Fecha cuando vence contrato actual |
| **Permanencia Meses** | Number | Meses restantes de compromiso |
| **Tipo Suministro** | Select | Electricidad / Gas / Ambos |
| **Estado** | Select | Activo / En Revisión / Procesando / Inactivo |
| **Teléfono Guest** | Phone Number | Teléfono del contacto en cliente |
| **Email Guest** | Email | Email del contacto en cliente |
| **Factura Última URL** | URL | Link a PDF en Google Drive o storage |
| **Última Comparativa Generada** | Relation | Link a la comparativa más reciente |
| **Comparativas Asociadas** | Relation | Link a todas las Comparativas |
| **Historial Tarifas** | Relation | Link a histórico de cambios |
| **Notas Técnicas** | Rich Text | Observaciones técnicas del suministro |
| **Fecha Creación** | Created Time | Auto |
| **Última Actualización** | Last Edited Time | Auto |

---

### 3. HISTORIAL COMPARATIVAS
**Data Source ID:** `d4917696-feb6-44fc-be55-9038ca6cbcb4`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| **ID Comparativa** | Title | ID único (Supraluz_C[N]) |
| **Punto de Suministro** | Relation | Link a PS |
| **Cliente** | Relation | Link a Clientes |
| **Fecha Generación** | Date | Cuándo se generó la comparativa |
| **Tarifa Actual** | Text | Tarifa del PS en momento comparativa |
| **Tarifa Recomendada** | Relation | Link a Tarifas del Mercado (mejor opción) |
| **Ahorro Estimado Anual** | Number | € (coste_actual - coste_nuevo) |
| **Porcentaje Ahorro** | Formula | ahorro_estimado / coste_anual_actual * 100 |
| **Coste Actual Anual** | Number | € |
| **Coste Nuevo Anual** | Number | € |
| **Top 3 Tarifas Recomendadas** | Relation | Link a Tarifas (opciones 1, 2, 3) |
| **PDF Comparativa** | URL | Link a PDF en Google Drive (Supraluz_C[N]_[Nombre].pdf) |
| **Resumen Análisis** | Rich Text | Párrafos de recomendación |
| **Propuesta Enviada al Cliente** | Checkbox | Fue enviado PDF al número del cliente |
| **Fecha Envío Cliente** | Date | Cuándo se envió |
| **Aprobado por Domin** | Checkbox | Comercial aprobó para envío |
| **Fecha Aprobación Domin** | Date | Cuándo aprobó |
| **Canal Envío** | Select | WhatsApp / Email / Presencial / SMS |
| **Estado** | Select | Generada / Aprobada / Enviada / Aceptada / Rechazada |
| **Respuesta Cliente** | Text | Feedback del cliente |
| **Fecha Respuesta** | Date | Cuándo respondió cliente |
| **Notas Comerciales** | Rich Text | Observaciones de Domin |
| **Última Actualización** | Last Edited Time | Auto |

---

### 4. TARIFAS DEL MERCADO
**Data Source ID:** `5b881746-bd78-403e-84e5-24a5924da649`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| **Nombre Tarifa** | Title | Nombre comercial (ej: ENDESA TARIFA FIJA) |
| **Comercializadora** | Text | Nombre comercializadora |
| **Tipo Tarifa** | Select | 2.0A / 3.0A / 3.1A / 6.1 / 6.2 / 6.3 / 6.4 / Otra |
| **Precio Energía** | Number | €/kWh (vigente hoy) |
| **Precio Potencia** | Number | €/kW/día (vigente hoy) |
| **Permanencia** | Number | Meses (0 = sin compromiso) |
| **Comisión Supraluz** | Number | % (margen de Supraluz, NUNCA restar de cliente) |
| **Fecha Vigencia Inicio** | Date | Desde cuándo son válidos estos precios |
| **Fecha Vigencia Fin** | Date | Hasta cuándo |
| **Activa** | Checkbox | Tarifa disponible para ofertas |
| **Apta para Comparativa** | Checkbox | Puede aparecer en comparativas automáticas |
| **Descripción Comercial** | Rich Text | Pitch de venta |
| **Condiciones Especiales** | Text | Descuentos, bonificaciones, etc. |
| **Requisitos Mínimos** | Text | Consumo mínimo, potencia máxima, etc. |
| **Penalización Rescisión** | Number | € (si vence antes del plazo) |
| **Sector Apto** | Multi Select | Individual / PYME / Comercial / Industrial |
| **Ranking Recomendación** | Number | Posición en ranking (1=mejor) |
| **Última Actualización** | Last Edited Time | Auto |

---

### 5. COMUNICACIONES
**Data Source ID:** `9ff3c108-0907-4267-b49d-89254ef20b58`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| **ID Comunicación** | Title | ID único (auto-generated) |
| **Cliente** | Relation | Link a Clientes |
| **Tipo** | Select | Llamada / WhatsApp / Email / Presencial / SMS |
| **Dirección** | Select | Entrante / Saliente |
| **Asunto** | Text | Breve resumen |
| **Contenido** | Rich Text | Detalles de la comunicación |
| **Resultado** | Select | Positivo / Neutral / Negativo / Pendiente |
| **Próxima Acción** | Text | Qué hacer después |
| **Fecha Próxima Acción** | Date | Cuándo |
| **Realizado Por** | Text | Nombre del usuario (ej: Domin) |
| **Fecha Hora** | Created Time | Cuándo ocurrió |
| **Comparativa Asociada** | Relation | Link a comparativa (si aplica) |
| **Punto de Suministro Asociado** | Relation | Link a PS (si aplica) |
| **Notas Internas** | Text | Solo para equipo |

---

## IDs de Notion - Referencia Rápida

```
CLIENTES                           → 9db380cf-6310-4a9f-a5a7-e97d18db9e75
PUNTOS DE SUMINISTRO              → 7648879b-8d9d-4a52-b669-20f485b07ddf
HISTORIAL COMPARATIVAS            → d4917696-feb6-44fc-be55-9038ca6cbcb4
TARIFAS DEL MERCADO               → 5b881746-bd78-403e-84e5-24a5924da649
COMUNICACIONES                    → 9ff3c108-0907-4267-b49d-89254ef20b58
```

---

## Mapa de Relaciones

```
CLIENTES
  ├─ N Puntos de Suministro (1:N)
  ├─ N Comparativas (1:N)
  └─ N Comunicaciones (1:N)

PUNTOS DE SUMINISTRO
  ├─ 1 Cliente (N:1)
  ├─ N Comparativas (1:N)
  └─ 1 Última Comparativa (1:1 optional)

HISTORIAL COMPARATIVAS
  ├─ 1 Cliente (N:1)
  ├─ 1 Punto de Suministro (N:1)
  ├─ 1 Tarifa Recomendada (N:1)
  └─ N Top 3 Tarifas (N:N)

TARIFAS DEL MERCADO
  └─ N Comparativas (1:N)

COMUNICACIONES
  ├─ 1 Cliente (N:1)
  ├─ 1 Punto de Suministro (N:1) [optional]
  └─ 1 Comparativa (N:1) [optional]
```

---

## Reglas de Negocio Críticas

### Regla 1: Cálculo de Coste Anual
```
coste_anual_actual = (consumo_kwh * precio_energia) + (potencia_kw * 365 * precio_potencia)
```
- Se usa para todas las comparaciones
- Consumo de últimos 12 meses (de factura más reciente)
- Precios vigentes de tarifa actual

### Regla 2: Comisión Supraluz NO se Resta al Cliente
- La comisión es **interna** (entre Supraluz y comercializadora)
- **NUNCA** se descuenta del precio mostrado al cliente
- El cliente ve siempre el precio final comercializadora
- En PDF comparativa: `precio_energia` y `precio_potencia` son brutos

### Regla 3: Nombre Titular en Comparativa
- **Siempre** usar `Nombre Titular` del Punto de Suministro
- No usar nombre del Cliente (que puede ser intermediario)
- Validation: si `Nombre Titular` vac�;o → no generar PDF

### Regla 4: Filtros de Tarifas en Comparativa
Solo se incluyen tarifas donde:
- `Activa` = TRUE
- `Apta para Comparativa` = TRUE
- `Fecha Vigencia Inicio` <= hoy <= `Fecha Vigencia Fin`
- `Sector Apto` contiene sector del cliente

### Regla 5: Ranking de Mejores Tarifas
Ordenar por ahorro DESC, secundaria por `Ranking Recomendación` ASC
- Top 1: Recomendada
- Top 2-3: Alternativas

### Regla 6: Notificación a Domin
- Domin recibe notificación **solo una vez** por comparativa
- Solo después de validación (si ahorro >= €100 anual recomendado)
- Texto: "Comparativa lista - Cliente: [NOMBRE] - CUPS: [CUPS] - Ahorro: €X/año"

### Regla 7: Estado de Punto de Suministro
Transiciones válidas:
```
Activo → En Revisión (cuando llega factura)
       → Procesando (cuando se genera comparativa)
       → Activo (cuando se cierra ciclo sin cambio)

En Revisión → Procesando → Activo (ciclo normal)
           ↓            ↓
        Inactivo (error terminal)
```

---

## Flujos Operativos

### Flujo 1: Recepción y Análisis de Factura
1. PDF llega por WhatsApp/Email (webhook enruta a WF-00)
2. WF-00 → WF-01 (event_type: `factura_recibida`)
3. WF-01 extrae datos con Claude API
4. WF-01 actualiza Punto de Suministro (estado: En Revisión)
5. WF-01 retorna contexto enriquecido a WF-00

### Flujo 2: Comparativa y Aprobación Comercial
1. WF-00 recibe trigger `comparativa_lista`
2. WF-00 → WF-02 (genera comparativa)
3. WF-02 descarga tarifas de Notion
4. WF-02 calcula ahorro para cada tarifa
5. WF-02 genera PDF con Claude (párrafos análisis)
6. WF-02 sube PDF a Google Drive
7. WF-02 crea registro en Historial Comparativas
8. WF-02 envía WhatsApp a Domin: "Comparativa lista"
9. Domin aprueba en WhatsApp → WF-00 recibe `propuesta_aprobada`

### Flujo 3: Renovación Proactiva
1. WF-03 ejecuta cada día 8:00 AM
2. Obtiene PS con Fecha Renovación en próximos 30 días
3. Agrupa por cliente
4. Envía WhatsApp a Domin: "Renovación próxima: [LISTA]"
5. Domin puede generar nueva comparativa manualmente

### Flujo 4: Input WhatsApp de Domin
1. Domin envía WhatsApp libre (ej: "Llamé a García, dijo que estudia")
2. WF-04 procesa mensaje con Claude
3. Estructura datos: canal, resultado, próxima acción
4. Crea Comunicación en Notion
5. Actualiza Clientes (último contacto, Próximo contacto)
6. Confirma a Domin: "✅ Registrado"

---

## Nomenclatura de Archivos

### PDFs Comparativos
Patrón: `Supraluz_C[N]_[Nombre_Titular].pdf`

Ejemplos:
- `Supraluz_C00001_Juan_García_López.pdf`
- `Supraluz_C00002_EMPRESA_ACME_SL.pdf`

Donde:
- `[N]` = Número secuencial (auto-increment desde Notion)
- `[Nombre_Titular]` = Nombre completo titular (sin caracteres especiales)

### Ubicación
- Google Drive: `/Supraluz-Comparativas/[Año]/[Mes]/Supraluz_C[N]_[...].pdf`

---

## Stack Tecnológico por Fase

### Fase 1: Ingestimo (Actual)
- **n8n** v3: Orquestación workflows
- **Claude API** (Sonnet 4.6): Extracción de datos, análisis
- **Notion API*j: Lectura/escritura BD
- **Google Drive API*j: Almacenamiento PDFs

### Fase 2: Generación PDF (Next)
- **Python 3.11**: ReportLab para generación PDF
- Endpoint interno: `POST /generate-pdf`
- Input: JSON cliente + comparativa
- Output: PDF binario

### Fase 3: WhatsApp (Next)
- **WhatsApp Business API**: Envío mensajes/PDFs
- Twilio o similar como intermediario
- Webhook para recepción

### Fase 4: Machine Learning (Future)
- **Clustering**: Segmentación clientes por perfil consumo
- **Recomendación**: Predicción tarifa ideal por cliente
- **Churn**: Alertas de riesgo pérdida cliente

### Fase 5: Analytics (Future)
- **Looker Studio**: Dashboards comerciales
- **BigQuery**: Data warehouse agregado

---

## Convenciones de Código en n8n

1. **Node naming**: camelCase, ej: `extractFacturaData`, `notifyDomin`
2. **Variable naming**: UPPER_SNAKE_CASE para env vars, camelCase para datos
3. **Code nodes**: Usar `$input.first()` para acceder al payload
4. **Error handling**: Todos los HTTP nodes con catch + fallback
5. **Logging**: Code node final con trazabilidad completa

---

## Referencias Cruzadas

- **Webhook Schema**: Ver `supraluz_WEBHOOK_PAYLOAD_SCHEMA.md`
- **IDs Notion Detallados**: Ver `supraluz_NOTION_IDS.md`
- **WF-00 Dispatcher**: Ver `wf00_dispatcher_supraluz.md`
- **Workflows Específicos**: Ver `wf0X_*.md` correspondiente

