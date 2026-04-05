# WF-03: Alarmas de Renovación Próxima

## Objetivo

WF-03 ejecuta diariamente a las 8:00 AM para identificar todos los Puntos de Suministro cuyo contrato vence en los próximos 30 días. Agrupa por cliente y notifica a Domin con lista detallada para que genere comparativas proactivas.

---

## Trigger

**Schedule Trigger**
- **Frequency:** Diariamente
- **Time:** 08:00 AM (timezone local)
- **No trigger de WF-00:** Este workflow es independiente y
