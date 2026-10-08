# Test Plan and Verification Strategy

This document details the test strategy, coverage matrix, and test results for verifying the backend (Java) and frontend (JavaScript) fixes in **RentRewards**.

---

## 1. Scope and Test Levels

The verification strategy encompasses three levels:
1. **Model & Unit Tests (Java / JUnit 5):** Validates entities (`MemberAccount`, `PaymentEvent`, `PointsResult`, `ProcessingOutcome`), deduplication store (`ProcessedEventStore`), and calculation rules (`PointsCalculator`).
2. **Integration & Concurrency Tests (Java / JUnit 5):** Validates out-of-order deliveries and multithreaded stress testing in `RewardsEngineTest`.
3. **Frontend Tests (Node.js Test Runner):** Validates viewmodel generation in `web/dashboard.test.js`.

---

## 2. Test Coverage Matrix - Backend (Java)

| Test Class | Test Case | Scenario / Purpose | Expected Result |
| :--- | :--- | :--- | :--- |
| **`RewardsEngineTest`** | `awardsOnePointPerDollarByDefault` | Standard $1,500 payment | 1,500 points awarded (`AWARDED`) |
| **`RewardsEngineTest`** | `appliesLinkedAccountMultiplier` | Linked account payment | 2x multiplier (3,000 points, `AWARDED`) |
| **`RewardsEngineTest`** | `appliesStreakBonusWhenEligible` | 6+ months streak | 2x + 10% bonus (4,400 points, `AWARDED`) |
| **`RewardsEngineTest`** | `enforcesMonthlyPointsCap` | Payment exceeds monthly limit | Cap enforced at 100,000 points |
| **`RewardsEngineTest`** | `reportsCappedWhenALaterEventCannotAwardMorePoints` | Subsequent event after reaching cap | 0 points awarded (`CAPPED`) |
| **`RewardsEngineTest`** | `doesNotDoubleAwardPointsWhenSameWebhookEventIsResent` | Immediate retry of same event | Second attempt receives `DUPLICATE` |
| **`RewardsEngineTest`** | `doesNotDoubleAwardPointsWhenAnOlderEventIsResentOutOfOrder` | Interleaved events (A -> B -> A) | Redelivered A is recognized as `DUPLICATE` |
| **`RewardsEngineTest`** | `awardsPointsAtMostOnceWhenTheSameEventArrivesConcurrently` | 16 concurrent workers competing simultaneously | Exactly 1 worker awards points, 15 receive `DUPLICATE` |
| **`ProcessedEventStoreTest`** | `isDuplicateReturnsFalseForUnseenEvent` | Query unseen event | Returns `false` |
| **`ProcessedEventStoreTest`** | `markProcessedMakesEventADuplicate` | Mark and query event | Returns `true` |
| **`ProcessedEventStoreTest`** | `remembersMultipleDistinctEventsOutOfOrder` | Interleaved event IDs | All previously stored IDs are recognized |
| **`ProcessedEventStoreTest`** | `tryRecordReturnsTrueForFirstAttemptAndFalseSubsequently` | Atomic insert check | Returns `true` on first call, `false` on retry |
| **`ProcessedEventStoreTest`** | `tryRecordIsThreadSafeUnderHighConcurrency` | 32 parallel threads invoking `tryRecord` | Exactly 1 returns `true`, 31 return `false` |
| **`PointsCalculatorTest`** | `calculateBasePointsWithoutLinkedAccount` | Unlinked calculation | Correct truncated base points |
| **`PointsCalculatorTest`** | `calculateBasePointsWithLinkedAccount` | Linked 2x calculation | Correct truncated multiplied points |
| **`PointsCalculatorTest`** | `applyStreakBonusBelowThresholdDoesNotApplyBonus` | 5 months streak | Base points unchanged |
| **`PointsCalculatorTest`** | `applyStreakBonusAtThresholdAppliesTenPercent` | 6 months streak | 10% bonus applied |
| **`PointsCalculatorTest`** | `applyStreakBonusAboveThresholdAppliesTenPercent` | 12 months streak | 10% bonus applied |
| **`PointsCalculatorTest`** | `applyStreakBonusRoundsDownFractionalBonus` | Fractional bonus rounding | Rounds down fractional point amounts |
| **`ModelCoverageTest`** | `testPaymentEventModel` | Model construction & null checks | Validates immutability and preconditions |
| **`ModelCoverageTest`** | `testMemberAccountModel` | Monthly points aggregation | Validates thread-safe monthly accrual |
| **`ModelCoverageTest`** | `testPointsResultModel` | Result accessors and toString | Validates getters and helper methods |
| **`ModelCoverageTest`** | `testProcessingOutcomeEnum` | Enum values check | All 3 enum values present and valid |

---

## 3. Test Coverage Matrix - Frontend (JavaScript)

| Test File | Test Case | Input | Expected Outcome |
| :--- | :--- | :--- | :--- |
| **`dashboard.test.js`** | `shows awarded points as a successful payment` | `{ pointsAwarded: 1500, outcome: "AWARDED" }` | `title: "1,500 points credited"`, `tone: "success"` |
| **`dashboard.test.js`** | `shows a duplicate event as skipped rather than credited` | `{ pointsAwarded: 0, outcome: "DUPLICATE" }` | `title: "Duplicate event skipped"`, `tone: "neutral"` |
| **`dashboard.test.js`** | `shows when the member has reached the monthly cap` | `{ pointsAwarded: 0, outcome: "CAPPED" }` | `title: "Monthly cap reached"`, `tone: "warning"` |
| **`dashboard.test.js`** | `caps progress percent at 100 even if points exceed monthly cap` | `{ pointsThisMonth: 120000, monthlyCap: 100000 }` | `progressPercent: 100` |

---

## 4. Test Execution Results

* **Backend (`mvn test`):**
  * Total tests: **30**
  * Failures: **0**
  * Errors: **0**
  * Skipped: **0**
  * Result: **BUILD SUCCESS**
* **Frontend (`npm run test:web`):**
  * Total tests: **4**
  * Passing: **4**
  * Failing: **0**
  * Result: **ALL PASSED**
