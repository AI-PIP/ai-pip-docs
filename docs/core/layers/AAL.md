# AAL (Agent Action Lock)

**Layer type:** Hybrid (semantic core in `@ai-pip/core`, execution in SDK)  
**Consumes:** ISLSignal (for decisions); ISLResult (for remediation plan)  
**Produces:** AnomalyAction, DecisionReason, RemediationPlan, lineage

---

## 1. Overview

AAL (Agent Action Lock) is the **hybrid layer** of the AI-PIP protocol. The **core** defines the semantic contract: value objects, types, and pure decision functions. The **SDK** implements execution: applying the decision (ALLOW/WARN/BLOCK), performing remediation (e.g. AI-powered cleanup), and managing state.

### 1.1 Responsibilities

| In core | In SDK |
|--------|--------|
| Evaluate ISL signals against policy | Execute ALLOW/WARN/BLOCK |
| Compute AnomalyAction (semantic decision) | Enforce policy (e.g. block request, log, notify) |
| Build DecisionReason (audit) | Persist logs, send to observability |
| Build RemediationPlan (what to clean: goals, constraints, target segments) | Perform cleanup (e.g. AI tool) using the plan |
| Build AAL lineage entry | Orchestrate pipeline, apply CPE per layer for integrity |

AAL describes **what** to do (goals, constraints, which segments need remediation). The SDK (or an AI agent) decides **how** to do it (e.g. call an AI cleanup tool that removes malicious instructions while preserving user intent).

### 1.2 Design principles

- **Decisions use only ISLSignal.** Resolving action (ALLOW/WARN/BLOCK) and building DecisionReason use only the signal. Layer separation is preserved.
- **Remediation plan from ISLResult.** When the SDK has access to ISLResult, the core builds a **RemediationPlan** with `targetSegments` (segment IDs with detections), `goals` (e.g. remove_prompt_injection), and `constraints` (e.g. preserve_user_intent). The core does **not** perform content removal; the SDK or an AI tool does.
- **No side effects in core.** All AAL core functions are pure: same inputs → same outputs, no I/O, no mutable state.
- **Policy describes intent.** `AgentPolicy` is a configuration object; the core interprets it to produce decisions and remediation plans. The SDK is responsible for enforcing the action and executing remediation.

---

## 2. Contract: ISL → AAL

AAL’s input from ISL is **ISLSignal** for decisions and **ISLResult** for the remediation plan.

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

For **decisions** (resolveAgentAction, buildDecisionReason), AAL uses only the signal. For the **remediation plan** (buildRemediationPlan), the caller passes ISLResult so the core can determine which segments have detections and derive goals from detection types.

### 2.2 Flow

**Decision path (signal only):**

```
ISL.sanitize(cslResult) → ISLResult (internal)
ISL.emitSignal(islResult) → ISLSignal
AAL.resolveAgentAction(islSignal, policy) → AnomalyAction
AAL.buildDecisionReason(action, islSignal, policy) → DecisionReason
```

The SDK uses `AnomalyAction` and `DecisionReason` to execute behavior (e.g. block call, log).

**Remediation plan path (when you have ISLResult):**

```
ISLResult + policy → AAL.buildRemediationPlan(islResult, policy) → RemediationPlan
```

The RemediationPlan contains `strategy`, `goals`, `constraints`, and `targetSegments`. The SDK (or an AI agent) uses this plan to clean the content of the target segments—for example, by calling an AI tool that removes malicious instructions while respecting the constraints. The core does **not** apply any removal; it only produces the plan.

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

### 3.3 AgentPolicy

Configuration that drives AAL decisions. Pure data; no logic inside.

```ts
interface AgentPolicy {
  thresholds: {
    warn: RiskScore   // 0..1
    block: RiskScore  // 0..1
  }
  remediation: {
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
- **remediation.enabled:** If `true`, `buildRemediationPlan` can return a plan with non-empty `targetSegments` and `goals` when threats exist; if `false`, the plan has `needsRemediation: false` and empty lists.
- **mode / explain:** Reserved for SDK or future use; core does not interpret them.

Typical convention: `0 ≤ warn < block ≤ 1`. Core does not enforce ordering.

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
// ... RoleProtectionConfig, ContextLeakPreventionConfig, etc.
```

