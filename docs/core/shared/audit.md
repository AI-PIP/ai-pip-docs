# Audit and Pretty-Print Pure Functions

Documentation of the AI-PIP core audit utilities: pure functions that format layer results (CSL, ISL, AAL, CPE) into ordered, human-readable text for logging, compliance, and debugging.

**Location in core:** `@ai-pip/core/shared` (audit)  
**Characteristics:** No side effects, accept minimal shapes (layer-agnostic), do not depend on layer internals.

---

## 1. Function Summary

| Function | Description |
|----------|-------------|
| **`formatLineageForAudit(lineage)`** | Formats an array of lineage entries for audit output. |
| **`formatCSLForAudit(result)`** | Pretty-prints CSL result (segments, trust, lineage). |
| **`formatISLForAudit(result)`** | Pretty-prints ISL result (sanitized segments, metadata, lineage). |
| **`formatISLSignalForAudit(signal)`** | Pretty-prints ISLSignal (risk score, threats, detections). |
| **`formatAALForAudit(reason, removalPlan?)`** | Pretty-prints AAL decision reason and optionally the removal plan. |
| **`formatCPEForAudit(result)`** | Pretty-prints CPE result (envelope, metadata, signature, lineage). |
| **`formatPipelineAudit(csl, isl, cpe, options?)`** | Full pipeline audit report CSL → ISL → CPE. |

---

## 2. Types (Minimal Shapes for Audit)

Functions accept objects that satisfy these interfaces, so you can pass the actual result of each layer or any compatible object.

### 2.1 LineageEntryLike

```ts
interface LineageEntryLike {
  readonly step: string
  readonly timestamp: number
}
```

Used for lineage in any layer.

---

### 2.2 CSLResultLike

```ts
interface CSLResultLike {
  readonly segments: ReadonlyArray<{
    readonly id: string
    readonly content: string
    readonly trust: { readonly value: string }
    readonly lineage?: readonly LineageEntryLike[]
  }>
  readonly lineage: readonly LineageEntryLike[]
  readonly processingTimeMs?: number
}
```

Minimal shape for CSL result used by `formatCSLForAudit`.

---

### 2.3 ISLResultLike

```ts
interface ISLResultLike {
  readonly segments: ReadonlyArray<{
    readonly id: string
    readonly originalContent: string
    readonly sanitizedContent: string
    readonly trust: { readonly value: string }
    readonly sanitizationLevel: string
    readonly lineage?: readonly LineageEntryLike[]
  }>
  readonly lineage: readonly LineageEntryLike[]
  readonly metadata: {
    readonly totalSegments: number
    readonly sanitizedSegments: number
    readonly processingTimeMs?: number
  }
}
```

Minimal shape for ISL result used by `formatISLForAudit`.

---

### 2.4 ISLSignalLike

```ts
interface ISLSignalLike {
  readonly riskScore: number
  readonly hasThreats: boolean
  readonly timestamp: number
  readonly piDetection: {
    readonly detected: boolean
    readonly score: number
    readonly detections: ReadonlyArray<unknown>
    readonly patterns?: readonly string[]
  }
}
```

Minimal shape for ISLSignal used by `formatISLSignalForAudit`.

---

### 2.5 DecisionReasonLike

```ts
interface DecisionReasonLike {
  readonly action: string
  readonly reason: string
  readonly riskScore: number
  readonly threshold: number
  readonly hasThreats: boolean
  readonly detectionCount: number
}
```

Minimal shape for AAL DecisionReason used by `formatAALForAudit`.

---

### 2.6 RemovalPlanLike

```ts
interface RemovalPlanLike {
  readonly shouldRemove: boolean
  readonly removalEnabled: boolean
  readonly instructionsToRemove: ReadonlyArray<{
    readonly type?: string
    readonly pattern?: string
    readonly description?: string
  }>
}
```

Minimal shape for AAL RemovalPlan used as the second argument of `formatAALForAudit`.

---

### 2.7 CPEResultLike

```ts
interface CPEResultLike {
  readonly envelope: {
    readonly metadata: {
      readonly timestamp: number
      readonly nonce: string
      readonly protocolVersion?: string
    }
    readonly signature: {
      readonly algorithm: string
      readonly value?: string
    }
    readonly lineage: readonly LineageEntryLike[]
  }
  readonly processingTimeMs?: number
}
```

Minimal shape for CPE result used by `formatCPEForAudit`.

---

## 3. Functions

### 3.1 formatLineageForAudit

Formats an array of lineage entries for audit output.

