# ISL (Instruction Sanitization Layer)

> **Instruction Sanitization Layer** — Second layer of the AI-PIP protocol. Sanitizes content from CSL, detects prompt-injection patterns, and emits signals (ISLSignal) for AAL and the SDK.

---

## 1. Overview

The **Instruction Sanitization Layer (ISL)** receives segmented content from CSL and applies trust-level–based sanitization. It preserves the original content for audit, optionally runs prompt-injection detection, and exposes a **signal** (ISLSignal) so other layers (e.g. AAL) can make decisions without depending on ISL internals.

### 1.1 Principles

- **Differentiated sanitization**: Level depends on segment trust (TC → minimal, STC → moderate, UC → aggressive).
- **Prompt-injection detection**: Identifies malicious patterns and builds risk scores and detections (PiDetection, PiDetectionResult).
- **Original preserved**: Each segment keeps `originalContent` and `sanitizedContent` for audit.
- **Signal contract**: External consumers receive **ISLSignal**, not **ISLResult**. ISLResult is internal; ISLSignal is the semantic contract for AAL and the SDK.

### 1.2 What ISL does not do

- **PolicyRule**, **AnomalyScore**, **AnomalyAction**, **RemovedInstruction**: These belong to **AAL**, not ISL. ISL does not decide ALLOW/WARN/BLOCK or build removal plans.
- **Execution of actions**: Blocking, logging, or removing instructions is done by the SDK or AAL; ISL only produces results and signals.

---

## 2. Processing flow

```
CSLResult (from CSL)
    ↓
sanitize(cslResult)
    ↓
ISLResult (internal: segments, lineage, metadata)
    ↓
emitSignal(islResult)  [optional, for downstream layers]
    ↓
ISLSignal (external contract: riskScore, piDetection, hasThreats, timestamp)
    → consumed by AAL / SDK
```

- **sanitize**: Single public entry for sanitization; returns **ISLResult**.
- **emitSignal**: Converts **ISLResult** → **ISLSignal** so AAL (and others) consume only the signal.

---

## 3. Main API

### 3.1 Sanitization

| Function | Description |
|----------|-------------|
| **`sanitize(cslResult: CSLResult): ISLResult`** | Sanitizes CSL segments by trust level (TC → minimal, STC → moderate, UC → aggressive). Returns internal result with segments, lineage, and metadata. |

### 3.2 Signals (external contract)

| Function | Description |
|----------|-------------|
| **`emitSignal(islResult: ISLResult, timestamp?: number): ISLSignal`** | Builds the external signal from an internal ISLResult. Aggregates detections from all segments and computes risk score. |
| **`createISLSignal(riskScore, piDetection, timestamp?): ISLSignal`** | Builds an ISLSignal from risk score, PiDetectionResult, and optional timestamp. |
| **`isHighRiskSignal(signal, threshold?)`** | `true` if `signal.riskScore >= threshold` (default 0.7). |
| **`isMediumRiskSignal(signal, lowThreshold?, highThreshold?)`** | `true` if risk score is in [low, high) (defaults 0.3, 0.7). |
| **`isLowRiskSignal(signal, threshold?)`** | `true` if `signal.riskScore < threshold` (default 0.3). |

### 3.3 Internal process and lineage

| Function | Description |
|----------|-------------|
| **`buildISLResult(segments, lineage, processingTimeMs?): ISLResult`** | Builds ISLResult from sanitized segments and lineage. Used internally by the pipeline. |
| **`buildISLLineage(previousLineage, timestamp?): readonly LineageEntry[]`** | Appends an ISL lineage entry; used internally when building segment/result lineage. |

---

## 4. Types and value objects

### 4.1 RiskScore

- **Type**: `number` in range **0.0–1.0** (semantic alias).
- **Meaning**: 0 = no risk, 1 = maximum risk. Decisions (ALLOW/WARN/BLOCK) are made by **AAL**, not ISL.

**Constants**

- `MIN_RISK_SCORE = 0`, `MAX_RISK_SCORE = 1`

**Functions**

- **`createRiskScore(value: number): RiskScore`** — Validates and returns a RiskScore; throws if out of range.
- **`normalizeRiskScore(value: number): RiskScore`** — Clamps to [0, 1].
- **`isHighRiskScore(score, threshold?)`** — `score >= threshold` (default 0.7).
- **`isMediumRiskScore(score, lowThreshold?, highThreshold?)`** — Score in [low, high) (defaults 0.3, 0.7).
- **`isLowRiskScore(score, threshold?)`** — `score < threshold` (default 0.3).

---

### 4.2 ISLSignal (external contract)

Consumed by AAL and the SDK. Does not expose ISL internals (segments, full lineage, etc.).

```ts
interface ISLSignal {
  readonly riskScore: RiskScore
  readonly piDetection: PiDetectionResult
  readonly hasThreats: boolean
  readonly timestamp: number
}
```

- **riskScore**: Aggregated risk (0–1).
- **piDetection**: All detections and aggregated score.
- **hasThreats**: Shortcut for “any detection” (`piDetection.detected`).
- **timestamp**: Processing time for audit.

---

