# ISL (Instruction Sanitization Layer)

> **Instruction Sanitization Layer** — Second layer of the AI-PIP protocol. Sanitizes content from CSL, detects prompt-injection patterns, and emits signals (ISLSignal) for AAL and the SDK.

---

## 1. Overview

The **Instruction Sanitization Layer (ISL)** receives segmented content from CSL and applies trust-level–based sanitization. It preserves the original content for audit, optionally runs prompt-injection detection, and exposes a **signal** (ISLSignal) so other layers (e.g. AAL) can make decisions without depending on ISL internals.

### 1.1 Principles

- **Differentiated sanitization**: Level depends on segment trust (TC → minimal, STC → moderate, UC → aggressive).
- **Prompt-injection detection**: Identifies malicious patterns and builds risk scores and detections (PiDetection, PiDetectionResult).
- **Configurable risk score**: The aggregated risk on the signal is computed from detections using a **strategy** (e.g. max confidence, severity plus volume, weighted by type). The strategy is chosen at emit time and stored in the signal metadata for auditability.
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
ISLResult (internal: segments, lineage, metadata; segments may have piDetection)
    ↓
emitSignal(islResult, options?)  [optional, for downstream layers]
    ↓
  Aggregates all segment detections → chooses risk score strategy → computes riskScore [0,1]
    ↓
ISLSignal (external contract: riskScore, piDetection, hasThreats, timestamp, metadata?.strategy)
    → consumed by AAL / SDK
```

- **sanitize**: Single public entry for sanitization; returns **ISLResult**. Optionally runs threat detection per segment (`piDetection`).
- **emitSignal**: Converts **ISLResult** → **ISLSignal**. Aggregates detections from all segments and computes **riskScore** using a configurable **risk score strategy** (default: `MAX_CONFIDENCE`). The strategy used is stored in `ISLSignal.metadata.strategy` for auditability.

---

## 3. Main API

### 3.1 Sanitization

| Function | Description |
|----------|-------------|
| **`sanitize(cslResult: CSLResult): ISLResult`** | Sanitizes CSL segments by trust level (TC → minimal, STC → moderate, UC → aggressive). Returns internal result with segments, lineage, and metadata. |

### 3.2 Signals (external contract)

| Function | Description |
|----------|-------------|
| **`emitSignal(islResult: ISLResult, options?: EmitSignalOptions \| number): ISLSignal`** | Builds the external signal from an internal ISLResult. Aggregates detections from all segments and computes risk score using the chosen **risk score strategy** (see §5). Second argument can be a **number** (timestamp) for backward compatibility, or an **options** object. |
| **`createISLSignal(riskScore, piDetection, timestamp?, metadata?): ISLSignal`** | Builds an ISLSignal from risk score, PiDetectionResult, optional timestamp, and optional metadata (e.g. `strategy`). |
| **`isHighRiskSignal(signal, threshold?)`** | `true` if `signal.riskScore >= threshold` (default 0.7). |
| **`isMediumRiskSignal(signal, lowThreshold?, highThreshold?)`** | `true` if risk score is in [low, high) (defaults 0.3, 0.7). |
| **`isLowRiskSignal(signal, threshold?)`** | `true` if `signal.riskScore < threshold` (default 0.3). |

**EmitSignalOptions**

```ts
interface EmitSignalOptions {
  readonly timestamp?: number
  readonly riskScore?: {
    readonly strategy: RiskScoreStrategy
    readonly typeWeights?: Record<string, number>
  }
}
```

- **timestamp**: When the signal was emitted (default: `Date.now()`).
- **riskScore.strategy**: Which risk score formula to use (default: `RiskScoreStrategy.MAX_CONFIDENCE`). Only registered strategies are allowed; see §5.
- **riskScore.typeWeights**: Optional per-`pattern_type` weights for `WEIGHTED_BY_TYPE` strategy. Omitted or `undefined` uses default weights.

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
  readonly metadata?: ISLSignalMetadata
}

interface ISLSignalMetadata {
  readonly strategy: RiskScoreStrategy
}
```

- **riskScore**: Aggregated risk (0–1), computed from detections using the strategy in `metadata.strategy`.
- **piDetection**: All detections and aggregated score.
- **hasThreats**: Shortcut for “any detection” (`piDetection.detected`).
- **timestamp**: Processing time for audit.
- **metadata**: Optional; when present, **metadata.strategy** is the risk score strategy used to compute **riskScore** (auditability and reproducibility).

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