**Creation:** `createPolicyRule(...)`  
**Predicates:** `isIntentBlocked`, `isScopeSensitive`, `isRoleProtected`, etc.

Core only exposes these helpers; it does not use PolicyRule inside `resolveAgentAction` or `buildRemediationPlan`. The SDK can use PolicyRule to refine behavior.

### 3.5 RemediationPlan

Describes *what* to do for remediation, not *how*. The SDK (or an AI agent) performs the actual cleanup.

```ts
interface RemediationPlan {
  readonly strategy: string           // e.g. 'AI_CLEANUP'
  readonly goals: readonly string[]   // e.g. 'remove_prompt_injection', 'remove_role_hijacking'
  readonly constraints: readonly string[]  // e.g. 'preserve_user_intent', 'do_not_add_information'
  readonly targetSegments: readonly string[]  // segment IDs with detections
  readonly needsRemediation: boolean
}
```

- **strategy:** Identifier for the consumer (e.g. `'AI_CLEANUP'`).
- **goals:** Derived from ISL detection types (e.g. `prompt-injection` → `remove_prompt_injection`).
- **constraints:** Fixed set (e.g. preserve_user_intent, do_not_add_information, do_not_change_language) that the cleanup must respect.
- **targetSegments:** IDs of segments that have at least one detection; the SDK/AI agent should run cleanup on these segments only.
- **needsRemediation:** `true` when there are target segments and `policy.remediation.enabled` is true.

If `!policy.remediation.enabled` or there are no segments with detections, `targetSegments` and `goals` are empty and `needsRemediation` is false.

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

### 3.7 Display constants (SDK/UI)

```ts
const ACTION_DISPLAY_COLORS: Record<AnomalyAction, string> = {
  ALLOW: 'green',
  WARN: 'yellow',
  BLOCK: 'red'
}

function getActionDisplayColor(action: AnomalyAction): string
```

Use `getActionDisplayColor(action)` for consistent presentation in logs or UI.

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

- Returns `{ score: islSignal.riskScore, action }` for auditing or downstream use.

### 4.3 buildDecisionReason

```ts
function buildDecisionReason(
  action: AnomalyAction,
  islSignal: ISLSignal,
  policy: AgentPolicy
): DecisionReason
```

- Builds a `DecisionReason` with the action, risk score, the relevant threshold, a short reason string, and threat/detection counts from `islSignal`.

### 4.4 buildRemediationPlan

```ts
function buildRemediationPlan(islResult: ISLResult, policy: AgentPolicy): RemediationPlan
```

- Builds a **RemediationPlan** from the ISL result and policy.
- If `!policy.remediation.enabled` → returns plan with `needsRemediation: false`, empty `goals` and `targetSegments`.
- Otherwise, collects segment IDs that have `piDetection.detections.length > 0` into `targetSegments`, and derives `goals` from the detection `pattern_type` (e.g. `prompt-injection` → `remove_prompt_injection`). Uses fixed `constraints` (e.g. preserve_user_intent, do_not_add_information, do_not_change_language) and `strategy: 'AI_CLEANUP'`.
- The SDK (or an AI agent) uses this plan to clean the content of the target segments; the core does **not** perform any removal.

### 4.5 buildAALLineage

```ts
function buildAALLineage(
  previousLineage: readonly LineageEntry[],
  timestamp?: number
): readonly LineageEntry[]
```

- Appends one lineage entry with step `'AAL'` and the given (or current) timestamp.

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
  remediation: { enabled: true }
}
const action = resolveAgentAction(islSignal, policy)
// action === 'ALLOW' | 'WARN' | 'BLOCK' → SDK acts accordingly
```

### 5.2 Action + audit reason + remediation plan

```ts
import {
  resolveAgentAction,
  buildDecisionReason,
  buildRemediationPlan
} from '@ai-pip/core/aal'

const action = resolveAgentAction(islSignal, policy)
const reason = buildDecisionReason(action, islSignal, policy)
const remediationPlan = buildRemediationPlan(islResult, policy)

