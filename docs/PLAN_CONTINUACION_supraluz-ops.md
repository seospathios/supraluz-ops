# PLAN DE CONTINUACIÓN — supraluz-ops

> **Repositorio:** `seospathios/supraluz-ops` (público — pendiente hacer privado)  
> **Descripción:** Automatizaciones n8n para Supraluz — CRM de comparación de tarifas eléctricas  
> **Versión activa:** Sin versión definida aún (proyecto en fase inicial)  
> **Stack:** n8n cloud + Notion (por definir) + Telegram (pendiente)  
> **Última actualización de este documento:** 11 abril 2026

---

## 1. ESTADO ACTUAL

### Situación del repositorio

El repo `seospathios/supraluz-ops` existe en GitHub y contiene estructura básica:

```
supraluz-ops/
├── README.md                     ← Descripción general del proyecto
├── docs/
│   └── NOTION_IDS.md             ← IDs de bases de datos Notion
└── workflows/
    └── wf05_envio_propuesta.md   ← (y otros workflows documentados)
```

### Visibilidad

El repo está actualmente **público**. Se intentó hacer privado pero GitHub bloqueó la acción requiriendo verificación de identidad por email (sudo mode). **Pendiente:** hacer click en el enlace del email de verificación que GitHub envió, luego repetir el proceso de cambio de visibilidad.

### Workflows existentes

Hay al menos un workflow documentado: `wf05_envio_propuesta` (envío de propuesta comercial). El número de versiones y workflows activos no está completamente documentado en el historial disponible.

---

## 2. CONTEXTO DE NEGOCIO

**Supraluz** es un comparador/broker de tarifas eléctricas. El CRM de operaciones gestiona:
- Leads de potenciales clientes (empresas o particulares buscando mejorar su tarifa eléctrica)
- Proceso de comparación de ofertas de proveedores
- Generación y envío de propuestas comerciales
- Seguimiento del ciclo de venta

Las automatizaciones n8n orquestan este proceso, probablemente integrando Notion como base de datos operacional y Telegram/WhatsApp para notificaciones.

---

## 3. PENDIENTES CRÍTICOS

### 3.1 Hacer el repositorio privado

**Estado:** Bloqueado por verificación de identidad de GitHub (sudo mode).

**Pasos:**
1. Revisar el email de la cuenta GitHub (`seospathios@gmail.com`)
2. Localizar el email de GitHub con el asunto "Confirm access" o similar
3. Hacer click en el enlace de verificación
4. Volver a `https://github.com/seospathios/supraluz-ops/settings`
5. Bajar a "Danger Zone" → "Change visibility" → "Change to private"
6. Confirmar con la contraseña o el código enviado

### 3.2 Documentar los workflows existentes

El historial de conversación no tiene suficiente detalle sobre qué workflows existen en supraluz-ops y qué hacen exactamente. Antes de continuar el desarrollo, conviene hacer un inventario:

- ¿Cuántos workflows hay? ¿Qué versión está activa en n8n?
- ¿Qué hace exactamente `wf05_envio_propuesta`?
- ¿Existen wf01–wf04? ¿Qué hacen?
- ¿Hay workflows que ya corren en producción?

**Acción:** Exportar todos los workflows activos de n8n y subirlos al repo.

### 3.3 Definir arquitectura completa

Lo que falta definir formalmente:

- **Schema Notion:** ¿Qué bases de datos existen? ¿Qué campos tiene cada una?
- **Integraciones activas:** ¿Qué servicios usa el sistema hoy? (Notion, email, CRM externo…)
- **Telegram:** ¿Está configurado? ¿Qué notificaciones se quieren?
- **WhatsApp:** ¿Se usa para comunicarse con clientes? ¿Con qué proveedor?

### 3.4 Configurar Telegram

Igual que en spathios-automations y ganaderia-ops, Telegram es el canal de notificaciones estándar en estos proyectos.

**Pasos:**
1. Abrir Telegram → @BotFather → `/newbot`
2. Nombrar: `Supraluz Ops Bot` / Username: `supraluz_ops_bot`
3. Copiar token
4. Obtener chat ID con @userinfobot o `getUpdates`
5. Añadir a Set Config del workflow principal

---

## 4. PRÓXIMOS PASOS RECOMENDADOS

El proyecto está en fase muy inicial. El orden recomendado para estructurarlo:

**Paso 1 — Hacer el repo privado** (ver §3.1)

**Paso 2 — Inventario de workflows activos**
- Exportar todos los workflows activos de n8n cloud → subir al repo en `/workflows/`
- Documentar qué hace cada uno en el README

**Paso 3 — Definir y documentar el schema de Notion**
- Listar todas las bases de datos existentes con sus IDs (completar `NOTION_IDS.md`)
- Documentar campos de cada DB

**Paso 4 — Crear estructura estándar de proyecto**
Similar a spathios-automations:
```
supraluz-ops/
├── README.md                    ← Arquitectura + changelog
├── workflows/
│   └── wf01_nombre_vXX.json    ← Workflows exportados con versionado
├── docs/
│   ├── NOTION_IDS.md
│   └── PLAN_CONTINUACION.md    ← Este documento
└── .claude/skills/              ← Skills de Claude cuando se creen
```

**Paso 5 — Configurar Telegram y probar notificaciones**

**Paso 6 — Definir roadmap de automatizaciones**
- ¿Qué procesos manuales quedan por automatizar?
- ¿Qué métricas se quieren en el report diario?
- ¿Se quiere integrar WhatsApp para comunicación con clientes?

---

## 5. VARIABLES DE CONFIGURACIÓN (PENDIENTES DE DOCUMENTAR)

No hay un Set Config estándar documentado para este proyecto. Una vez se haga el inventario de workflows, definir y registrar aquí:

| Variable | Valor | Estado |
|----------|-------|--------|
| `NOTION_TOKEN` | (token de integración Notion) | Por confirmar |
| `NOTION_*_DB_ID` | (IDs de las DBs — ver NOTION_IDS.md) | Por confirmar |
| `TELEGRAM_BOT_TOKEN` | (pendiente crear bot) | 🔵 Pendiente |
| `TELEGRAM_CHAT_ID` | (pendiente obtener) | 🔵 Pendiente |
| Otros... | (depende de workflows) | Por documentar |

---

## 6. NOTAS ADICIONALES

- Este proyecto sigue la misma filosofía técnica que `spathios-automations`: n8n como orquestador, Notion como base de datos, Claude como motor de IA cuando se necesita procesamiento de texto.
- Cuando se creen skills de Claude para este proyecto, seguir la convención de nombres: `supraluz-{función}/SKILL.md`.
- El README del repo debe incluir: descripción, arquitectura, changelog, variables de Set Config y deployment checklist (mismo formato que spathios-automations).