## 5. Risk score strategies and calculators

The **risk score** on the signal is derived solely from aggregated segment detections. The formula is chosen at **emit** time via a **strategy**; only registered strategies are allowed (no custom inline formulas), so results are auditable and reproducible.

### 5.1 Strategy enum and calculator interface

```ts
enum RiskScoreStrategy {
  MAX_CONFIDENCE = 'max-confidence',
  SEVERITY_PLUS_VOLUME = 'severity-plus-volume',
  WEIGHTED_BY_TYPE = 'weighted-by-type'
}

interface RiskScoreCalculator {
  readonly strategy: RiskScoreStrategy
  calculate(detections: readonly PiDetection[]): number
}
```

- **RiskScoreStrategy**: The only allowed strategy identifiers. AAL and the SDK do **not** choose the formula; the caller of **emitSignal** (or sanitize, if extended) does.
- **RiskScoreCalculator**: Pure, deterministic, no side effects. **calculate** returns a **raw** score; the caller (e.g. emitSignal) clamps it to [0, 1] with **normalizeRiskScore**.

### 5.2 Registered calculators and formulas

| Strategy | Description | Formula (raw) |
|----------|-------------|----------------|
| **MAX_CONFIDENCE** | Default; single highest confidence. | `max(d.confidence)` over all detections |
| **SEVERITY_PLUS_VOLUME** | Highest confidence plus a small bump per extra detection. | `min(1, max(d.confidence) + 0.1 * (detections.length - 1))` |
| **WEIGHTED_BY_TYPE** | Weight by `pattern_type`; unknown types use weight 1. | `min(1, max(d.confidence * (weights[d.pattern_type] ?? 1)))` |

**Exports**

- **maxConfidenceCalculator**, **severityPlusVolumeCalculator**: Fixed calculators for the first two strategies.
- **weightedByTypeCalculator(weights: Record<string, number>): RiskScoreCalculator**: Returns a calculator for **WEIGHTED_BY_TYPE** with the given weights.
- **defaultWeightedByTypeCalculator**: Pre-built calculator with **DEFAULT_TYPE_WEIGHTS** (all pattern types weight 1).
- **DEFAULT_TYPE_WEIGHTS**: Frozen map of default weights (e.g. `prompt-injection`, `jailbreak`, `role_hijacking`, `script_like`, `hidden_text` → 1).
- **getCalculator(strategy, typeWeights?): RiskScoreCalculator**: Returns the registered calculator for the strategy. For **WEIGHTED_BY_TYPE**, pass **typeWeights** to use custom weights; omit to use default weights.

### 5.3 Rules (auditability and reproducibility)

- The strategy is decided **once** (at emit time). It is stored on **ISLSignal.metadata.strategy**.
- **No per-segment strategy**: The same strategy applies to the whole aggregated detection set.
- **No dynamic strategy**: Strategy cannot change based on content or runtime conditions.
- **No custom inline strategies**: Only the registered enum values and calculators are allowed.

---

## 6. Exceptions

- **`SanitizationError`**: Thrown when sanitization fails (invalid content or validation error). Optional `cause` for chaining.

---

## 7. Usage examples

### 7.1 Basic sanitization

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

### 7.2 Emit signal for AAL

```typescript
import { sanitize, emitSignal } from '@ai-pip/core/isl'
import type { ISLSignal } from '@ai-pip/core/isl'

const islResult = sanitize(cslResult)

// Default: MAX_CONFIDENCE strategy; timestamp = Date.now()
const islSignal: ISLSignal = emitSignal(islResult)

// AAL consumes islSignal (not islResult):
// - islSignal.riskScore, islSignal.piDetection, islSignal.hasThreats, islSignal.timestamp
// - islSignal.metadata?.strategy (risk score strategy used)
```

**With options (strategy and timestamp)**

