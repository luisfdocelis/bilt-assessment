# Plan de Implementación Paso a Paso

Este documento especifica el **plan de implementación detallado**, con los cambios exactos de código, estrategia de commits, mitigación de riesgos y verificación técnica para solucionar el problema de idempotencia y la representación visual en el dashboard.

---

## 1. Visión General de la Implementación

El plan se divide en 3 pasos ejecutables y verificables:
- **Paso 1 (Backend - Java):** Refactorizar `ProcessedEventStore` y `RewardsEngine` para deduplicación atómica y concurrente.
- **Paso 2 (Frontend - JavaScript):** Mapear adecuadamente los resultados en `dashboard.js` de acuerdo a `result.outcome`.
- **Paso 3 (Validación Integral):** Ejecutar las suites de pruebas completas y verificar no-regresiones.

---

## 2. Paso 1: Implementación en el Backend (Java)

### A. Archivo: `src/main/java/com/rentrewards/challenge/service/ProcessedEventStore.java`

#### Estado Actual:
Almacena solo una variable de tipo `String` (`lastProcessedEventId`), sin soporte para retener múltiples eventos ni control de concurrencia.

#### Código a Implementar:
1. Utilizar un conjunto concurrente `Set<String>` creado con `ConcurrentHashMap.newKeySet()`.
2. Mantener los métodos públicos existentes para no romper compatibilidad (`isDuplicate` y `markProcessed`).
3. Introducir un método atómico `tryRecord(String eventId)` que aproveche la garantía de atomicidad de `Set.add(eventId)`. Retorna `true` si el evento fue añadido por primera vez (no existía), o `false` si ya había sido procesado.

```java
package com.rentrewards.challenge.service;

import java.util.Set;
import java.util.concurrent.ConcurrentHashMap;

public class ProcessedEventStore {

    private final Set<String> processedEventIds = ConcurrentHashMap.newKeySet();

    /**
     * @return true si este eventId ya ha sido procesado previamente.
     */
    public boolean isDuplicate(String eventId) {
        return processedEventIds.contains(eventId);
    }

    public void markProcessed(String eventId) {
        processedEventIds.add(eventId);
    }

    /**
     * Intenta registrar el evento de forma atómica.
     * @return true si es la primera vez que se procesa este eventId,
     *         false si ya existía en el almacén (es un duplicado).
     */
    public boolean tryRecord(String eventId) {
        return processedEventIds.add(eventId);
    }
}
```

---

### B. Archivo: `src/main/java/com/rentrewards/challenge/service/RewardsEngine.java`

#### Estado Actual:
Realiza una consulta no atómica `isDuplicate()` y posteriormente ejecuta `markProcessed()` luego de calcular y sumar puntos en `member`. Si 16 hilos entran concurrentemente con el mismo `eventId`, todos pasan `isDuplicate()`.

#### Código a Implementar:
Utilizar la reserva atómica del `eventId` al inicio del método `processPayment`:

```java
    public PointsResult processPayment(PaymentEvent event, MemberAccount member) {
        if (!processedEventStore.tryRecord(event.getEventId())) {
            return new PointsResult(member.getMemberId(), 0, ProcessingOutcome.DUPLICATE);
        }

        long basePoints = pointsCalculator.calculateBasePoints(event);
        long pointsWithBonus = pointsCalculator.applyStreakBonusIfEligible(
                basePoints, member.getCurrentStreakMonths());

        YearMonth month = YearMonth.from(event.getPaymentDate());
        long alreadyEarnedThisMonth = member.getPointsForMonth(month);
        long remainingCap = Math.max(0, MONTHLY_POINTS_CAP - alreadyEarnedThisMonth);
        long pointsToAward = Math.min(pointsWithBonus, remainingCap);

        member.addPointsForMonth(month, pointsToAward);

        ProcessingOutcome outcome = pointsToAward == 0
                ? ProcessingOutcome.CAPPED
                : ProcessingOutcome.AWARDED;
        return new PointsResult(member.getMemberId(), pointsToAward, outcome);
    }
```

---

## 3. Paso 2: Implementación en el Frontend (JavaScript)

### Archivo: `web/dashboard.js`

#### Estado Actual:
La función `buildViewModel(result, member)` retorna un título y tono fijo de éxito sin importar `result.outcome`.

#### Código a Implementar:
Configurar las respuestas adecuadas para cada uno de los tres posibles estados:
1. `AWARDED`: Tono `success`, anunciando los puntos acreditados.
2. `DUPLICATE`: Tono `neutral`, indicando `"Duplicate event skipped"`.
3. `CAPPED`: Tono `warning`, indicando `"Monthly cap reached"`.

```javascript
export function buildViewModel(result, member) {
  const progressPercent = Math.min(
    100,
    (member.pointsThisMonth / member.monthlyCap) * 100,
  );

  let title;
  let description;
  let tone;

  switch (result.outcome) {
    case "DUPLICATE":
      title = "Duplicate event skipped";
      description = "This payment event was already processed.";
      tone = "neutral";
      break;
    case "CAPPED":
      title = "Monthly cap reached";
      description = "You have reached the monthly points cap.";
      tone = "warning";
      break;
    case "AWARDED":
    default:
      title = `${numberFormatter.format(result.pointsAwarded)} points credited`;
      description = "Your rent payment was processed successfully.";
      tone = "success";
      break;
  }

  return {
    title,
    description,
    tone,
    progressPercent,
  };
}
```

---

## 4. Paso 3: Plan de Pruebas y Criterios de Aceptación

### Verificación Automatizada:
1. **Ejecutar tests de Java:**
   ```bash
   mvn test
   ```
   *Criterio de éxito:* 8 tests ejecutados, 0 fallos, 0 errores.
2. **Ejecutar tests de Web/Node:**
   ```bash
   npm run test:web
   ```
   *Criterio de éxito:* 3 tests ejecutados, 3 tests pasados.

---

## 5. Matriz de Riesgos y Mitigación

| Riesgo | Impacto | Mitigación |
| :--- | :--- | :--- |
| **Condición de carrera entre múltiples workers** | Duplicación de puntos | Uso de `ConcurrentHashMap.newKeySet().add()` que garantiza atomicidad a nivel de CPU / memoria sin cuellos de botella de bloqueos pesados. |
| **Ruptura de firmas existentes en `ProcessedEventStore`** | Tests existentes podrían romperse si usan `isDuplicate` o `markProcessed` directamente | Se preservan ambos métodos públicos tal como estaban definidos, respaldados por la nueva estructura concurrente. |
| **Eventos con cálculo de 0 puntos por tope mensual** | Clasificación incorrecta del estado | Se conserva la regla de negocio existente: si `pointsToAward == 0` tras aplicar el límite, el resultado es `CAPPED`. |

