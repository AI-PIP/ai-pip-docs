# AAL (Agent Action Lock)

**Layer type:** Hybrid (semantic core in `@ai-pip/core`, execution in SDK)  
**Consumes:** ISLSignal (from ISL)  
**Produces:** AnomalyAction, DecisionReason, RemovalPlan, lineage

---

## 1. Overview

AAL (Agent Action Lock) is the **hybrid layer** of the AI-PIP protocol. The **core** defines the semantic contract: value objects, types, and pure decision functions. The **SDK** implements execution: applying the decision (ALLOW/WARN/BLOCK), removing instructions, and managing state.

### 1.1 Responsibilities

| In core | In SDK |
|--------|--------|
| Evaluate ISL signals against policy | Execute ALLOW/WARN/BLOCK |
| Compute AnomalyAction (semantic decision) | Enforce policy (e.g. block request, log, notify) |
| Build DecisionReason (audit) | Persist logs, send to observability |
| Build RemovalPlan (what to remove) | Actually remove text from prompt / segments |
| Build AAL lineage entry | Apply removals, call model, handle errors |

### 1.2 Design principles

- **Consumes only ISLSignal.** AAL never receives or depends on ISLResult. Layer separation is preserved by the signal contract.
- **No side effects in core.** All AAL core functions are pure: same inputs → same outputs, no I/O, no mutable state.
- **Policy describes intent.** `AgentPolicy` is a configuration object; the core interprets it to produce decisions and plans. The SDK is responsible for acting on them.

---

## 2. Contract: ISL → AAL

AAL’s only input from ISL is **ISLSignal**.

### 2.1 ISLSignal (consumed by AAL)

```ts
interface ISLSignal {
  readonly riskScore: RiskScore        // 0..1, aggregated risk
  readonly piDetection: PiDetectionResult
  readonly hasThreats: boolean
  readonly timestamp: number
}
```

- **riskScore:** Single number in `[0, 1]` used for threshold comparison (warn/block).
- **piDetection:** Contains `detections: readonly PiDetection[]`, plus aggregated score and `detected` flag.
- **hasThreats:** Convenience for “any detection present.”
- **timestamp:** For lineage and auditing; core does not interpret it for decisions.

AAL does **not** receive:

- ISLResult (segments, full lineage, internal metadata).
- Raw segment content or trust levels; only what is encoded in the signal.

### 2.2 Flow

```
ISL.sanitize(cslResult) → ISLResult (internal)
ISL.emitSignal(islResult) → ISLSignal
AAL.resolveAgentAction(islSignal, policy) → AnomalyAction
AAL.buildDecisionReason(action, islSignal, policy) → DecisionReason
AAL.buildRemovalPlan(islSignal, policy) → RemovalPlan
```

The SDK then uses `AnomalyAction`, `DecisionReason`, and `RemovalPlan` to execute behavior (e.g. block call, log, strip instructions).

---

## 3. Types and value objects

### 3.1 AnomalyAction

```ts
type AnomalyAction = 'ALLOW' | 'WARN' | 'BLOCK'
```

Semantic outcome of policy evaluation:

- **ALLOW:** Risk below warn threshold; agent may proceed.
- **WARN:** Risk ≥ warn and < block; agent may proceed with caution (SDK may log or flag).
- **BLOCK:** Risk ≥ block threshold; agent should not proceed (SDK typically blocks the request).

### 3.2 AnomalyScore

Pairs the numeric risk with the chosen action (for auditing and downstream use).

```ts
type AnomalyScore = {
  readonly score: RiskScore
  readonly action: AnomalyAction
}
```

**Creation:** `createAnomalyScore(score, action)`  
**Predicates:** `isHighRisk(anomaly)`, `isWarnRisk(anomaly)`, `isLowRisk(anomaly)` (based on `action`).

- `score` must be in `[0, 1]`; `action` must be one of `'ALLOW' | 'WARN' | 'BLOCK'`.

### 3.3 AgentPolicy

Configuration that drives AAL decisions. Pure data; no logic inside.

```ts
interface AgentPolicy {
  thresholds: {
    warn: RiskScore   // 0..1
    block: RiskScore  // 0..1
  }
  removal: {
    enabled: boolean
  }
  mode?: 'strict' | 'balanced' | 'permissive'
  explain?: boolean
}
```

- **thresholds.warn / block:** Used by `resolveAgentAction`:  
  - riskScore ≥ block → BLOCK  
  - riskScore ≥ warn (and < block) → WARN  
  - else → ALLOW  
- **removal.enabled:** If `true`, `buildRemovalPlan` can return instructions to remove; if `false`, plan is empty and `removalEnabled: false`.
- **mode / explain:** Reserved for SDK or future use; core does not interpret them.

Typical convention: `0 ≤ warn < block ≤ 1`. Core does not enforce ordering; misconfiguration can make WARN unreachable.