**Signature:**

```ts
function formatLineageForAudit(lineage: readonly LineageEntryLike[]): string
```

**Parameters:**

- **`lineage`**: Array of entries with `step` and `timestamp`.

**Returns:** String with one line per entry, numbered, with step and ISO date. If `lineage` is empty, returns `'Lineage: (none)'`.

**Example output:**

```
Lineage:
  1. [CSL] 2026-01-31T12:00:00.000Z
  2. [ISL] 2026-01-31T12:00:00.005Z
  3. [CPE] 2026-01-31T12:00:00.010Z
```

---

### 3.2 formatCSLForAudit

Formats CSL (Context Segmentation Layer) result for audit: segments, trust, processing time, and lineage.

**Signature:**

```ts
function formatCSLForAudit(result: CSLResultLike): string
```

**Parameters:**

- **`result`**: Object compatible with CSLResultLike (e.g. `CSLResult`).

**Returns:** Text block with title `[CSL]`, segment count, processing time (if present), one line per segment (id, trust, content length), and formatted lineage.

**Example output:**

```
[CSL] Context Segmentation Layer
---
Segments: 1
Processing time: 5ms
  Segment 1: id=seg-xxx trust=STC content_length=42

Lineage:
  1. [CSL] 2026-01-31T12:00:00.000Z
```

---

### 3.3 formatISLForAudit

Formats ISL (Instruction Sanitization Layer) result: sanitized segments, metadata, and lineage.

**Signature:**

```ts
function formatISLForAudit(result: ISLResultLike): string
```

**Parameters:**

- **`result`**: Object compatible with ISLResultLike (e.g. `ISLResult`).

**Returns:** Block with title `[ISL]`, total/sanitized segments, processing time (if present), one line per segment (id, trust, sanitization level, original/sanitized lengths), and lineage.

**Example output:**

```
[ISL] Instruction Sanitization Layer
---
Segments: 1 (sanitized: 1)
Processing time: 8ms
  Segment 1: id=seg-xxx trust=STC level=moderate
    original_length=42 sanitized_length=42

Lineage:
  1. [CSL] 2026-01-31T12:00:00.000Z
  2. [ISL] 2026-01-31T12:00:00.008Z
```

---

### 3.4 formatISLSignalForAudit

Formats ISLSignal (ISL external contract): risk score, hasThreats, timestamp, and detections.

**Signature:**

```ts
function formatISLSignalForAudit(signal: ISLSignalLike): string
```

**Parameters:**

- **`signal`**: Object compatible with ISLSignalLike (e.g. `ISLSignal`).

**Returns:** Block with title `[ISL Signal]`, risk score (3 decimals), hasThreats, ISO timestamp, detection count and piDetection score/detected; if `patterns` exist, they are listed.

**Example output:**

```
[ISL Signal] External contract
---
Risk score: 0.000
Has threats: false
Timestamp: 2026-01-31T12:00:00.000Z
Detections: 0 (score: 0.000, detected: false)
```

---

### 3.5 formatAALForAudit

Formats AAL decision reason and, optionally, the removal plan.

**Signature:**

```ts
function formatAALForAudit(
  reason: DecisionReasonLike,
  removalPlan?: RemovalPlanLike | null
): string
```

**Parameters:**

- **`reason`**: Object compatible with DecisionReasonLike (e.g. output of `buildDecisionReason`).
- **`removalPlan`**: Optional. Object compatible with RemovalPlanLike (e.g. output of `buildRemovalPlan`). If `null` or omitted, no removal block is included.

**Returns:** Block with title `[AAL]`, action, risk score and threshold, reason, threats and detection count. If `removalPlan` is passed, adds removal enabled, should remove, and list of instructions to remove (type and description or pattern).

**Example output (with removal plan):**

```
[AAL] Agent Action Lock
---
Action: ALLOW
Risk score: 0.120 (threshold: 0.300)
Reason: Risk score 0.120 is below warn threshold 0.300
Threats: false (count: 0)

Removal enabled: true
Should remove: false
```

---

### 3.6 formatCPEForAudit

Formats CPE (Cryptographic Prompt Envelope) result: envelope metadata, signature, and lineage.

**Signature:**

```ts
function formatCPEForAudit(result: CPEResultLike): string
```

**Parameters:**

- **`result`**: Object compatible with CPEResultLike (e.g. `CPEResult`).

**Returns:** Block with title `[CPE]`, nonce, ISO timestamp, protocol version (if present), signature algorithm, processing time (if present), and envelope lineage.

