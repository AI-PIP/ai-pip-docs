# Semantic Architecture of the AI-PIP Protocol

> Architecture document of the AI Prompt Integrity Protocol (AI-PIP)
> 
> This document describes the semantic architecture of the protocol, not a concrete implementation.

**Version**: 2.0  
**Author**: Felipe Masliah  
**Last Updated**: 2026-01-01

---

## 📋 Table of Contents

1. [Introduction](#1-introduction)
2. [Protocol vs Implementation Separation](#2-protocol-vs-implementation-separation)
3. [Fundamental Principles](#3-fundamental-principles)
4. [Protocol Flow](#4-protocol-flow)
   - [4.1 Semantic Processing Pipeline](#41-semantic-processing-pipeline)
   - [4.2 Fail-Secure Principle](#42-fail-secure-principle)
5. [Protocol Layers](#5-protocol-layers)
   - [5.1 CSL: Context Segmentation Layer](#51-csl-context-segmentation-layer)
   - [5.2 ISL: Instruction Sanitization Layer](#52-isl-instruction-sanitization-layer)
   - [5.3 CPE: Cryptographic Prompt Envelope](#53-cpe-cryptographic-prompt-envelope)
   - [5.4 ModelGateway: Deployment Infrastructure](#54-modelgateway-deployment-infrastructure)
6. [Semantic Contracts Between Layers](#6-semantic-contracts-between-layers)
   - [6.1 CSL → ISL Contract](#61-csl--isl-contract)
   - [6.2 ISL → CPE Contract](#62-isl--cpe-contract)
   - [6.3 CPE → ModelGateway Contract](#63-cpe--modelgateway-contract)
7. [Protocol Guarantees](#7-protocol-guarantees)
   - [7.1 Semantic Integrity Guarantee](#71-semantic-integrity-guarantee)
   - [7.2 Traceability Guarantee](#72-traceability-guarantee)
   - [7.3 Composition Guarantee](#73-composition-guarantee)
8. [Design Principles](#8-design-principles)
9. [Conclusion](#9-conclusion)

---

## 1. Introduction

This document describes the **semantic architecture** of the AI-PIP protocol. The protocol is defined as an abstract specification composed of pure functions, immutable data structures, and semantic contracts between components.

**Important**: This document describes **what** the protocol must fulfill, not **how** it is implemented. Implementation details (normalization, heuristic detection, hashing, policies, serialization) are the responsibility of the SDK or specific implementations.

---

## 2. Protocol vs Implementation Separation

### 2.1 Fundamental Principle

> **The protocol defines semantic contracts. The SDK implements mechanisms.**

This document describes the architecture of the AI-PIP protocol, not a concrete implementation. The functions described in each layer represent **semantic contracts** that must be fulfilled. Implementation details (normalization, heuristic detection, hashing, policies, serialization) are not part of the protocol core and are the responsibility of the SDK or specific implementations.

### 2.2 Protocol Responsibilities (Semantic Core)

The protocol defines:

- ✅ **Semantic Structures**: Immutable data types that represent the processing state
- ✅ **Pure Functions**: Deterministic transformations without side effects
- ✅ **Inter-Layer Contracts**: Formal specifications of interfaces between components
- ✅ **Risk Signals**: Structures that describe semantic conditions (TrustLevel, AnomalyScore, PiDetection)
- ✅ **Composition Rules**: How layers combine to form the complete pipeline

### 2.3 Implementation Responsibilities (SDK)

The implementation provides:

- ❌ **Normalization**: Removal of invisible characters, Unicode normalization
- ❌ **Heuristic Detection**: Pattern identification through heuristics
- ❌ **Cryptographic Hashing**: Generation of SHA-256/SHA-512 hashes
- ❌ **Policies and Decisions**: Application of blocking rules, access validation
- ❌ **Serialization**: Specific format for transmission and storage
- ❌ **Cryptographic Verification**: Signature and integrity validation

### 2.4 Core / SDK Relationship

```
┌─────────────────────────────────────┐
│   Semantic Core (Protocol)          │
│  - Defines structures               │
│  - Defines pure functions          │
│  - Defines contracts                │
│  - Produces signals                 │
└──────────────┬──────────────────────┘
               │ used by
               ▼
┌─────────────────────────────────────┐
│      SDK (Implementation)           │
│  - Implements normalization        │
│  - Implements detection             │
│  - Implements hashing               │
│  - Implements policies              │
│  - Executes actions                 │
└─────────────────────────────────────┘
```

---

## 3. Fundamental Principles

### 3.1 Formal Semantic Protocol

AI-PIP is a formal semantic protocol defined through:

- **Pure Functions**: Without side effects, deterministic
- **Immutable Structures**: Data types that do not change after creation
- **Functional Composition**: Pipeline built through composition of pure functions

### 3.2 Separation of Responsibilities

- **Protocol**: Defines **what** must occur
- **Implementation**: Defines **how** it occurs

### 3.3 Signals, Not Actions

The protocol produces **semantic signals** (TrustLevel, AnomalyScore, PiDetection), it does not execute actions (blocks, validations). Actions belong to the implementation.

---

## 4. Protocol Flow

### 4.1 Semantic Processing Pipeline

The protocol processes content through a pure functional pipeline:

```
┌─────────────────────────────────────────────────────────┐
│                    SEMANTIC CORE                        │
│                                                          │
│  CSLInput                                               │
│    │                                                     │
│    ▼                                                     │
│  segment() → CSLResult                                  │
│    │  (signals: TrustLevel, LineageEntry)               │
│    ▼                                                     │
│  sanitize() → ISLResult                                 │
│    │  (signals: PiDetection, AnomalyScore)            │
│    ▼                                                     │
│  envelope() → CPEResult                                 │
│    │  (structure: CPEEnvelope)                         │
│    ▼                                                     │
│  Semantic Signals                                       │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│                    SDK / IMPLEMENTATION                 │
│                                                          │
│  - Reads DOM                                            │
│  - Generates hashes                                     │
│  - Serializes content                                   │
│  - Verifies signatures                                  │
│  - Decides actions (ALLOW/BLOCK/WARN)                  │
│  - Blocks if necessary                                  │
└─────────────────────────────────────────────────────────┘
```

### 4.2 Fail-Secure Principle

The protocol operates under the **fail-secure** principle at the semantic level: if any layer produces signals indicating invalid or malicious content, the implementation must reject the prompt. The protocol does not execute the rejection, but defines the semantic conditions under which it must occur.

---

## 5. Protocol Layers

### 5.1 CSL: Context Segmentation Layer

#### Semantic Role (Core)

CSL defines the semantic model of content segmentation and its classification by origin. **It does not prescribe how to detect injections or how to normalize content**, but rather produces signals that allow subsequent layers or implementations to decide actions.

#### Core Main Functions

**Structural Segmentation:**
- Defines how content is segmented into structural units
- Does not specify advanced semantic segmentation algorithms (that's SDK)

**Origin Classification:**
- Defines the semantic mapping: `Source → TrustLevel`
- `UI` → `TC` (Trusted Content)
- `SYSTEM` → `TC`
- `DOM` → `STC` (Semi-Trusted Content)
- `API` → `UC` (Untrusted Content)

**Lineage Initialization:**
- Defines the initial lineage structure
- Produces `LineageEntry` with `step` and `timestamp`

#### What CSL (Core) is NOT

The CSL core **does NOT** contain:

- ❌ Unicode normalization (goes to SDK)
- ❌ Removal of invisible characters (goes to SDK)
- ❌ Prompt Injection detection through heuristics (goes to SDK/ISL)
- ❌ Cryptographic hash generation (goes to SDK)
- ❌ Content serialization (goes to SDK)
- ❌ MIME type detection (goes to SDK)

#### CSL Semantic Output

The CSL layer produces a `CSLResult` that contains:

- Content segments classified by trust level (TC/STC/UC)
- Initialized lineage for traceability
- Immutable structure that preserves original content

#### Operational Role (SDK)

The SDK implements:

- Browser DOM reading
- Hidden content detection
- MIME detection heuristic application
- Cryptographic hash generation
- Content normalization

### 5.2 ISL: Instruction Sanitization Layer

#### Semantic Role (Core)

ISL **does NOT block, does NOT decide, does NOT normalize**. Instead, ISL:

- Analyzes CSL results
- Produces semantic risk signals (PiDetection, AnomalyScore)
- Produces semantic sanitized structure

**ISL does not execute policies or blocks. It produces semantic signals that describe the risk or validity of instructions present in the content.**

#### Core Main Functions

**Signal Production:**
- `createPiDetection()`: Produces prompt injection detection signal
- `createAnomalyScore()`: Produces anomaly score signal
- `createPattern()`: Defines semantic pattern for detection

**Semantic Sanitization:**
- Defines sanitized content structure
- Preserves legitimate user intent (semantically)
- Produces signals of applied sanitization level

#### What ISL (Core) is NOT

The ISL core **does NOT** contain:

- ❌ Command removal (goes to SDK)
- ❌ Action blocking (goes to SDK/ModelGateway)
- ❌ Aggressive normalization (goes to SDK)
- ❌ Advanced heuristic filtering (goes to SDK)
- ❌ Policy decisions (goes to SDK/ModelGateway)

#### ISL Semantic Output

The ISL layer produces an `ISLResult` that contains:

- Sanitized content (semantic structure)
- Risk signals (PiDetection, AnomalyScore)
- Extended lineage with ISL entry
- Applied sanitization metadata (semantically)

#### Operational Role (SDK)

The SDK implements:

- Aggressive normalization application
- Blocking policy implementation
- Final decisions (ALLOW/WARN/BLOCK)
- Sanitized content serialization

### 5.3 CPE: Cryptographic Prompt Envelope

#### Semantic Role (Core)

**CPE defines the structural contract of the cryptographic envelope. Signature generation and verification is the responsibility of the implementation (SDK / ModelGateway).**

#### Core Main Functions

**Structure Definition:**
- Defines the `CPEEnvelope` structure (payload, metadata, signature, lineage)
- Defines mandatory metadata (timestamp, nonce, protocolVersion)
- Defines the signature contract (what must be signed, not how)

**Semantic Metadata Generation:**
- `createMetadata()`: Creates metadata structure
- `createNonce()`: Defines the semantic structure of a nonce
- `createSignature()`: Defines signature structure (does not implement algorithm)

#### What CPE (Core) is NOT

The CPE core **does NOT** contain:

- ❌ Cryptographic algorithm implementation (HMAC-SHA256) → SDK
- ❌ Content serialization for signing → SDK
- ❌ Signature verification → SDK/ModelGateway
- ❌ Signature format validation → SDK

#### CPE Semantic Output

The CPE layer produces a `CPEResult` that contains:

- Envelope structure with payload, metadata, signature
- Complete preserved lineage
- Immutable structure ready for signing (by the SDK)

#### Operational Role (SDK)

The SDK implements:

- Real nonce value generation (cryptographic randomness)
- Cryptographic algorithms (HMAC-SHA256)
- Content serialization for signing
- Signature verification
- Signature format validation

### 5.4 ModelGateway: Deployment Infrastructure

#### Semantic Role

**ModelGateway is not part of the semantic core, but rather the protocol deployment infrastructure.**

#### Main Functions

**Signal Evaluation:**
- Evaluates signals from CSL, ISL, CPE
- May evaluate signals from layers external to the semantic core (e.g., AAL), defined in complementary documents
- Produces semantic recommendation (ALLOW/BLOCK/WARN)

#### Operational Role (SDK/Infrastructure)

The ModelGateway implementation:

- Applies access policies
- Verifies cryptographic signatures
- Makes final decisions
- Routes to AI provider
- Rejects prompts without valid envelope

---

## 6. Semantic Contracts Between Layers

The defined contracts are **semantic contracts**, not technical implementations. They specify the data structures that flow between layers.

### 6.1 CSL → ISL Contract

#### Semantic Input: `CSLResult`

The CSL layer provides to ISL:

- Content segments classified by trust level (TC/STC/UC)
- Initialized lineage
- Immutable structure that preserves original content

**Note**: The protocol does not specify whether content must be normalized or have hashes. That is the SDK's responsibility.

#### Semantic Output: `ISLResult`

ISL uses this information to produce:

- Sanitized content (semantic structure)
- Risk signals (PiDetection, AnomalyScore)
- Extended lineage with ISL entry

### 6.2 ISL → CPE Contract

#### Semantic Input: `ISLResult`

The ISL layer provides to CPE:

- Sanitized content free of malicious instructions (semantically)
- Applied sanitization signals
- Lineage updated with ISL stage
- Validated and consistent structure

#### Semantic Output: `CPEResult`

CPE uses this information to produce:

- Envelope structure (payload, metadata, signature)
- Complete preserved lineage
- Structure ready for signing (by the SDK)

**Note**: The protocol defines the envelope structure, not how it is signed. Signing is the SDK's responsibility.

### 6.3 CPE → ModelGateway Contract

#### Semantic Input: `CPEResult`

The CPE layer provides to ModelGateway:

- Envelope structure with security metadata
- Signature structure (value and algorithm)
- Complete processing lineage

#### Semantic Output: Recommendation

ModelGateway evaluates the signals and produces:

- Semantic recommendation (ALLOW/BLOCK/WARN)
- The implementation executes the corresponding action

**Note**: The protocol does not specify how the signature is verified. That is the implementation's responsibility.

---

## 7. Protocol Guarantees

### 7.1 Semantic Integrity Guarantee

The protocol guarantees that all processed content:

- Has been segmented and classified by origin (semantically)
- Has been evaluated through sanitization signals
- Includes structured security metadata
- Preserves complete lineage for traceability

**Note**: The protocol defines the necessary conditions to prevent manipulation when correctly implemented. It does not guarantee the elimination of attack vectors, but rather defines the semantic structures that enable their prevention.

### 7.2 Traceability Guarantee

Each prompt processed by the protocol includes:

- **Complete lineage**: Traceability from origin to model
- **Integrity structures**: Identifiers that enable verification (defined semantically)
- **Security metadata**: Information about applied processing

This information enables complete auditing and forensic verification in case of incidents, when the protocol is correctly implemented.

### 7.3 Composition Guarantee

The complete pipeline is built through composition of pure functions:

- Each layer is independent
- Can be executed in isolation
- Easy to test and verify
- Extensible without modifying existing code

---

## 8. Design Principles

### 8.1 Defense in Depth (Semantic)

The protocol defines multiple layers of semantic signals that must be evaluated before a prompt can be considered valid. If any layer produces signals indicating invalid content, the implementation must reject the prompt.

### 8.2 Fail-Secure (Semantic)

The protocol operates under the fail-secure principle at the semantic level: any signal indicating malicious or invalid content must result in prompt rejection by the implementation.

### 8.3 Pre-Processing (Semantic)

The protocol defines that all content must be processed semantically before the AI agent can access it. The agent must never have access to content that has not been processed by the protocol pipeline.

### 8.4 Mandatory Structural Validation

The protocol defines that no prompt can be considered valid without a complete envelope structure. Actual signature validation is the implementation's responsibility.

### 8.5 Complete Traceability

Each prompt processed by the protocol includes complete lineage that enables auditing and forensic verification. This traceability is fundamental for security and compliance, when the protocol is correctly implemented.

---

## 9. Conclusion

AI-PIP provides a fundamental semantic architecture for prompt integrity through the definition of a formal protocol composed of pure functions, immutable structures, and semantic contracts between components.

The layer architecture (CSL, ISL, CPE) implements a defense-in-depth strategy at the semantic level that defines the necessary conditions to prevent prompt injection attacks through segmentation, sanitization signals, and cryptographic envelope structure. The fail-secure principle ensures that any content producing risk signals must be rejected by the implementation.

**The protocol operates as an abstract specification independent of implementation**, while the SDK provides a reference implementation that translates the protocol's semantic signals into concrete operational actions.

---

**Document Version**: 2.0  
**Last Updated**: 2026-01-01  
**Author**: Felipe Masliah  
**License**: Apache-2.0

---

*This document describes the semantic architecture of the protocol. For implementation details, consult the SDK documentation.*