### 3.4 PolicyRule (extended policy semantics)

`PolicyRule` models **what** is protected or forbidden, not **how** to react to a risk score. Used for semantic checks (e.g. “is this intent blocked?”, “is this role protected?”).

```ts
type PolicyRule = {
  readonly version: string
  readonly blockedIntents: readonly BlockedIntent[]
  readonly sensitiveScope: readonly SensitiveScope[]
  readonly roleProtection: RoleProtectionConfig
  readonly contextLeakPrevention: ContextLeakPreventionConfig
}

type RoleProtectionConfig = {
  readonly protectedRoles: readonly ProtectedRole[]
  readonly immutableInstructions: readonly ImmutableInstruction[]
}

type ContextLeakPreventionConfig = {
  readonly enabled: boolean
  readonly blockMetadataExposure: boolean
  readonly sanitizeInternalReferences: boolean
}
```

**Type aliases:** `BlockedIntent`, `SensitiveScope`, `ProtectedRole`, `ImmutableInstruction` are `string` (semantic labels).

**Creation:** `createPolicyRule(version, blockedIntents, sensitiveScope, roleProtection, contextLeakPrevention)`  
**Predicates:**

- `isIntentBlocked(policy, intent)`
- `isScopeSensitive(policy, scope)`
- `isRoleProtected(policy, role)`
- `isInstructionImmutable(policy, instruction)`
- `isContextLeakPreventionEnabled(policy)`

Core only exposes these helpers; it does not use PolicyRule inside `resolveAgentAction` or `buildRemovalPlan`. The SDK (or higher layers) can use PolicyRule to refine behavior (e.g. extra checks before ALLOW).

### 3.5 RemovedInstruction

Describes one instruction that should be removed. Used in `RemovalPlan`; actual removal is done by the SDK.

```ts
interface RemovedInstruction {
  readonly type: 'system_command' | 'role_swapping' | 'jailbreak' | 'override' | 'manipulation'
  readonly pattern: string
  readonly position: Position
  readonly description: string
}
```

- **type:** Threat category (aligned with ISL `PiDetection.pattern_type` where applicable).
- **pattern:** Matched pattern text from the detection.
- **position:** `{ start, end }` (inclusive start, exclusive end) in the original content.
- **description:** Human-readable reason (e.g. confidence and pattern type).

`Position` is shared (e.g. from `shared/types`): `{ start: number, end: number }`.

### 3.6 DecisionReason

Explains why a given AnomalyAction was chosen (for logs and audits).

```ts
interface DecisionReason {
  readonly action: AnomalyAction
  readonly riskScore: number
  readonly threshold: number
  readonly reason: string
  readonly hasThreats: boolean
  readonly detectionCount: number
}
```

- **threshold:** The threshold that was decisive (warn or block).
- **reason:** Text like “Risk score X exceeds block threshold Y” or “below warn threshold”, plus optional “N threat(s) detected.”

### 3.7 RemovalPlan

Describes whether and what to remove; execution is SDK’s responsibility.

```ts
interface RemovalPlan {
  readonly instructionsToRemove: readonly RemovedInstruction[]
  readonly shouldRemove: boolean
  readonly removalEnabled: boolean
}
```

- **instructionsToRemove:** Built from `islSignal.piDetection.detections` when `policy.removal.enabled` and `islSignal.hasThreats`; each detection is mapped to a `RemovedInstruction` (type from `pattern_type`, pattern, position, description with confidence).
- **shouldRemove:** `instructionsToRemove.length > 0`.
- **removalEnabled:** Copy of `policy.removal.enabled`.

If `!policy.removal.enabled` or `!islSignal.hasThreats`, `instructionsToRemove` is empty and `shouldRemove` is false.

---

## 4. Core API (pure functions)

### 4.1 resolveAgentAction

```ts
function resolveAgentAction(islSignal: ISLSignal, policy: AgentPolicy): AnomalyAction
```

- Compares `islSignal.riskScore` to `policy.thresholds.block` and `policy.thresholds.warn`.
- Returns `'BLOCK'` if riskScore ≥ block; else `'WARN'` if riskScore ≥ warn; else `'ALLOW'`.
- No I/O; deterministic.

### 4.2 resolveAgentActionWithScore

```ts
function resolveAgentActionWithScore(islSignal: ISLSignal, policy: AgentPolicy): AnomalyScore
```

- Calls `resolveAgentAction(islSignal, policy)` and wraps result with `islSignal.riskScore` in `createAnomalyScore(score, action)`.
- Returns an `AnomalyScore` for auditing or downstream use.

### 4.3 buildDecisionReason

```ts
function buildDecisionReason(
  action: AnomalyAction,
  islSignal: ISLSignal,
  policy: AgentPolicy
): DecisionReason
```

