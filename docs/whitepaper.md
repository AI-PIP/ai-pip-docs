# AI Prompt Integrity Protocol (AI-PIP)
## A Semantic Protocol for Secure AI Prompt Processing

**Version**: 2.0  
**Author**: Felipe Masliah  
**License**: Apache-2.0  
**Language**: English  
**Status**: Formal Semantic Protocol  
**Last Updated**: 2026-01-01

---

## 📋 Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Introduction](#2-introduction)
3. [Context and Problem Statement](#3-context-and-problem-statement)
4. [State of the Art](#4-state-of-the-art)
5. [AI-PIP Protocol Definition](#5-ai-pip-protocol-definition)
   - [5.1 Semantic Protocol vs Implementation](#51-semantic-protocol-vs-implementation)
   - [5.2 Semantic Core](#52-semantic-core)
   - [5.3 Reference SDK](#53-reference-sdk)
6. [Protocol Semantic Architecture](#6-protocol-semantic-architecture)
   - [6.1 Fundamental Principles](#61-fundamental-principles)
   - [6.2 Protocol Layers](#62-protocol-layers)
   - [6.3 Semantic Contracts Between Layers](#63-semantic-contracts-between-layers)
7. [Semantic Processing Flow](#7-semantic-processing-flow)
8. [Protocol Guarantees](#8-protocol-guarantees)
9. [Advantages and Protocol Properties](#9-advantages-and-protocol-properties)
10. [Limitations and Future Work](#10-limitations-and-future-work)
11. [Conclusion](#11-conclusion)
12. [References](#12-references)

---

## 1. Executive Summary

**AI-PIP (AI Prompt Integrity Protocol)** is a formal semantic protocol designed to ensure the integrity, authenticity, and traceability of prompts before their processing by language models. Unlike implementation-based solutions, AI-PIP is defined as an **abstract protocol** independent of its concrete implementation.

### Fundamental Principles

1. **AI-PIP is a semantic protocol, not software**: The protocol defines semantic structures, risk signals, and contracts between layers. Software is only one possible implementation.

2. **Semantic Core / SDK Separation**: 
   - **Semantic Core**: Pure, deterministic functions without side effects that define **what** the protocol does
   - **SDK**: Reference implementation that defines **how** to use the protocol in concrete environments

3. **The protocol produces signals, it does not execute actions**: The semantic core emits risk signals, recommendations, and data structures. Blocking, validation, and execution decisions belong to the implementation (SDK).

### Main Objective

The main objective of the AI-PIP protocol is to **protect intelligent browsers** from new cybersecurity attacks through prompt injections, jailbreaks, and other malicious techniques, preventing that malicious information from being processed by language models (LLM).

To achieve this objective, AI-PIP implements a processing pipeline that **segments, classifies, sanitizes, and encapsulates** content before its processing by language models. This process guarantees the **integrity and security of data**, providing a semantic protection barrier that prevents the execution of malicious or manipulated instructions in the browser context.

---

## 2. Introduction

The integration of artificial intelligence agents in web browsers has transformed human-computer interaction, but has introduced critical vulnerabilities related to prompt integrity. Language models process content without native capability to distinguish between legitimate user instructions and manipulated or malicious content injected into the context.

**AI-PIP** addresses this need by defining a formal semantic protocol that establishes the conditions under which a prompt can be considered valid for processing. The protocol is not specific software, but an abstract specification that can be implemented in different environments (browsers, servers, edge computing, artificial intelligence agents).

### Key Features

- **Formal Semantic Protocol**: Defines data structures, pure functions, and contracts between components
- **Implementation Independent**: The protocol can execute offline, without specific runtime dependencies
- **Functional Composition**: Each protocol layer is a pure function that transforms immutable data
- **Complete Traceability**: The protocol preserves a complete lineage of all applied transformations

---

## 3. Context and Problem Statement

### 3.1 Identified Problems

Interaction with language models in browsers presents specific vulnerabilities:

1. **Hidden Text in Web Pages**: Invisible or manipulated content that can be injected into prompts without user knowledge
2. **Prompt Manipulation**: Malicious modification of instructions during transmission or processing
3. **Lack of Traceability**: Absence of standard mechanisms to verify the origin and authenticity of processed content
4. **Semantic Vulnerabilities**: Attacks based on social engineering and jailbreaks that exploit model interpretation

### 3.2 Need for a Formal Protocol

These problems require a systematic approach that guarantees prompt integrity from its origin to its processing. However, current solutions focus on specific implementations without defining a reusable abstract protocol.

**AI-PIP** proposes the first formal specification of a prompt integrity protocol, independent of its concrete implementation.

---

## 4. State of the Art

The state of the art in security applied to interaction with language models within the browser **lacks a formal protocol** prior to prompt processing. Current research is divided into four main approaches:

### 4.1 Heuristic Filtering

Based on blocked word lists or regular expressions.

**Limitations**: Easy to evade, does not consider semantics, not a formal protocol.

### 4.2 Self-Guarding with LLM

The model evaluates its own prompt ("Is this instruction malicious?").

**Limitations**: Vulnerable to jailbreaks and role hijacking, requires model access.

### 4.3 Partial Context Sandboxing

Some extensions attempt to separate visible from hidden text.

**Limitations**: No cryptographic guarantees, no origin authentication, specific implementation.

### 4.4 Isolation through AI Proxies

Companies implement private middleware for data cleaning.

**Limitations**: Not a standard, not interoperable, usually proprietary.

### 4.5 State of the Art Conclusion

> **Today there is no standardized mechanism that verifies integrity, authenticity, and semantics of text BEFORE reaching the AI model through a formal protocol independent of implementation.**

This validates the need for the AI-PIP protocol as the **first open standard** for prompt integrity defined as a formal semantic protocol.

---

## 5. AI-PIP Protocol Definition

### 5.1 Semantic Protocol vs Implementation

**AI-PIP is not software, it is a semantic protocol. Software is only one possible implementation.**

The AI-PIP protocol is defined through:

- **Semantic Structures**: Immutable data types that represent processing state
- **Pure Functions**: Deterministic transformations without side effects
- **Inter-Layer Contracts**: Formal specifications of interfaces between components
- **Composition Rules**: How layers combine to form the complete pipeline

The SDK (Software Development Kit) implements:

- **Blocks and Policies**: Operational decisions based on protocol signals
- **Real Cryptography**: Concrete implementation of signature and verification algorithms
- **Serialization**: Specific format for transmission and storage
- **Environment Integration**: Adapters for browsers, Node.js, edge computing

### 5.2 Semantic Core

The AI-PIP semantic core is composed exclusively of:

#### Pure Functions

Functions that, given the same input, always produce the same output and have no side effects:

- `segment(input: CSLInput): CSLResult` - Segments and classifies content
- `sanitize(cslResult: CSLResult): ISLResult` - Produces sanitization signals
- `envelope(islResult: ISLResult, secretKey: string): CPEResult` - Defines envelope structure

**Shared Features**:
- `addLineageEntry(lineage, entry): LineageEntry[]` - Adds entry to global lineage
- `addLineageEntries(lineage, entries): LineageEntry[]` - Adds multiple entries
- `filterLineageByStep(lineage, step): LineageEntry[]` - Filters lineage by step
- `getLastLineageEntry(lineage): LineageEntry | undefined` - Gets last entry

**Note**: Shared contains fundamental protocol features used across multiple layers. Lineage is global and incremental between layers, enabling complete processing audit. It does not belong to a single layer (CSL, ISL, or CPE), but is shared and incremented throughout the entire pipeline.

#### Immutable Value Objects

Data types defined by their value, without identity:

- `TrustLevel` - Trust level (TC, STC, UC)
- `Origin` - Content origin (DOM, UI, SYSTEM, API)
- `LineageEntry` - Lineage entry (step, timestamp)
- `PiDetection` - Prompt injection detection
- `AnomalyScore` - Anomaly score
- `Nonce` - Unique value to prevent replay attacks
- `SignatureVO` - Cryptographic signature (value + algorithm)
- `CPEMetadata` - Envelope security metadata

#### Semantic Contracts

Formal specifications of interfaces between layers:

- **CSL → ISL**: `CSLResult` with classified segments
- **ISL → CPE**: `ISLResult` with sanitized content and signals
- **CPE → SDK**: `CPEResult` with structured envelope and semantic signals

**Shared Features**:
- Global and incremental lineage used by all layers for auditing
- Lineage increments throughout the entire pipeline (CSL → ISL → CPE)
- Not part of the sequential layer flow, but fundamental protocol features

**SDK Contracts** (outside semantic core):
- **CPE → ModelGateway**: SDK receives `CPEResult` and decides operational actions (ALLOW/BLOCK/WARN)
- **CPE → AAL**: SDK receives core signals and applies action locks according to policies

#### What is NOT Core

The semantic core **does NOT** contain:

- ❌ Functions that "decide" (shouldBlock, shouldWarn) → SDK/ModelGateway
- ❌ Functions that "detect risk" through advanced heuristics → SDK
- ❌ Functions that "aggressively normalize" → SDK/ISL
- ❌ Functions that "serialize" → SDK
- ❌ Functions that "verify" cryptographically → SDK
- ❌ "Audit" and analysis functions → SDK/tooling

### 5.3 Reference SDK

The SDK provides a reference implementation of the protocol that includes:

#### SDK Layers

- **AAL (Agent Action Lock)**: Implements action locks based on core signals
- **ModelGateway**: Evaluates core signals and decides final actions (ALLOW/BLOCK/WARN), routes to AI provider

#### Implementation Functions

- **Hash and Cryptography**: `hashContent()`, `verifyContentHash()`, `verifySignature()`
- **Detection**: `detectMimeType()`, `normalizeBasic()`, `segmentSemantic()`
- **Decisions**: `shouldBlock()`, `shouldWarn()`, access policies (ModelGateway)
- **Action Locks**: `lockNavigation()`, `preventAction()`, `applyActionLock()` (AAL)
- **Serialization**: `serializeContent()`, `serializeMetadata()`
- **Auditing**: `getLineageStats()`, `getLineageByNotes()`

#### Adapters

- `DOMAdapter` - Adapter for browser DOM
- `UIAdapter` - Adapter for user interfaces
- `CryptoHashGenerator` - Cryptographic hash generator
- `SystemTimestampProvider` - Timestamp provider
- `ModelProviderAdapter` - Adapter for AI providers (OpenAI, Anthropic, etc.)

#### Factory Functions

Functions that facilitate protocol use in concrete environments:

```typescript
const service = createCSLService({
  enablePolicyValidation: true,
  enableLineageTracking: true,
  hashAlgorithm: 'sha256'
})
```

---

## 6. Protocol Semantic Architecture

### 6.1 Fundamental Principles

#### Principle 1: Semantic Separation

The protocol defines **what** must occur; the SDK defines **how** it occurs.

#### Principle 2: Pure Functions

All core functions are pure, deterministic, and without side effects. The same input always produces the same output.

#### Principle 3: Immutability

All data structures are immutable. Transformations produce new structures, never modify existing ones.

#### Principle 4: Composition

The complete pipeline is built through composition of pure functions:

```
CSLResult = segment(CSLInput)
ISLResult = sanitize(CSLResult)
CPEResult = envelope(ISLResult, secretKey)
```

#### Principle 5: Signals, Not Actions

The protocol produces signals (TrustLevel, AnomalyScore, PiDetection), it does not execute actions (blocks, validations). Actions belong to the SDK.

### 6.2 Protocol Layers

The AI-PIP protocol consists of **three main semantic core layers** that process content sequentially:

**Semantic Core (implemented)**:
- CSL: Context Segmentation Layer
- ISL: Instruction Sanitization Layer
- CPE: Cryptographic Prompt Envelope

**Shared Features**:
- Fundamental protocol features used across multiple layers
- Global and incremental lineage for inter-layer auditing
- Not a protocol layer, but an essential part of the semantic core

**SDK (operational implementation)**:
- AAL: Agent Action Lock
- ModelGateway

#### 6.2.1 CSL: Context Segmentation Layer

**Semantic Role (Core)**:
- Segments content into structural units
- Classifies content origin (Source → TrustLevel)
- Initializes lineage for traceability
- Produces immutable data structures

**Operational Role (SDK)**:
- Reads browser DOM
- Detects hidden content
- Applies MIME detection heuristics
- Generates cryptographic hashes

**Core Functions**:
- `segment(input: CSLInput): CSLResult`
- `classifySource(source: Source): TrustLevel`
- `classifyOrigin(origin: Origin): TrustLevel`
- `initLineage(segment: CSLSegment): LineageEntry[]`

**Semantic Output**:
- Segments classified by trust level (TC, STC, UC)
- Initialized lineage
- Immutable `CSLResult` structure

#### 6.2.2 ISL: Instruction Sanitization Layer

**Semantic Role (Core)**:
- Produces sanitization signals according to TrustLevel
- Detects prompt injection patterns
- Calculates anomaly scores
- Emits risk signals (does not decide actions)

**Operational Role (SDK)**:
- Applies aggressive normalization
- Implements blocking policies
- Makes final decisions (ALLOW/WARN/BLOCK)
- Serializes sanitized content

**Core Functions**:
- `sanitize(cslResult: CSLResult): ISLResult`
- `createPiDetection(...): PiDetection`
- `createAnomalyScore(...): AnomalyScore`
- `createPattern(...): Pattern`

**Semantic Output**:
- Sanitized content with risk signals
- Prompt injection detections
- Anomaly scores
- Immutable `ISLResult` structure

#### 6.2.3 CPE: Cryptographic Prompt Envelope

**Semantic Role (Core)**:
- Defines cryptographic envelope structure
- Generates security metadata (timestamp, nonce, version)
- Establishes signature contract (does not implement signature)
- Preserves complete lineage

**Operational Role (SDK)**:
- Implements cryptographic algorithms (HMAC-SHA256)
- Serializes content for signing
- Verifies signatures
- Validates signature formats

**Core Functions**:
- `envelope(islResult: ISLResult, secretKey: string): CPEResult`
- `createMetadata(...): CPEMetadata`
- `createNonce(...): Nonce`
- `createSignature(...): SignatureVO`

**Semantic Output**:
- Envelope structure with payload, metadata, signature
- Complete preserved lineage
- Immutable `CPEResult` structure

### 6.3 Shared Features

**Note**: Shared is not a protocol layer, but contains fundamental features used across multiple layers.

**Global and Incremental Lineage**:
Lineage is a fundamental part of the AI-PIP protocol for inter-layer auditing. It does not belong to a single layer (CSL, ISL, or CPE), but is **global and incremental** throughout the entire processing pipeline.

**Shared Features**:
- `addLineageEntry(lineage, entry): LineageEntry[]` - Adds an entry to global lineage
- `addLineageEntries(lineage, entries): LineageEntry[]` - Adds multiple entries to lineage
- `filterLineageByStep(lineage, step): LineageEntry[]` - Filters entries by step for auditing
- `getLastLineageEntry(lineage): LineageEntry | undefined` - Gets the last lineage entry

**Purpose**:
- **Global Lineage**: Lineage increments across all layers (CSL → ISL → CPE)
- **Complete Auditing**: Enables tracking all processing from origin to final envelope
- **Traceability**: Each layer adds its entry to lineage, creating a complete and immutable history
- **Pure Functions**: Without side effects, guaranteeing determinism

**Lineage Characteristics**:
- **Incremental**: Each layer adds its entry to existing lineage
- **Global**: Does not belong to a single layer, is shared across all
- **Immutable**: Functions return new arrays, never modify original lineage
- **Auditable**: Enables complete auditing of prompt processing

**Note**: Shared functions are available from the main entry point `@ai-pip/core`, not as a specific subpath.

### 6.4 SDK Components

The following layers belong to the SDK (operational implementation), not the semantic core, as they require operational decisions and side effects:

#### 6.3.1 AAL: Agent Action Lock

**Role in SDK**:
- Implements action locks based on core signals
- Blocks navigation when necessary
- Prevents reading of risky content
- Applies operational security policies

**Status**: In design

**Reason for being in SDK**: AAL requires side effects (blocking navigation, preventing actions) and operational decisions that do not belong to pure semantic core.

#### 6.3.2 ModelGateway

**Role in SDK**:
- Evaluates signals from CSL, ISL, CPE, Shared
- Applies access policies based on signals
- Verifies cryptographic signatures
- Makes final decisions (ALLOW/BLOCK/WARN)
- Routes to AI provider

**Status**: In design

**Reason for being in SDK**: ModelGateway requires final operational decisions, routing to external providers, and side effects that do not belong to pure semantic core.

### 6.5 Semantic Contracts Between Layers

The defined contracts are **semantic contracts**, not technical implementations. They specify the data structures that flow between layers:

#### CSL → ISL Contract

**Input**: `CSLResult`
- Segments classified by TrustLevel
- Initialized lineage
- Immutable structure

**Output**: `ISLResult`
- Sanitized content
- Risk signals (PiDetection, AnomalyScore)
- Extended lineage

#### ISL → CPE Contract

**Input**: `ISLResult`
- Sanitized content
- Applied sanitization signals
- Complete lineage

**Output**: `CPEResult`
- Envelope with payload, metadata, signature
- Preserved lineage

#### CPE → SDK Contract

**Input**: `CPEResult`
- Structured cryptographic envelope
- Security metadata
- Complete lineage
- Semantic signals (TrustLevel, AnomalyScore, PiDetection)

**SDK Output**: 
- Operational decisions (ALLOW/BLOCK/WARN) - ModelGateway
- Applied action locks - AAL
- Routing to AI provider - ModelGateway

---

## 7. Semantic Processing Flow

The protocol processes content through a pure functional pipeline:

```
┌─────────────────────────────────────────────────────────┐
│                    SEMANTIC CORE                        │
│         (Pure Functions, No Side Effects)              │
│                                                          │
│  CSLInput                                               │
│    │                                                     │
│    ▼                                                     │
│  segment() → CSLResult                                  │
│    │                                                     │
│    ▼                                                     │
│  sanitize() → ISLResult                                 │
│    │                                                     │
│    ▼                                                     │
│  envelope() → CPEResult                                 │
│    │                                                     │
│    ▼                                                     │
│  CPEResult with Semantic Signals                        │
│  (TrustLevel, AnomalyScore, PiDetection)                │
│                                                          │
│  Global Lineage (Shared): Incremental across            │
│  all layers (CSL → ISL → CPE) for auditing              │
└───────────────────┬─────────────────────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────────────────────┐
│                    SDK / IMPLEMENTATION                 │
│         (Operational Decisions, Side Effects)          │
│                                                          │
│  ModelGateway:                                          │
│  - Evaluates core signals                              │
│  - Verifies cryptographic signatures                   │
│  - Decides actions (ALLOW/BLOCK/WARN)                  │
│  - Routes to AI provider                                │
│                                                          │
│  AAL (Agent Action Lock):                               │
│  - Applies action locks                                │
│  - Blocks navigation if necessary                      │
│  - Prevents reading of risky content                   │
│                                                          │
│  Utilities:                                             │
│  - Reads DOM                                            │
│  - Generates hashes                                     │
│  - Serializes content                                   │
│  - Auditing and analysis                                │
└─────────────────────────────────────────────────────────┘
```

### Flow Characteristics

1. **Deterministic**: Same input always produces same output
2. **Composable**: Each layer is a pure function that can execute independently
3. **Offline**: The semantic pipeline can execute without internet connection
4. **Testable**: Each function can be tested in isolation

---

## 8. Protocol Guarantees

### 8.1 Semantic Integrity Guarantee

The protocol guarantees that all processed content:

- Has been segmented and classified by origin
- Has been evaluated through sanitization signals
- Includes structured security metadata
- Preserves complete lineage for traceability

### 8.2 Determinism Guarantee

All core functions are deterministic:

- Same input → same output
- No external state dependencies
- No side effects
- Executable in any order

### 8.3 Immutability Guarantee

All data structures are immutable:

- Not modified after creation
- Transformations produce new structures
- Thread-safe by design
- Predictable and safe

### 8.4 Composition Guarantee

The complete pipeline is built through composition:

- Each layer is independent
- Can execute in isolation
- Easy to test and verify
- Extensible without modifying existing code

---

## 9. Advantages and Protocol Properties

### 9.1 Technical Properties

| Property | Description |
|----------|-------------|
| **Provider Independence** | The protocol works with any AI provider (OpenAI, Google, Anthropic, Local LLM) |
| **Structure-Based Prevention** | Does not depend solely on heuristics, but on semantic classification |
| **Cryptographic Security** | Defines structures that guarantee message integrity |
| **Low Overhead** | Processing in milliseconds, executable offline |
| **Extensibility** | Modular architecture allows adding new layers |

### 9.2 Academic Advantages

1. **Formal Protocol**: Precise mathematical definition through pure functions
2. **Verifiable**: Each component can be formally verified
3. **Composable**: Functional architecture enables compositional reasoning
4. **Implementation Independent**: The protocol can be implemented in different environments (browsers, servers, edge computing, artificial intelligence agents)

### 9.3 Practical Advantages

1. **Reusable**: The semantic core can be used in different contexts
2. **Testable**: Pure functions facilitate exhaustive testing
3. **Maintainable**: Clear separation between protocol and implementation
4. **Scalable**: Modular architecture allows incremental growth

---

## 10. Limitations and Future Work

### 10.1 Current Limitations

1. **Social Engineering Detection**: The protocol does not detect 100% of social engineering-based attacks
2. **Industry Adoption**: The protocol requires adoption to become a standard
3. **Gateway Compatibility**: CPE requires explicit compatibility in AI gateways

### 10.2 Future Work

#### Planned Layers

- **SRL (Semantic Review Layer)**: Advanced semantic review
- **SPL (Secure Policy Layer)**: Security policy layer

#### Protocol Improvements

- Extension of semantic contracts
- New risk signal types
- Traceability improvements

For detailed information about the roadmap, see: [Roadmap](./roadmap.md)

---

## 11. Conclusion

AI-PIP proposes the **first formal semantic protocol** for prompt security in browsers, defined as an abstract specification independent of its implementation.

### Main Contributions

1. **Formal Semantic Protocol**: First abstract specification of prompt integrity
2. **Core/SDK Separation**: Clear distinction between protocol and implementation
3. **Functional Architecture**: Pipeline composed of pure and deterministic functions
4. **Complete Traceability**: Lineage preserved in all transformations

### Expected Impact

This work opens the path for:

- ✅ Secure AI-assisted navigation
- ✅ Compatibility with agent ecosystem
- ✅ Academic and professional framework for LLM security
- ✅ Open and interoperable standard

### Key Phrase

> **AI-PIP validates and protects every interaction before it reaches the AI model, functioning as a browser extension, integrated in agents, or as independent security layers.**

---

## 12. References

### Protocol Documentation

- [Semantic Architecture](./architecture.md) - Detailed protocol architecture
- [Layer Specifications](./core/layers/) - Documentation for each layer:
  - [CSL (Context Segmentation Layer)](./core/layers/csl.md)
  - [ISL (Instruction Sanitization Layer)](./core/layers/isl.md)
  - [CPE (Cryptographic Prompt Envelope)](./core/layers/cpe.md)
  - [Shared (Shared Functions)](./core/shared/shared.md)
- [SDK](./sdk/SDK.md) - SDK implementation guide
- [SDK Reference](./sdk/sdk-reference.md) - Complete SDK reference guide
  

### Implementation

- [Core Repository](https://github.com/AI-PIP/ai-pip-core) - Semantic core implementation
- [SDK Repository](https://github.com/AI-PIP/ai-pip-sdk-ts) - TypeScript reference SDK

### Roadmap

- [Roadmap](./roadmap.md) - Development plan and future improvements

---

**Document Version**: 2.0  
**Last Updated**: 2026-01-01  
**Author**: Felipe Masliah  
**License**: Apache-2.0

---

*This document defines the AI-PIP protocol as a formal semantic specification. For concrete implementations, consult the SDK documentation.*

