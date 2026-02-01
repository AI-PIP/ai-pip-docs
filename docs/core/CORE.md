# AI-PIP Semantic Core

## Overview

The **AI-PIP Semantic Core** defines the formal, implementation-agnostic foundation of the AI Prompt Integrity Protocol.

The core specifies **what the protocol does**, not how it is implemented. It is composed exclusively of semantic definitions, immutable data structures, pure transformations, and formal guarantees. Any valid implementation of AI-PIP must conform to the contracts defined in this core.

The semantic core is independent of programming language, runtime, platform, or SDK.

---

## Core Principles

The AI-PIP core is governed by the following non-negotiable principles.

### 1. Protocol, Not Software

The core defines a **protocol specification**, not an application or library.  
It describes **semantic behavior**, not operational behavior.

Implementations (SDKs, gateways, agents) consume the core; they do not redefine it.

---

### 2. Pure Semantic Transformations

All core operations are defined as **pure transformations**:

- Same input → same output  
- No side effects  
- No hidden state  
- No dependency on external systems  

This guarantees determinism, reproducibility, and formal verifiability.

---

### 3. Immutability

All structures produced by the core are **immutable**.

Transformations always produce **new semantic states**.  
No layer is allowed to mutate data produced by a previous layer.

This ensures:
- Traceability  
- Referential transparency  
- Safe composition  
- Thread safety by design  

---

### 4. Signals, Not Actions

The semantic core **never executes actions**.

It produces:
- Signals  
- Classifications  
- Risk indicators  
- Recommendations  

Decisions such as *blocking*, *warning*, *logging*, or *enforcing* are explicitly outside the core and belong to implementations.

---

### 5. Composition by Layers

The protocol is composed as a pipeline of semantic layers, each one transforming the output of the previous layer.

Input
↓
CSL → Context Segmentation
↓
ISL → Instruction Sanitization
↓
CPE → Cryptographic Prompt Envelope
↓
Semantic Signals

Each layer has a single, well-defined responsibility.

---

## Role of the Semantic Core

The semantic core is responsible for:

- Defining **semantic meaning** of input content  
- Preserving **origin and trust classification**  
- Producing **sanitization and anomaly signals**  
- Defining **cryptographic envelope structure**  
- Maintaining **complete lineage** across transformations  

The core is **not responsible** for:

- Enforcing policies  
- Blocking execution  
- Interacting with models  
- Performing runtime I/O  
- Managing keys or secrets operationally  

---

## Core Components

The semantic core is defined by four main elements.

### 1. Semantic Data Structures

The core defines a set of **value objects** that represent the protocol state at each step.

Examples include:
- Segments  
- Trust classifications  
- Origins and sources  
- Sanitization signals  
- Anomaly indicators  
- Lineage entries  
- Envelope metadata  

These structures:
- Are immutable  
- Have no identity beyond their value  
- Are serializable in a deterministic way  
- Do not contain behavior  

---

### 2. Semantic Transformations

Each layer is defined as a transformation from one semantic state to another.

Conceptually:

- CSL transforms raw input into classified segments  
- ISL transforms classified segments into sanitized semantic content  
- CPE transforms sanitized content into a cryptographic envelope definition  

These transformations:
- Are deterministic  
- Do not inspect runtime environment  
- Do not perform cryptographic operations directly  
- Do not depend on external services  

---

### 3. Semantic Contracts

Between layers, the core defines **formal contracts**.

A contract specifies:
- Required input properties  
- Guaranteed output properties  
- Invariants that must hold  
- Information that must be preserved  

Implementations must not:
- Skip contracts  
- Alter semantic meaning  
- Inject additional state into core results  

---

### 4. Lineage System

The core mandates a **complete lineage trail**.

Each transformation:
- Appends a new lineage entry  
- Preserves previous entries  
- Never rewrites history  

The lineage guarantees:
- Full traceability  
- Auditable transformations  
- Deterministic reconstruction of processing steps  

Lineage is semantic, not operational.  
It does not encode logs or runtime metrics.

---

## Layer Documentation

For detailed documentation of each layer:

- **[CSL (Context Segmentation Layer)](./layers/CSL.md)** — Segmentation and classification by trust level
- **[ISL (Instruction Sanitization Layer)](./layers/ISL.md)** — Sanitization, prompt-injection detection, and ISLSignal for downstream layers
- **[CPE (Cryptographic Prompt Envelope)](./layers/CPE.md)** — Envelope structure and metadata
- **[AAL (Agent Action Lock)](./layers/AAL.md)** — Policy-based decisions (ALLOW/WARN/BLOCK) and removal plans from ISLSignal
- **[Shared](./shared/shared.md)** — Shared utilities and lineage management

---

## Determinism Guarantees

The AI-PIP core guarantees:

- **Deterministic output** for identical inputs  
- **Order-independent reasoning**  
- **Reproducibility across environments**  
- **Formal testability**  

This enables:
- Offline execution  
- Formal verification  
- Cross-language implementations  
- Long-term protocol stability  

---

## Separation from SDKs

The semantic core intentionally excludes:

- Cryptographic implementations  
- Serialization formats  
- Policy decisions  
- Runtime enforcement  
- Environment-specific behavior  

SDKs exist to **consume and interpret** the core, not redefine it.

The relationship is strictly one-directional:

Semantic Core → SDK → Runtime / Environment

Never the inverse.

---

## Core Stability

The semantic core is designed to be:

- Stable over time  
- Backward-compatible where possible  
- Versioned explicitly when semantics change  

Changes to the core represent **protocol evolution**, not refactoring.

---

## Summary

The AI-PIP Semantic Core defines the formal meaning of the protocol.

It provides:
- A deterministic semantic pipeline  
- Immutable protocol states  
- Clear separation of concerns  
- A foundation for interoperable implementations  

Any system claiming to implement AI-PIP **must conform to the semantics defined in this core**, regardless of language, platform, or runtime environment.


## Reference Implementation

The semantic core has a reference implementation available as an NPM package:

**Package**: [`@ai-pip/core`](https://www.npmjs.com/package/@ai-pip/core)  
**Repository**: [ai-pip-core](https://github.com/AI-PIP/ai-pip-core)

The reference implementation demonstrates how the semantic core can be realized in TypeScript/JavaScript while maintaining full adherence to the protocol semantics defined here.