**Example output:**

```
[CPE] Cryptographic Prompt Envelope
---
Nonce: abc123...
Timestamp: 2026-01-31T12:00:00.010Z
Protocol version: 1.0
Signature algorithm: HMAC-SHA256
Processing time: 2ms

Lineage:
  1. [CSL] 2026-01-31T12:00:00.000Z
  2. [ISL] 2026-01-31T12:00:00.008Z
  3. [CPE] 2026-01-31T12:00:00.010Z
```

---

### 3.7 formatPipelineAudit

Builds a full pipeline audit report (CSL → ISL → CPE) by concatenating each layer’s formatter output.

**Signature:**

```ts
function formatPipelineAudit(
  csl: CSLResultLike,
  isl: ISLResultLike,
  cpe: CPEResultLike,
  options?: { title?: string; sectionSeparator?: string }
): string
```

**Parameters:**

- **`csl`**: CSL result (or compatible shape).
- **`isl`**: ISL result (or compatible shape).
- **`cpe`**: CPE result (or compatible shape).
- **`options`**: Optional.
  - **`title`**: Title at the start of the report (default: `'AI-PIP Pipeline Audit'`).
  - **`sectionSeparator`**: String between sections (default: `'\n\n'`).

**Returns:** A single string with title, separator, then `formatCSLForAudit(csl)`, separator, `formatISLForAudit(isl)`, separator, and `formatCPEForAudit(cpe)`.

---

## 4. Usage and Imports

All functions and types are exported from `@ai-pip/core/shared`:

```ts
import {
  formatLineageForAudit,
  formatCSLForAudit,
  formatISLForAudit,
  formatISLSignalForAudit,
  formatAALForAudit,
  formatCPEForAudit,
  formatPipelineAudit
} from '@ai-pip/core/shared'

import type {
  LineageEntryLike,
  CSLResultLike,
  ISLResultLike,
  ISLSignalLike,
  DecisionReasonLike,
  RemovalPlanLike,
  CPEResultLike
} from '@ai-pip/core/shared'
```

From the main package:

```ts
import {
  formatCSLForAudit,
  formatISLForAudit,
  formatISLSignalForAudit,
  formatAALForAudit,
  formatCPEForAudit,
  formatPipelineAudit
} from '@ai-pip/core'
```

---

## 5. Full Example: Pipeline + AAL

```ts
import { segment, sanitize, emitSignal, envelope } from '@ai-pip/core'
import {
  resolveAgentAction,
  buildDecisionReason,
  buildRemovalPlan,
  formatCSLForAudit,
  formatISLForAudit,
  formatISLSignalForAudit,
  formatAALForAudit,
  formatCPEForAudit,
  formatPipelineAudit
} from '@ai-pip/core'
import type { AgentPolicy } from '@ai-pip/core'

const cslResult = segment({ content: 'User input', source: 'UI', metadata: {} })
const islResult = sanitize(cslResult)
const islSignal = emitSignal(islResult)
const cpeResult = envelope(islResult, 'secret-key')

const policy: AgentPolicy = {
  thresholds: { warn: 0.3, block: 0.7 },
  removal: { enabled: true }
}
const action = resolveAgentAction(islSignal, policy)
const reason = buildDecisionReason(action, islSignal, policy)
const removalPlan = buildRemovalPlan(islSignal, policy)

// Pretty-print per layer
console.log(formatCSLForAudit(cslResult))
console.log(formatISLForAudit(islResult))
console.log(formatISLSignalForAudit(islSignal))
console.log(formatAALForAudit(reason, removalPlan))
console.log(formatCPEForAudit(cpeResult))

// Full pipeline report (CSL → ISL → CPE)
const fullAudit = formatPipelineAudit(cslResult, islResult, cpeResult, {
  title: 'AI-PIP Pipeline Audit',
  sectionSeparator: '\n\n'
})
console.log(fullAudit)
```

---

## 6. Implementation Notes

- **Internal constants:** Functions use line indent `  ` (two spaces) and border `---` for consistent styling.
- **Layer-agnostic:** They accept minimal shapes (the `*Like` interfaces), so they work with actual layer results or test/mock objects.
- **Purity:** No I/O, no mutation; same input → same output. Suitable for logging, compliance, and tests.
- **Order:** Output always follows the same order (title, metadata, segments/details, lineage) for readability and simple parsing.

This document describes the core audit and pretty-print pure functions; for lineage helpers (addLineageEntry, filterLineageByStep, etc.) see the **Shared** documentation in the protocol documentation repository.