### 4.3 PiDetection (single detection)

```ts
type PiDetection = {
  readonly pattern_type: string
  readonly matched_pattern: string
  readonly position: Position
  readonly confidence: RiskScore
}
```

- **pattern_type**: Category of the pattern (e.g. jailbreak, override).
- **matched_pattern**: Exact matched text.
- **position**: `{ start, end }` (inclusive start, exclusive end).
- **confidence**: Detection confidence in [0, 1].

**Creation**: `createPiDetection(pattern_type, matched_pattern, position, confidence): PiDetection`

**Helpers**

- `getDetectionLength(detection): number` — `position.end - position.start`
- `isHighConfidence(detection)` — confidence >= 0.7
- `isMediumConfidence(detection)` — confidence in [0.3, 0.7)
- `isLowConfidence(detection)` — confidence < 0.3

**Position** (shared type): `{ start: number, end: number }`, start inclusive, end exclusive.

---

### 4.4 PiDetectionResult (aggregated detections)

```ts
type PiDetectionResult = {
  readonly detections: readonly PiDetection[]
  readonly score: RiskScore
  readonly patterns: readonly string[]
  readonly detected: boolean
}
```

- **detections**: All individual detections.
- **score**: Aggregated risk (complementary probability over detections).
- **patterns**: List of `pattern_type` values from detections.
- **detected**: `detections.length > 0`.

**Creation**: **`createPiDetectionResult(detections: readonly PiDetection[]): PiDetectionResult`**

- Only argument is `detections`. `score` and `patterns` are computed inside; no `overallConfidence` or separate `patterns` argument.

**Helpers**

- `hasDetections(result)` — same as `result.detected`
- `getDetectionCount(result)` — `result.detections.length`
- `getDetectionsByType(result, pattern_type)` — detections with that `pattern_type`
- `getHighestConfidenceDetection(result)` — detection with highest `confidence`, or `undefined`

---

### 4.5 Pattern (detection pattern)

```ts
type Pattern = {
  readonly pattern_type: string
  readonly regex: RegExp
  readonly base_confidence: RiskScore
  readonly description: string
}
```

**Constants**

- `MAX_CONTENT_LENGTH = 10_000_000`, `MAX_PATTERN_LENGTH = 10_000`, `MAX_MATCHES = 10_000`

**Creation**: `createPattern(pattern_type, regex, base_confidence, description?): Pattern` — `description` is optional.

**Helpers**

- `matchesPattern(pattern, content): boolean` — whether `pattern.regex` matches `content`
- `findMatch(pattern, content): { matched: string; position: { start, end } } | null` — first match; property is **`matched`**, not `match`.

---

### 4.6 ISLSegment

```ts
interface ISLSegment {
  readonly id: string
  readonly originalContent: string
  readonly sanitizedContent: string
  readonly trust: TrustLevel
  readonly lineage: LineageEntry[]
  readonly piDetection?: PiDetectionResult
  readonly sanitizationLevel: 'minimal' | 'moderate' | 'aggressive'
}
```

- **originalContent**: Preserved from CSL for audit.
- **sanitizedContent**: Output after sanitization.
- **trust**: Trust level from CSL (TC, STC, UC).
- **lineage**: Lineage including ISL step.
- **piDetection**: Optional prompt-injection result for this segment.
- **sanitizationLevel**: Level applied (minimal / moderate / aggressive).

ISL does **not** add `anomalyScore` or `instructionsRemoved` to segments; those concepts belong to AAL.

---

### 4.7 ISLResult (internal result)

```ts
interface ISLResult {
  readonly segments: readonly ISLSegment[]
  readonly lineage: readonly LineageEntry[]
  readonly metadata: {
    readonly totalSegments: number
    readonly sanitizedSegments: number
    readonly processingTimeMs?: number
  }
}
```

- **metadata** contains only `totalSegments`, `sanitizedSegments`, and optional `processingTimeMs`. No `blockedSegments` or `instructionsRemoved` (those are AAL/SDK concerns).

---

## 5. Exceptions

- **`SanitizationError`**: Thrown when sanitization fails (invalid content or validation error). Optional `cause` for chaining.

---

## 6. Usage examples

### 6.1 Basic sanitization

```typescript
import { segment } from '@ai-pip/core/csl'
import { sanitize } from '@ai-pip/core/isl'
import type { CSLResult, ISLResult } from '@ai-pip/core'

const cslResult: CSLResult = segment({
  content: 'User input with potential injection',
  source: 'UI',
  metadata: {}
})

const islResult: ISLResult = sanitize(cslResult)

// Each segment has:
// - id, originalContent, sanitizedContent, trust, lineage
// - piDetection? (if detection ran)
// - sanitizationLevel: 'minimal' | 'moderate' | 'aggressive'
```

### 6.2 Emit signal for AAL

```typescript
import { sanitize, emitSignal } from '@ai-pip/core/isl'
import type { ISLSignal } from '@ai-pip/core/isl'

const islResult = sanitize(cslResult)
const islSignal: ISLSignal = emitSignal(islResult)

// AAL consumes islSignal (not islResult):
// - islSignal.riskScore, islSignal.piDetection, islSignal.hasThreats, islSignal.timestamp
```

