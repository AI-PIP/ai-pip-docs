# AI-PIP SDK - Implementation Guide

> **Reference Software Development Kit (SDK) for the AI-PIP protocol**
> 
> The SDK is an integration layer that consumes the semantic core (`@ai-pip/core`) and translates its semantic results into operational actions according to the execution environment.

**Version**: 2.0  
**Author**: Felipe Masliah  
**Last Updated**: 2026-01-01

---

## 📋 Table of Contents

1. [Introduction](#1-introduction)
2. [SDK vs Core Architecture](#2-sdk-vs-core-architecture)
3. [Installation](#3-installation)
4. [Main API](#4-main-api)
5. [SDK Functions](#5-sdk-functions)
6. [TypeScript Types](#6-typescript-types)
7. [Usage Examples](#7-usage-examples)
8. [Integration](#8-integration)
9. [Configuration](#9-configuration)
10. [Events and Callbacks](#10-events-and-callbacks)

---

## 1. Introduction

### 1.1 What is the SDK?

The AI-PIP SDK **does NOT define the protocol's security**. Instead, it acts as:

- **Adapter**: Translates between the semantic core and concrete environments (browser, Node.js, edge)
- **Orchestrator**: Coordinates the core's pure functions and applies operational policies
- **Integrator**: Provides helpers and utilities to facilitate protocol usage

### 1.2 Relationship with Core

```
┌─────────────────────────────────────┐
│         SDK (Implementation)        │
│  - Adapters (DOM, Node, Edge)        │
│  - Operational policies             │
│  - Serialization                     │
│  - Cryptographic verification       │
│  - Runtime decisions                │
└──────────────┬──────────────────────┘
               │ consumes
               ▼
┌─────────────────────────────────────┐
│   Semantic Core (@ai-pip/core)      │
│  - Pure functions                   │
│  - Immutable value objects           │
│  - Semantic contracts               │
│  - Risk signals                      │
└─────────────────────────────────────┘
```

**Key principle**: The SDK orchestrates and adapts the core; the core defines what the protocol does.

### 1.3 SDK Objective

The SDK enables:

- ✅ Integrating AI-PIP into JavaScript/TypeScript projects
- ✅ Using protocol layers (CSL, ISL, CPE) in a practical way
- ✅ Adapting the core to specific environments (browser, server, edge)
- ✅ Applying operational policies based on core signals
- ✅ Receiving notifications when the SDK interprets risks

**The SDK does NOT**:
- ❌ Define security logic (that's the core)
- ❌ Decide what is safe (interprets core signals)
- ❌ Is the protocol (it's a reference implementation)

### 1.4 Main Features

- ✅ **Native TypeScript**: Complete types and autocomplete
- ✅ **Modular**: Use only the layers you need
- ✅ **Optimized for low overhead**: Performance dependent on execution environment
- ✅ **Extensible**: Easy to extend with custom functionality
- ✅ **Cross-platform**: Works in Node.js, browsers, and edge environments

---

## 2. SDK vs Core Architecture

### 2.1 Separation of Responsibilities

| Aspect | Semantic Core | SDK |
|--------|--------------|-----|
| **Purpose** | Defines **what** the protocol does | Defines **how** to use the protocol |
| **State** | Stateless | May have state |
| **Dependencies** | Pure functions only | May use external libraries |
| **Decision** | Produces signals | Makes operational decisions |
| **Serialization** | Defines structure | Implements format |
| **Verification** | Defines contract | Implements validation |
| **Events** | Does not emit events | Interprets results and generates events |

### 2.2 Workflow

```
1. SDK receives input from environment (DOM, prompt, etc.)
   ↓
2. SDK adapts input to core format (CSLInput, etc.)
   ↓
3. SDK invokes core pure functions (segment, sanitize, envelope)
   ↓
4. Core produces semantic signals (TrustLevel, AnomalyScore, etc.)
   ↓
5. SDK interprets signals according to configured policies
   ↓
6. SDK executes operational actions (blocking, logging, events)
```

### 2.3 Conceptual Example

```typescript
// Core: Pure function that produces signals
const cslResult = segment({ content: '...', source: 'UI' })
// cslResult contains: TrustLevel, LineageEntry, etc.

// SDK: Interprets signals and executes actions
if (isUntrusted(cslResult.segments[0].trust)) {
  // This decision is from the SDK, not the core
  lock('navigation', 'Untrusted content detected')
  emitRiskEvent({ level: 'high', reason: 'UC detected' })
}
```

---

## 3. Installation

### 3.1 Required Packages

The SDK depends on the semantic core:

```bash
# Install both packages
pnpm add @ai-pip/sdk @ai-pip/core

# Or with npm
npm install @ai-pip/sdk @ai-pip/core

# Or with yarn
yarn add @ai-pip/sdk @ai-pip/core
```

**Note**: The SDK internally depends on `@ai-pip/core`. Although technically only the SDK can be installed (which will include the core as a dependency), it's recommended to install both explicitly for better control.

### 3.2 Import

```typescript
// Complete SDK import
import * as pip from '@ai-pip/sdk'

// Selective import of SDK functions
import { scanDOM, scanPrompt, purify, report, lock, onRiskDetected } from '@ai-pip/sdk'

// SDK type imports
import type { RiskEvent, SDKOptions, LockResult } from '@ai-pip/sdk'

// Direct core import (optional)
import { segment, sanitize, envelope } from '@ai-pip/core'
import type { CSLResult, ISLResult, CPEResult } from '@ai-pip/core'
```

---

## 4. Main API

### 4.1 API Structure

The SDK exposes a main API through the `pip` object that acts as a core adapter:

```typescript
interface AIPIP {
    // Adapter functions (invoke the core)
    scanDOM(element?: HTMLElement | Document): Promise<CSLResult>
    scanPrompt(prompt: string): Promise<ISLResult>
    purify(content: string | CSLResult | ISLResult): Promise<CPEResult>
    
    // Operational functions (interpret core signals)
    report(result: CSLResult | ISLResult | CPEResult): void
    lock(action: string, reason?: string): boolean
    
    // Events (generated by SDK, not by core)
    onRiskDetected(callback: (risk: RiskEvent) => void): void
    offRiskDetected(callback: (risk: RiskEvent) => void): void
    
    // Configuration
    configure(options: SDKOptions): void
    getConfig(): SDKOptions
    
    // Utilities
    version: string
}
```

### 4.2 Adapter Functions

These functions act as adapters that invoke AI-PIP core pure functions and apply actions according to the environment:

#### `scanDOM(element?: HTMLElement | Document): Promise<CSLResult>`

**Description**: Adapts the browser DOM to the core format and executes `segment()`.

**Internal flow**:
1. Reads the DOM (or specified element)
2. Detects MIME type of content
3. Generates content hash
4. Adapts to `CSLInput`
5. Invokes core `segment()`
6. Returns core `CSLResult`

**Example**:
```typescript
const result = await pip.scanDOM(document.body)
// result is a CSLResult from the core, with TrustLevel, LineageEntry, etc.
```

#### `scanPrompt(prompt: string): Promise<ISLResult>`

**Description**: Adapts a text prompt to the core format and executes the CSL → ISL pipeline.

**Internal flow**:
1. Normalizes the prompt
2. Creates `CSLInput` with source 'UI'
3. Invokes core `segment()`
4. Invokes core `sanitize()`
5. Returns core `ISLResult`

**Example**:
```typescript
const result = await pip.scanPrompt('User input here')
// result is an ISLResult from the core, with sanitization signals
```

#### `purify(content: string | CSLResult | ISLResult): Promise<CPEResult>`

**Description**: Executes the complete CSL → ISL → CPE core pipeline.

**Internal flow**:
1. If string, executes `scanPrompt()`
2. If `CSLResult`, executes core `sanitize()`
3. Invokes core `envelope()`
4. Returns core `CPEResult`

**Example**:
```typescript
const cslResult = await pip.scanDOM()
const cpeResult = await pip.purify(cslResult)
// cpeResult is a CPEResult from the core, with structured envelope
```

### 4.3 Operational Functions

These functions interpret core signals and execute operational actions:

#### `report(result: CSLResult | ISLResult | CPEResult): void`

**Description**: Generates operational reports based on core results.

**Note**: This function is purely operational from the SDK. The core does not generate reports, it only produces data structures.

#### `lock(action: string, reason?: string): boolean`

**Description**: Executes a local action according to the configured policy.

**⚠️ Important**: `lock()` **does NOT belong to the AI-PIP protocol**. It's an operational SDK action that can block actions in the environment (browser, server, etc.) according to signals interpreted from the core.

**Example**:
```typescript
const result = await pip.scanDOM()
if (result.segments.some(s => isUntrusted(s.trust))) {
  // This decision is from the SDK, not the core
  pip.lock('navigation', 'Untrusted content detected')
}
```

---

## 5. SDK Functions

### 5.1 Hash and Cryptography Functions

These functions implement cryptographic operations that the core defines but does not implement:

#### `hashContent(content: string, algorithm?: HashAlgorithm): ContentHash`

Generates cryptographic hash of content. The core defines the `ContentHash` type, the SDK implements the generation.

#### `verifyContentHash(content: string, hash: ContentHash): boolean`

Verifies if a hash corresponds to content.

#### `verifySignature(content: string, signature: string, secretKey: string): boolean`

Verifies a cryptographic signature. The core defines the envelope structure, the SDK implements the verification.

### 5.2 Detection Functions

These functions implement heuristics that the core does not contain:

#### `detectMimeType(content: string): string`

Detects the MIME type of content using deterministic heuristics.

#### `normalizeBasic(content: string): string`

Applies basic normalization to content.

#### `segmentSemantic(content: string, source: Source): string[]`

Segments content in an advanced semantic way.

### 5.3 Decision Functions

These functions interpret core signals and make decisions:

#### `shouldBlock(result: PiDetectionResult): boolean`

Determines if blocking is needed based on the core detection result.

**Note**: The core produces `PiDetectionResult` with signals. The SDK interprets these signals and decides actions.

#### `shouldWarn(result: PiDetectionResult): boolean`

Determines if a warning is needed based on the result.

### 5.4 Serialization Functions

These functions implement serialization that the core defines but does not implement:

#### `serializeContent(segments: readonly ISLSegment[]): string`

Serializes sanitized content for signing.

#### `serializeMetadata(metadata: CPEMetadata): string`

Serializes metadata for signing.

#### `generateSignableContent(content: string, metadata: string, algorithm: string): string`

Generates complete content for signing.

### 5.5 Auditing Functions

These functions analyze lineage for observability:

#### `getLineageStats(lineage: readonly LineageEntry[]): {...}`

Gets lineage statistics.

#### `getLineageByStep(lineage: readonly LineageEntry[], step: string): readonly LineageEntry[]`

Filters lineage by step.

#### `getLineageByTimeRange(...)`, `getLineageByNotes(...)`, etc.

Lineage analysis functions for operational auditing.

---

## 6. TypeScript Types

### 6.1 Core Types (Re-exported)

The SDK re-exports all semantic core types:

```typescript
// Core types (re-exported)
import type {
  // CSL
  CSLInput,
  CSLResult,
  CSLSegment,
  TrustLevel,
  Origin,
  LineageEntry,
  ContentHash,
  
  // ISL
  ISLResult,
  ISLSegment,
  PiDetection,
  PiDetectionResult,
  AnomalyScore,
  Pattern,
  
  // CPE
  CPEResult,
  CPEEvelope,
  CPEMetadata,
  Nonce,
  SignatureVO,
  
  // Value Objects
  TrustLevelType,
  OriginType,
  HashAlgorithm,
  Source,
  RiskScore,
  AnomalyAction
} from '@ai-pip/sdk'
```

### 6.2 SDK Types (Own)

The SDK defines additional types for its own operations:

```typescript
// SDK types (own)
interface RiskEvent {
  readonly level: 'low' | 'medium' | 'high'
  readonly reason: string
  readonly timestamp: number
  readonly source: 'CSL' | 'ISL' | 'CPE'
  readonly metadata?: Record<string, unknown>
}

interface SDKOptions {
  readonly enablePolicyValidation?: boolean
  readonly enableLineageTracking?: boolean
  readonly hashAlgorithm?: 'sha256' | 'sha512'
  readonly secretKey?: string
  readonly onRiskDetected?: (event: RiskEvent) => void
}

interface LockResult {
  readonly success: boolean
  readonly action: string
  readonly reason?: string
  readonly timestamp: number
}

interface RuntimeDecision {
  readonly action: 'ALLOW' | 'WARN' | 'BLOCK'
  readonly reason: string
  readonly confidence: number
  readonly source: 'policy' | 'heuristic' | 'core-signal'
}
```

**Important clarification**: The SDK can enrich core results with operational metadata (timestamps, runtime decisions, events), but these types are not part of the semantic protocol.

---

## 7. Usage Examples

### 7.1 Basic Usage in Browser

```typescript
import { scanDOM, purify, onRiskDetected } from '@ai-pip/sdk'

// Scan DOM
const cslResult = await scanDOM(document.body)

// Execute complete pipeline
const cpeResult = await purify(cslResult)

// Listen to risk events (generated by SDK)
onRiskDetected((event) => {
  console.log('Risk detected:', event.level, event.reason)
})
```

### 7.2 Usage with Direct Core

```typescript
import { segment, sanitize, envelope } from '@ai-pip/core'
import { hashContent, verifySignature } from '@ai-pip/sdk'

// Use core directly
const cslResult = segment({ content: '...', source: 'UI' })
const islResult = sanitize(cslResult)
const cpeResult = envelope(islResult, secretKey)

// Use SDK functions for additional operations
const hash = hashContent(cslResult.segments[0].content)
const isValid = verifySignature(signableContent, cpeResult.envelope.signature.value, secretKey)
```

### 7.3 React Integration

```typescript
import { useEffect, useState } from 'react'
import { scanDOM, onRiskDetected } from '@ai-pip/sdk'

function useAIPIPProtection() {
  const [riskLevel, setRiskLevel] = useState<'low' | 'medium' | 'high' | null>(null)
  
  useEffect(() => {
    const handleRisk = (event: RiskEvent) => {
      setRiskLevel(event.level)
    }
    
    onRiskDetected(handleRisk)
    
    // Scan DOM periodically
    const interval = setInterval(async () => {
      await scanDOM()
    }, 5000)
    
    return () => {
      clearInterval(interval)
    }
  }, [])
  
  return riskLevel
}
```

### 7.4 Node.js Integration

```typescript
import { scanPrompt, purify } from '@ai-pip/sdk'

async function processUserInput(prompt: string) {
  // Process prompt
  const islResult = await scanPrompt(prompt)
  
  // Execute complete pipeline
  const cpeResult = await purify(islResult)
  
  // SDK can enrich with operational metadata
  return {
    ...cpeResult, // Core result
    processedAt: Date.now(), // SDK metadata
    environment: 'node' // SDK metadata
  }
}
```

---

## 8. Integration

### 8.1 Browser Integration

The SDK provides adapters for browsers:

```typescript
import { DOMAdapter } from '@ai-pip/sdk/adapters'

const adapter = new DOMAdapter({
  enableHiddenContentDetection: true,
  enableMimeDetection: true
})

const content = adapter.extractContent(document.body)
const cslResult = segment({ content, source: 'DOM' })
```

### 8.2 Node.js Integration

```typescript
import { SystemTimestampProvider, CryptoHashGenerator } from '@ai-pip/sdk/adapters'

const timestampProvider = new SystemTimestampProvider()
const hashGenerator = new CryptoHashGenerator('sha256')

// Use with core
const lineage = createLineageEntry('CSL', timestampProvider.now())
const hash = hashGenerator.generate(content)
```

### 8.3 Edge Computing Integration

The SDK can run in edge environments:

```typescript
import { segment, sanitize } from '@ai-pip/core'
import { serializeContent } from '@ai-pip/sdk'

// Execute core in edge
const cslResult = segment({ content: '...', source: 'API' })
const islResult = sanitize(cslResult)

// Serialize for transmission
const serialized = serializeContent(islResult.segments)
```

---

## 9. Configuration

### 9.1 SDK Configuration

```typescript
import { configure } from '@ai-pip/sdk'

configure({
  enablePolicyValidation: true,
  enableLineageTracking: true,
  hashAlgorithm: 'sha256',
  secretKey: process.env.AI_PIP_SECRET_KEY,
  onRiskDetected: (event) => {
    // Handle risk events
    console.log('Risk detected:', event)
  }
})
```

### 9.2 Operational Policies

The SDK allows configuring policies that interpret core signals:

```typescript
interface PolicyConfig {
  // TrustLevel levels that should be blocked
  blockUntrusted: boolean
  blockSemiTrusted: boolean
  
  // Anomaly scores that should be blocked
  blockHighRisk: boolean
  warnMediumRisk: boolean
  
  // Actions to take
  onBlock: (reason: string) => void
  onWarn: (reason: string) => void
}
```

**Note**: These policies are from the SDK, not the protocol. The protocol only produces signals.

---

## 10. Events and Callbacks

### 10.1 SDK Events

**Important**: The semantic core **does NOT emit events**. Events are generated by the SDK when it interprets core results.

#### `onRiskDetected(callback: (risk: RiskEvent) => void): void`

Registers a callback that executes when the SDK interprets a risk based on core signals.

**Example**:
```typescript
onRiskDetected((event) => {
  // This event is generated by the SDK, not the core
  if (event.level === 'high') {
    // SDK operational action
    lock('navigation', event.reason)
  }
})
```

#### `offRiskDetected(callback: (risk: RiskEvent) => void): void`

Unregisters an event callback.

### 10.2 Signal Interpretation

The SDK interprets core signals and generates events:

```typescript
// Core produces signals
const islResult = sanitize(cslResult)
const hasHighRisk = islResult.segments.some(s => 
  s.piDetection && isHighRisk(s.piDetection.confidence)
)

// SDK interprets and generates event
if (hasHighRisk) {
  emitRiskEvent({
    level: 'high',
    reason: 'High confidence PI detection',
    source: 'ISL',
    timestamp: Date.now()
  })
}
```

---

## 11. Best Practices

### 11.1 Core/SDK Separation

- ✅ Use the core directly when you need pure functions
- ✅ Use the SDK when you need adaptation to specific environments
- ✅ Don't mix responsibilities: the core produces signals, the SDK executes actions

### 11.2 Error Handling

```typescript
try {
  const result = await scanDOM()
  // Process core result
} catch (error) {
  if (error instanceof SegmentationError) {
    // Core error
  } else {
    // SDK or environment error
  }
}
```

### 11.3 Performance

- The core is optimized for low overhead
- Performance depends on the execution environment
- Use SDK functions only when you need specific adaptation

---

## 12. Conclusion

The AI-PIP SDK is an integration layer that:

- ✅ Consumes the semantic core (`@ai-pip/core`)
- ✅ Adapts the core to concrete environments
- ✅ Interprets core signals and executes operational actions
- ✅ Provides utilities and helpers to facilitate usage

**The SDK does NOT**:
- ❌ Define security logic (that's the core)
- ❌ Is the protocol (it's a reference implementation)
- ❌ Replaces the core (it complements it)

To understand the semantic protocol, see: [Semantic Core](../core/README.md)  
To understand the architecture, see: [Semantic Architecture](../architecture.md)

---

**Document Version**: 2.0  
**Last Updated**: 2026-01-01  
**Author**: Felipe Masliah  
**License**: Apache-2.0

---

*This document describes the reference SDK. To understand the formal semantic protocol, consult the core documentation.*

