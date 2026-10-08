# Análisis del Incidente y Diagnóstico Técnico

Este documento detalla el análisis del incidente de duplicación de puntos y el mal comportamiento del panel de control (dashboard) en la plataforma **RentRewards**.

---

## 1. Contexto del Negocio y Arquitectura

RentRewards premia a sus miembros con puntos al realizar pagos de renta e hipoteca. Los pagos son procesados por un proveedor externo que envía notificaciones mediante **webhooks** hacia el backend (`RewardsEngine`).

### Garantía de Entrega: *At-Least-Once Delivery*
El procesador de pagos opera bajo la premisa de **entrega al menos una vez**. En un entorno de red real, esto implica que un mismo evento de pago (`PaymentEvent` con un identificador único `eventId`) puede llegar:
1. **Múltiples veces**, producto de reintentos automáticos tras caídas de red o demoras en la respuesta HTTP.
2. **Fuera de orden cronológico**, intercalándose con eventos posteriores.
3. **De manera simultánea/concurrente**, procesándose en distintos hilos o workers paralelos.

---

## 2. Problemas Detectados y Causa Raíz

---

### Problema A: Falla de Idempotencia y Duplicación de Puntos (Backend - Java)

#### Descripción del Reporte
Se reportó que algunos usuarios reciben puntos duplicados tras reintentos del webhook, especialmente cuando los eventos llegan fuera de orden o son consumidos concurrentemente por distintos workers.

#### Diagnóstico y Causas Raíz

1. **Pérdida de Historial (Entregas fuera de orden):**
   * **Archivo:** `src/main/java/com/rentrewards/challenge/service/ProcessedEventStore.java`
   * **Causa:** La clase implementa el almacenamiento de eventos utilizando una sola variable:
     ```java
     private String lastProcessedEventId;

     public boolean isDuplicate(String eventId) {
         return eventId.equals(lastProcessedEventId);
     }
     ```
   * **Impacto:** Solo recuerda el **último** evento procesado. Si llega el evento $A$, luego el evento $B$, y posteriormente un reintento del evento $A$, la comparación `eventId.equals(lastProcessedEventId)` comparará $A$ contra $B$, retornando `false`. Como consecuencia, $A$ se procesa nuevamente y se acreditan puntos dobles al usuario.

2. **Condición de Carrera / Falta de Atomicidad (Entregas concurrentes):**
   * **Archivos:** `RewardsEngine.java` y `ProcessedEventStore.java`
   * **Causa:** En `RewardsEngine.processPayment(...)`:
     ```java
     if (processedEventStore.isDuplicate(event.getEventId())) {
         return new PointsResult(..., DUPLICATE);
     }
     // ... cálculo y acreditación de puntos ...
     processedEventStore.markProcessed(event.getEventId());
     ```
   * **Impacto:** Patrón anti-patrón clásico **Check-Then-Act (verificar y luego actuar)** no sincronizado:
     * Si dos o más hilos reciben el mismo `eventId` al mismo tiempo, ambos comprueban `isDuplicate(...)` antes de que cualquiera ejecute `markProcessed(...)`.
     * Ambos hilos concluyen erróneamente que el evento es nuevo y acreditan puntos en la cuenta del miembro.
     * La clase `ProcessedEventStore` no utiliza estructuras seguras para hilos (*thread-safe*) ni mecanismos de exclusión mutua (`synchronized`, `Lock`, o `ConcurrentHashMap`).

---

### Problema B: Inconsistencia en la UI del Dashboard (Frontend - JavaScript)

#### Descripción del Reporte
El panel de recompensas siempre anuncia que los puntos fueron acreditados con éxito, incluso si el evento fue descartado como duplicado o si el miembro alcanzó el límite mensual de puntos.

#### Diagnóstico y Causa Raíz

* **Archivo:** `web/dashboard.js`
* **Causa:** En la función `buildViewModel(result, member)`:
  ```javascript
  export function buildViewModel(result, member) {
    return {
      title: `${numberFormatter.format(result.pointsAwarded)} points credited`,
      description: "Your rent payment was processed successfully.",
      tone: "success",
      progressPercent,
    };
  }
  ```
* **Impacto:** 
  * Se ignora por completo la propiedad `result.outcome` (`AWARDED`, `DUPLICATE`, o `CAPPED`).
  * Cuando un evento es duplicado (`DUPLICATE`), la UI presenta el confuso mensaje `"0 points credited"` con estilo verde de éxito (`tone: "success"`). Debería indicar `"Duplicate event skipped"` con tono neutral (`tone: "neutral"`).
  * Cuando se alcanza el tope mensual de 100,000 puntos (`CAPPED`), la UI también muestra `"0 points credited"` como éxito, en lugar de alertar `"Monthly cap reached"` con tono de advertencia (`tone: "warning"`).

---

## 3. Plan de Mitigación y Solución Propuesta

### Backend:
1. **Estructura Concurrente:** Sustituir la variable única `lastProcessedEventId` en `ProcessedEventStore` por una colección thread-safe que retenga todos los identificadores procesados (ej. `ConcurrentHashMap.newKeySet()`).
2. **Operación Atómica:** Convertir la comprobación y el registro en una sola operación atómica (ej. un método `recordIfAbsent(String eventId)` que devuelva `boolean`, aprovechando `Set.add()` thread-safe o un bloque de sincronización), evitando la condición de carrera entre el check y el mark.

### Frontend:
1. **Control por Estados:** Actualizar `buildViewModel` en `web/dashboard.js` evaluando `result.outcome`:
   * **`AWARDED`:** Título `"${pointsAwarded} points credited"`, tono `success`.
   * **`DUPLICATE`:** Título `"Duplicate event skipped"`, tono `neutral`.
   * **`CAPPED`:** Título `"Monthly cap reached"`, tono `warning`.