### 6.3 PiDetection and PiDetectionResult

```typescript
import {
  createPiDetection,
  createPiDetectionResult,
  hasDetections,
  getDetectionCount,
  getHighestConfidenceDetection,
  isHighConfidence
} from '@ai-pip/core/isl'
import type { PiDetection, PiDetectionResult } from '@ai-pip/core/isl'

const detection: PiDetection = createPiDetection(
  'jailbreak',
  'ignore previous instructions',
  { start: 0, end: 25 },
  0.9
)

const result: PiDetectionResult = createPiDetectionResult([detection])
// result.score, result.detected, result.patterns are derived

console.log(hasDetections(result))                    // true
console.log(getDetectionCount(result))                // 1
console.log(getHighestConfidenceDetection(result))    // detection
console.log(isHighConfidence(detection))              // true
```

### 6.4 RiskScore and signal helpers

```typescript
import {
  createRiskScore,
  normalizeRiskScore,
  isHighRiskScore,
  isLowRiskScore,
  emitSignal,
  isHighRiskSignal,
  isLowRiskSignal
} from '@ai-pip/core/isl'

const score = createRiskScore(0.85)
console.log(isHighRiskScore(score))  // true (>= 0.7)

const signal = emitSignal(islResult)
console.log(isHighRiskSignal(signal))   // true if riskScore >= 0.7
console.log(isLowRiskSignal(signal))    // true if riskScore < 0.3
```

### 6.5 Pattern matching

```typescript
import {
  createPattern,
  matchesPattern,
  findMatch,
  MAX_CONTENT_LENGTH,
  MAX_PATTERN_LENGTH
} from '@ai-pip/core/isl'
import type { Pattern } from '@ai-pip/core/isl'

const pattern: Pattern = createPattern(
  'jailbreak',
  /ignore\s+previous\s+instructions/i,
  0.9,
  'Detects attempts to ignore previous instructions'
)
// description is optional

const content = 'Please ignore previous instructions'
console.log(matchesPattern(pattern, content))  // true

const match = findMatch(pattern, content)
if (match) {
  console.log(match.matched)           // "ignore previous instructions"
  console.log(match.position.start, match.position.end)
}
```

### 6.6 Full pipeline: CSL → ISL → signal for AAL

```typescript
import { segment } from '@ai-pip/core/csl'
import { sanitize, emitSignal } from '@ai-pip/core/isl'
import { resolveAgentAction } from '@ai-pip/core/aal'
import type { AgentPolicy } from '@ai-pip/core/aal'

const cslResult = segment({ content: userInput, source: 'UI', metadata: {} })
const islResult = sanitize(cslResult)
const islSignal = emitSignal(islResult)

const policy: AgentPolicy = {
  thresholds: { warn: 0.3, block: 0.7 },
  removal: { enabled: true }
}
const action = resolveAgentAction(islSignal, policy)  // 'ALLOW' | 'WARN' | 'BLOCK'
```

---

## 7. Integration with other layers

### 7.1 Input (from CSL)

- ISL receives **CSLResult**: segments with `id`, `content`, `trust`, `lineage`, etc.
- Sanitization level is derived from each segment’s **trust** (TC → minimal, STC → moderate, UC → aggressive).

### 7.2 Output (to CPE and AAL)

- **To CPE**: The same **ISLResult** (segments, lineage, metadata) can be passed to the CPE layer for envelope creation.
- **To AAL**: AAL must consume **ISLSignal** only. Use **`emitSignal(islResult)`** to obtain the signal; do not pass ISLResult to AAL.

---

## 8. File layout (reference)

In `@ai-pip/core`, the ISL layer is organized as:

```
isl/
├── index.ts
├── types.ts
├── sanitize.ts
├── signals.ts
├── value-objects/
│   ├── PiDetection.ts
│   ├── PiDetectionResult.ts
│   ├── Pattern.ts
│   ├── RiskScore.ts
│   └── index.ts
├── process/
│   ├── buildISLResult.ts
│   ├── emitSignal.ts
│   └── index.ts
├── lineage/
│   ├── buildISLLineage.ts
│   └── index.ts
└── exceptions/
    └── SanitizationError.ts
```

---

## 9. Summary

| Item | Description |
|------|-------------|
| **Input** | CSLResult (segments with trust and lineage) |
| **Output** | ISLResult (internal), ISLSignal (external contract for AAL/SDK) |
| **Main API** | sanitize, emitSignal, createISLSignal, signal helpers (isHighRiskSignal, etc.) |
| **Value objects** | RiskScore, PiDetection, PiDetectionResult, Pattern; ISLSegment, ISLResult, ISLSignal |
| **Not in ISL** | PolicyRule, AnomalyScore, AnomalyAction, RemovedInstruction (see AAL) |
| **Guarantees** | Original content preserved; lineage updated; fail-secure by trust level; signal contract for downstream layers |

This document describes the ISL layer as implemented in the current semantic core. For policy-based decisions (ALLOW/WARN/BLOCK) and instruction removal plans, see the **AAL (Agent Action Lock)** layer documentation.