if (action === 'BLOCK') {
  // SDK: reject request, log reason.reason, reason.riskScore, reason.detectionCount
}
if (remediationPlan.needsRemediation) {
  // SDK or AI agent: run cleanup on remediationPlan.targetSegments
  // using remediationPlan.goals and remediationPlan.constraints
}
```

### 5.3 Full flow with remediation (SDK performs cleanup)

When you have the ISL result and want the SDK (or an AI agent) to clean affected segments before sending content to the LLM:

```ts
import {
  resolveAgentAction,
  buildDecisionReason,
  buildRemediationPlan,
  getActionDisplayColor
} from '@ai-pip/core/aal'

const islResult = sanitize(cslResult)
const islSignal = emitSignal(islResult)
const action = resolveAgentAction(islSignal, policy)
const reason = buildDecisionReason(action, islSignal, policy)
const remediationPlan = buildRemediationPlan(islResult, policy)

if (action === 'BLOCK') {
  // Option: still run cleanup for audit, or block without cleanup
  // remediationPlan.targetSegments, .goals, .constraints → pass to your AI cleanup tool
}

if (remediationPlan.needsRemediation) {
  // SDK: call your AI cleanup tool with islResult.segments filtered by
  // remediationPlan.targetSegments; instruct it to follow goals and constraints.
  // Use the cleaned content for the prompt sent to the LLM.
}
```

`getActionDisplayColor(action)` returns `'green' | 'yellow' | 'red'` for ALLOW/WARN/BLOCK.

---

## 6. Threshold semantics and edge cases

- **Boundary:** If `riskScore === policy.thresholds.warn`, result is WARN; if `riskScore === policy.thresholds.block`, result is BLOCK (≥ is used).
- **Ordering:** Core does not enforce `warn < block`. Best practice: `0 ≤ warn < block ≤ 1`.
- **Remediation vs action:** The remediation plan is independent of the action. You can have `remediation.enabled: true` and `needsRemediation: true` even when action is ALLOW. The SDK can choose to run cleanup only on WARN/BLOCK or whenever `needsRemediation` is true.

---

## 7. What the core does not do

- **Does not** execute ALLOW/WARN/BLOCK (no network, no logging, no user notification). It only computes the semantic action and reason.
- **Does not** access ISLResult for *decisions*; decisions use only ISLSignal. The core accepts ISLResult only for `buildRemediationPlan`.
- **Does not** perform content removal or cleanup. It produces a **RemediationPlan** (what to clean and where); the SDK or an AI agent performs the actual cleanup.
- **Does not** enforce ordering of `warn`/`block` or validate `AgentPolicy` beyond what is needed for the pure functions.
- **Does not** interpret `AgentPolicy.mode` or `explain`; those are for the SDK.

**What the core does do:** It computes the action from the signal, builds a decision reason for audit, and builds a remediation plan (strategy, goals, constraints, target segment IDs) so the SDK or an AI tool can clean the content. All of this is pure and side-effect-free.

---

## 8. Summary

| Item | Description |
|------|-------------|
| **Input (decisions)** | ISLSignal (riskScore, piDetection, hasThreats, timestamp) |
| **Input (remediation plan)** | ISLResult + AgentPolicy → buildRemediationPlan |
| **Output** | AnomalyAction, AnomalyScore, DecisionReason, RemediationPlan, lineage |
| **Policy** | AgentPolicy (thresholds, remediation.enabled); optional PolicyRule |
| **Remediation** | buildRemediationPlan(islResult, policy) → plan with targetSegments, goals, constraints; SDK/AI performs cleanup |
| **Display** | ACTION_DISPLAY_COLORS, getActionDisplayColor(action) |
| **Core role** | Pure decisions and remediation plan; no I/O, no content removal |
| **SDK role** | Execute action (block/log/allow), run remediation (e.g. AI cleanup), persist audit, apply CPE for integrity |

AAL keeps decisions on the signal contract, provides a remediation plan when ISLResult is available, and leaves both action enforcement and content cleanup to the SDK.