```typescript
import { emitSignal, RiskScoreStrategy } from '@ai-pip/core/isl'

// Explicit strategy; default MAX_CONFIDENCE if omitted
const signal = emitSignal(islResult, {
  timestamp: Date.now(),
  riskScore: { strategy: RiskScoreStrategy.SEVERITY_PLUS_VOLUME }
})
// signal.metadata.strategy === 'severity-plus-volume'

// WEIGHTED_BY_TYPE with custom type weights
const signalWeighted = emitSignal(islResult, {
  riskScore: {
    strategy: RiskScoreStrategy.WEIGHTED_BY_TYPE,
    typeWeights: { 'prompt-injection': 1.2, jailbreak: 1.0 }
  }
})

// Backward compatibility: second argument as number = timestamp
const signalLegacy = emitSignal(islResult, 1234567890)
```

### 7.3 PiDetection and PiDetectionResult

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

### 7.4 RiskScore and signal helpers

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

### 7.5 Pattern matching

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

### 7.6 Full pipeline: CSL → ISL → signal for AAL

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

### 7.7 Risk score strategies and getCalculator

```typescript
import {
  getCalculator,
  RiskScoreStrategy,
  maxConfidenceCalculator,
  severityPlusVolumeCalculator,
  weightedByTypeCalculator,
  DEFAULT_TYPE_WEIGHTS
} from '@ai-pip/core/isl'
import type { RiskScoreCalculator } from '@ai-pip/core/isl'

// Get calculator for a strategy (used internally by emitSignal)
const calc = getCalculator(RiskScoreStrategy.MAX_CONFIDENCE)
const rawScore = calc.calculate(detections)  // 0 if empty; otherwise max(confidence)

// SEVERITY_PLUS_VOLUME: max confidence + 0.1 per extra detection, capped at 1
const calcSev = getCalculator(RiskScoreStrategy.SEVERITY_PLUS_VOLUME)

// WEIGHTED_BY_TYPE: custom weights or default
const calcWeighted = getCalculator(RiskScoreStrategy.WEIGHTED_BY_TYPE, {
  'prompt-injection': 1.2,
  jailbreak: 1
})
// Or use default weights:
const calcDefaultWeighted = getCalculator(RiskScoreStrategy.WEIGHTED_BY_TYPE)

// Direct use of calculators (same as getCalculator)
const raw = maxConfidenceCalculator.calculate(detections)
```

---

## 8. Integration with other layers

### 8.1 Input (from CSL)

- ISL receives **CSLResult**: segments with `id`, `content`, `trust`, `lineage`, etc.
- Sanitization level is derived from each segment’s **trust** (TC → minimal, STC → moderate, UC → aggressive).

### 8.2 Output (to CPE and AAL)

- **To CPE**: The same **ISLResult** (segments, lineage, metadata) can be passed to the CPE layer for envelope creation.
- **To AAL**: AAL must consume **ISLSignal** only. Use **`emitSignal(islResult)`** to obtain the signal; do not pass ISLResult to AAL.

---

## 9. File layout (reference)

In `@ai-pip/core`, the ISL layer is organized as:

```
isl/
├── index.ts
├── types.ts
├── sanitize.ts
├── signals.ts
├── riskScore/
│   ├── types.ts          # RiskScoreStrategy, RiskScoreCalculator
│   ├── calculators.ts    # maxConfidence, severityPlusVolume, weightedByType
│   └── index.ts         # getCalculator, exports
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

## 10. Summary

| Item | Description |
|------|-------------|
| **Input** | CSLResult (segments with trust and lineage) |
| **Output** | ISLResult (internal), ISLSignal (external contract for AAL/SDK) |
| **Main API** | sanitize, emitSignal (with optional risk score strategy), createISLSignal, signal helpers (isHighRiskSignal, etc.), getCalculator, RiskScoreStrategy |
| **Risk score** | Derived from aggregated detections; strategy chosen at emit time (MAX_CONFIDENCE, SEVERITY_PLUS_VOLUME, WEIGHTED_BY_TYPE); stored in ISLSignal.metadata.strategy |
| **Value objects** | RiskScore, PiDetection, PiDetectionResult, Pattern; ISLSegment, ISLResult, ISLSignal |
| **Not in ISL** | PolicyRule, AnomalyScore, AnomalyAction, RemovedInstruction (see AAL) |
| **Guarantees** | Original content preserved; lineage updated; fail-secure by trust level; signal contract for downstream layers; auditable risk score strategy |

This document describes the ISL layer as implemented in the current semantic core. For policy-based decisions (ALLOW/WARN/BLOCK) and instruction removal plans, see the **AAL (Agent Action Lock)** layer documentation.
