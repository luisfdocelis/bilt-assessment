# Plan Integral de Pruebas (Test Plan)

Este documento describe la estrategia, casos de prueba, herramientas y criterios de aceptación para validar exhaustivamente las correcciones del backend (Java) y frontend (JavaScript) de **RentRewards**.

---

## 1. Alcance y Objetivos de las Pruebas

El objetivo de este plan de pruebas es garantizar:
1. **Idempotencia Absoluta:** Que ningún evento de pago (`PaymentEvent`) con un mismo `eventId` otorgue puntos más de una vez, bajo cualquier condición de red (reintentos, desorden o alta concurrencia).
2. **Preservación de Reglas de Negocio:** Que los cálculos de puntos base, bonos de cuentas vinculadas, bonificaciones por racha (*streak*) y topes mensuales sigan funcionando correctamente.
3. **Exactitud en la Presentación de UI:** Que el panel del usuario en la web refleje fidedignamente los tres posibles resultados de procesamiento (`AWARDED`, `DUPLICATE`, `CAPPED`) con el texto y estilo visual correspondiente.

---

## 2. Niveles de Pruebas

```
+-----------------------------------------------------------+
|                   Pruebas de UI / Web                     |
|           (Node.js Test Runner: dashboard.test.js)        |
+-----------------------------------------------------------+
                             |
+-----------------------------------------------------------+
|             Pruebas de Integración y Concurrencia         |
|              (JUnit 5: RewardsEngineTest.java)            |
+-----------------------------------------------------------+
                             |
+-----------------------------------------------------------+
|             Pruebas Unitarias de Componentes              |
|        (ProcessedEventStoreTest, PointsCalculatorTest)    |
+-----------------------------------------------------------+
```

---

## 3. Matriz de Casos de Prueba - Backend (Java)

| ID | Nombre de Caso de Prueba | Tipo | Entrada / Escenario | Comportamiento Esperado | Criterio de Éxito |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **TC-BE-01** | `awardsOnePointPerDollarByDefault` | Regla Negocio | Pago de $1,500 sin bonos. | Acredita 1,500 puntos. | `outcome == AWARDED`, `points == 1500` |
| **TC-BE-02** | `appliesLinkedAccountMultiplier` | Regla Negocio | Pago de $1,500 con cuenta vinculada. | Aplica 2x: Acredita 3,000 puntos. | `outcome == AWARDED`, `points == 3000` |
| **TC-BE-03** | `appliesStreakBonusWhenEligible` | Regla Negocio | Pago de $2,000, vinculada, 6 meses racha. | 2,000 * 2 = 4,000; +10% = 4,400 puntos. | `outcome == AWARDED`, `points == 4400` |
| **TC-BE-04** | `enforcesMonthlyPointsCap` | Regla Negocio | Pago de $150,000 en un mes. | Limita a 100,000 puntos. | `outcome == AWARDED`, `points == 100000` |
| **TC-BE-05** | `reportsCappedWhenALaterEventCannotAwardMorePoints` | Regla Negocio | Evento tras alcanzar el límite mensual. | 0 puntos otorgados, estado CAPPED. | `outcome == CAPPED`, `points == 0` |
| **TC-BE-06** | `doesNotDoubleAwardPointsWhenSameWebhookEventIsResent` | Idempotencia | Mismo `eventId` recibido consecutivamente. | El primer evento acredita puntos; el reintento se marca como duplicado. | Reintento: `outcome == DUPLICATE`, `points == 0` |
| **TC-BE-07** | `doesNotDoubleAwardPointsWhenAnOlderEventIsResentOutOfOrder` | Idempotencia / Desorden | Llega Evento A, luego Evento B, luego reintento de Evento A. | Evento A redelivered no otorga puntos dobles. | Reintento A: `outcome == DUPLICATE`, balance total exacto (3,000) |
| **TC-BE-08** | `awardsPointsAtMostOnceWhenTheSameEventArrivesConcurrently` | Concurrencia | 16 hilos disparan simultáneamente el mismo evento (repetido 8 veces). | Exactamente 1 hilo acredita puntos; 15 retornan DUPLICATE. | Total `AWARDED` = 1, `member.getPoints()` = 1,500 |
| **TC-BE-09** | *(Nuevo test propuesto)* `ProcessedEventStoreTest` | Unitario | Inserción directa de duplicados y consultas concurrentes en el Store. | Métodos `tryRecord`, `isDuplicate` y `markProcessed` consistentes. | Respuestas booleanas correctas sin excepción |

---

## 4. Matriz de Casos de Prueba - Frontend (JavaScript)

Las pruebas se ejecutan utilizando el runner nativo de Node.js (`node --test`).

| ID | Nombre de Caso de Prueba | Entrada (`result`, `member`) | Comportamiento Esperado | Criterio de Éxito |
| :--- | :--- | :--- | :--- | :--- |
| **TC-FE-01** | `shows awarded points as a successful payment` | `{ pointsAwarded: 1500, outcome: "AWARDED" }` | `title: "1,500 points credited"`, `tone: "success"`, `progressPercent: 1.5` | Pasa la aserción estricta (`assert.equal`) |
| **TC-FE-02** | `shows a duplicate event as skipped rather than credited` | `{ pointsAwarded: 0, outcome: "DUPLICATE" }` | `title: "Duplicate event skipped"`, `tone: "neutral"` | `title` y `tone` coinciden exactamente |
| **TC-FE-03** | `shows when the member has reached the monthly cap` | `{ pointsAwarded: 0, outcome: "CAPPED" }`, `pointsThisMonth: 100000` | `title: "Monthly cap reached"`, `tone: "warning"`, `progressPercent: 100` | `title`, `tone` y `progressPercent` coinciden |

---

## 5. Estrategia de Ejecución de Pruebas

### Paso 1: Pruebas Unitarias y de Integración Backend
Ejecutar Maven con Surefire en terminal:
```bash
mvn test
```
*Monitoreo:* El test `@RepeatedTest(8) awardsPointsAtMostOnceWhenTheSameEventArrivesConcurrently` ejecutará 8 rondas de estrés con 16 hilos en competencia utilizando `CountDownLatch` para sincronizar el arranque simultáneo.

### Paso 2: Pruebas Unitarias Frontend
Ejecutar Node test runner:
```bash
npm run test:web
```

### Paso 3: Validación Visual / Exploratoria
Servir la aplicación web estática:
```bash
python3 -m http.server 8000
```
Abrir `http://localhost:8000/web/` y verificar que los elementos visuales (borde, fondo y color de texto según las clases de Tailwind de `toneClasses`) correspondan al tono indicado.

---

## 6. Criterios de Aceptación (Definición de Terminado - DoD)

1. **Backend:**
   * Todos los 8 tests predefinidos en `RewardsEngineTest` finalizan en estado `PASSED`.
   * Cero fallos (`Failures: 0`) y cero errores (`Errors: 0`).
2. **Frontend:**
   * Los 3 tests en `dashboard.test.js` finalizan en estado `pass`.
3. **No-Regresión:**
   * Ningún comportamiento preexistente de cálculo de puntos se altera.

