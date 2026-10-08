# Plan de Solución: Corrección de Idempotencia y Estado del Dashboard

Este plan de acción define los pasos técnicos para corregir de manera robusta, limpia y comprobable los dos problemas identificados en el proyecto RentRewards.

---

## 1. Objetivos

1. **Garantizar Idempotencia en el Backend (Java):**
   * Asegurar que un mismo `eventId` otorgue puntos como máximo una sola vez.
   * Manejar entregas fuera de orden cronológico.
   * Manejar entregas simultáneas/concurrentes en ambientes multihilo sin condiciones de carrera.
   * Lograr que el 100% de la suite `mvn test` pase sin errores.
2. **Reflejar el Resultado Real en el Dashboard (JavaScript):**
   * Mapear correctamente los resultados `AWARDED`, `DUPLICATE` y `CAPPED` al estado del ViewModel.
   * Lograr que el 100% de la suite `npm run test:web` pase sin errores.
3. **Preservar las Reglas de Negocio:**
   * No alterar las fórmulas de puntos base, bonos de cuentas vinculadas, bonos por racha ni el límite mensual de 100,000 puntos.

---

## 2. Fase 1: Solución en el Backend (Java)

### Diagnóstico del Problema Técnico
Actualmente `ProcessedEventStore` usa una variable simple `lastProcessedEventId`, olvidando eventos previos cuando se intercalan, y no ofrece métodos atómicos para registrar eventos frente a múltiples hilos.

### Estrategia de Solución
* **Almacenamiento Concurrente:** 
  Utilizar un `Set<String>` concurrente respaldado por `ConcurrentHashMap.newKeySet()` en `ProcessedEventStore`.
* **Atomicidad con *Check-and-Add*:**
  El método `Set.add(element)` en un set concurrente es atómico y retorna `true` si el elemento no existía y fue agregado con éxito, o `false` si ya existía.
  Añadiremos un método atómico `tryRecord(String eventId)` (o bien sincronización / implementación en `markProcessed` e `isDuplicate`) para que la verificación y el registro ocurran en una sola operación atómica.
* **Refactorización en `ProcessedEventStore.java`:**
  - Mantener retrocompatibilidad con los métodos públicos existentes `isDuplicate(String eventId)` y `markProcessed(String eventId)`.
  - Introducir `tryRecord(String eventId)` o hacer que `isDuplicate` consulte el `Set` y `markProcessed` lo registre en el `Set`.
  - Para evitar la condición de carrera *Check-Then-Act*, `RewardsEngine` o `ProcessedEventStore` debe reservar/registrar el `eventId` de manera atómica antes de acreditar los puntos al usuario.
* **Ajuste en `RewardsEngine.java`:**
  - Si un hilo intenta registrar un `eventId` que ya fue tomado por otro hilo o procesado previamente, retorna inmediatamente `new PointsResult(member.getMemberId(), 0, ProcessingOutcome.DUPLICATE)`.
  - De esta manera, únicamente un hilo tiene éxito en la reserva y procede a calcular y agregar los puntos a `MemberAccount`.

---

## 3. Fase 2: Solución en el Frontend (JavaScript)

### Diagnóstico del Problema Técnico
La función `buildViewModel(result, member)` en `web/dashboard.js` devuelve de forma fija:
- `title: "{pointsAwarded} points credited"`
- `tone: "success"`
Ignorando por completo `result.outcome`.

### Estrategia de Solución
Actualizar `buildViewModel(result, member)` para evaluar `result.outcome`:

1. **Caso `AWARDED`:**
   * `title`: `"${numberFormatter.format(result.pointsAwarded)} points credited"`
   * `description`: `"Your rent payment was processed successfully."`
   * `tone`: `"success"`
2. **Caso `DUPLICATE`:**
   * `title`: `"Duplicate event skipped"`
   * `description`: `"This payment event was already processed."`
   * `tone`: `"neutral"`
3. **Caso `CAPPED`:**
   * `title`: `"Monthly cap reached"`
   * `description`: `"You have reached the maximum points allowed for this calendar month."`
   * `tone`: `"warning"`

---

## 4. Fase 3: Pruebas y Validación

1. **Ejecución de Pruebas Backend:**
   * Comando: `mvn test`
   * Validaciones clave:
     * `doesNotDoubleAwardPointsWhenAnOlderEventIsResentOutOfOrder`: Verifica eventos intercalados.
     * `awardsPointsAtMostOnceWhenTheSameEventArrivesConcurrently`: Verifica que entre 16 workers concurrentes (repetido 8 veces), exactamente 1 consiga acreditar puntos.
2. **Ejecución de Pruebas Frontend:**
   * Comando: `npm run test:web`
   * Validaciones clave:
     * `shows awarded points as a successful payment`
     * `shows a duplicate event as skipped rather than credited`
     * `shows when the member has reached the monthly cap`
3. **Revisión de no-regresión:**
   * Confirmar que el cálculo de puntos y reglas preexistentes sigan funcionando al 100%.

---

## 5. Resumen de Archivos a Modificar

| Archivo | Cambio Propuesto |
| :--- | :--- |
| `src/main/java/com/rentrewards/challenge/service/ProcessedEventStore.java` | Implementar `Set<String>` concurrente y método atómico de reserva/registro. |
| `src/main/java/com/rentrewards/challenge/service/RewardsEngine.java` | Validar y reservar atómicamente el `eventId` antes de otorgar puntos. |
| `web/dashboard.js` | Condicionar título, descripción y tono visual según `result.outcome`. |