- Builds a `DecisionReason` with the action, risk score, the relevant threshold (warn or block), a short reason string, and threat/detection counts from `islSignal`.
- Use after `resolveAgentAction` to get an auditable explanation.

### 4.4 buildRemovalPlan

```ts
function buildRemovalPlan(islSignal: ISLSignal, policy: AgentPolicy): RemovalPlan
```

- If `!policy.removal.enabled` → returns plan with empty list, `shouldRemove: false`, `removalEnabled: false`.
- If `!islSignal.hasThreats` → returns plan with empty list, `shouldRemove: false`, `removalEnabled: true`.
- Otherwise maps `islSignal.piDetection.detections` to `RemovedInstruction[]` (type from `pattern_type`, pattern, position, description including confidence) and returns plan with `shouldRemove: true` when the list is non-empty.
- Does not modify content; only produces the plan.

### 4.5 buildAALLineage

```ts
function buildAALLineage(
  previousLineage: readonly LineageEntry[],
  timestamp?: number
): readonly LineageEntry[]
```

- Appends one lineage entry with step `'AAL'` and the given (or current) timestamp.
- Used to extend the pipeline lineage (CSL → ISL → … → AAL) for audit trails.
- `LineageEntry` and `createLineageEntry` / `addLineageEntry` are from CSL/shared.

---

## 5. Usage patterns

### 5.1 Minimal: action only

```ts
import { sanitize, emitSignal } from '@ai-pip/core/isl'
import { resolveAgentAction } from '@ai-pip/core/aal'
import type { AgentPolicy } from '@ai-pip/core/aal'

const islResult = sanitize(cslResult)
const islSignal = emitSignal(islResult)
const policy: AgentPolicy = {
  thresholds: { warn: 0.3, block: 0.7 },
  removal: { enabled: true }
}
const action = resolveAgentAction(islSignal, policy)
// action === 'ALLOW' | 'WARN' | 'BLOCK' → SDK acts accordingly
```

### 5.2 Action + audit reason + removal plan

```ts
import {
  resolveAgentAction,
  buildDecisionReason,
  buildRemovalPlan
} from '@ai-pip/core/aal'

const action = resolveAgentAction(islSignal, policy)
const reason = buildDecisionReason(action, islSignal, policy)
const plan = buildRemovalPlan(islSignal, policy)

if (action === 'BLOCK') {
  // SDK: reject request, log reason.reason, reason.riskScore, reason.detectionCount
}
if (plan.shouldRemove && plan.removalEnabled) {
  // SDK: remove plan.instructionsToRemove from prompt/segments (e.g. by position)
}
```

### 5.3 AnomalyScore and lineage

```ts
import {
  resolveAgentActionWithScore,
  buildAALLineage
} from '@ai-pip/core/aal'

const anomaly = resolveAgentActionWithScore(islSignal, policy)
// anomaly.score, anomaly.action
const lineageWithAAL = buildAALLineage(islResult.lineage)
// or use previous pipeline lineage; then attach to your audit payload
```

---

## 6. Threshold semantics and edge cases

- **Boundary:** If `riskScore === policy.thresholds.warn`, result is WARN; if `riskScore === policy.thresholds.block`, result is BLOCK (≥ is used).
- **Ordering:** Core does not enforce `warn < block`. If `block < warn`, BLOCK can still trigger when riskScore ≥ block; the WARN range may be empty or counterintuitive. Best practice: `0 ≤ warn < block ≤ 1`.
- **Removal vs action:** Removal plan is independent of action. You can have `removal.enabled: true` and `shouldRemove: true` even when action is ALLOW (e.g. low aggregated score but some detections). The SDK can choose to remove only on WARN/BLOCK or always when `shouldRemove` is true, depending on product policy.

---

## 7. What the core does not do

- **Does not** execute ALLOW/WARN/BLOCK (no network, no logging, no user notification).
- **Does not** remove or modify prompt/segment content; it only produces `RemovalPlan`.
- **Does not** access ISLResult or raw segments; only ISLSignal.
- **Does not** enforce ordering of `warn`/`block` or validate policy beyond what is needed for the pure functions (e.g. `createAnomalyScore` and `createPolicyRule` validate their own arguments).
- **Does not** interpret `AgentPolicy.mode` or `explain`; those are for the SDK.

---



## 8. Summary

| Item | Description |
|------|-------------|
| **Input** | ISLSignal only (riskScore, piDetection, hasThreats, timestamp) |
| **Output** | AnomalyAction, AnomalyScore, DecisionReason, RemovalPlan, lineage |
| **Policy** | AgentPolicy (thresholds, removal.enabled); optional PolicyRule for intent/role/scope semantics |
| **Core role** | Pure decisions and plans; no execution |
| **SDK role** | Execute action, apply removal, persist audit, call model |

AAL keeps the protocol’s layer separation (consume signal, not result), keeps the core side-effect-free, and leaves all execution and policy enforcement to the SDK.
