# AI-PIP SDK Reference

> **Complete reference guide for the AI-PIP SDK**
> 
> This document describes all functions and features available in the SDK, including those outside the semantic core but necessary to implement the protocol.

---

## 📋 Table of Contents

1. [SDK vs Core Architecture](#1-sdk-vs-core-architecture)
2. [CSL SDK Features](#2-csl-sdk-features)
3. [ISL SDK Features](#3-isl-sdk-features)
4. [CPE SDK Features](#4-cpe-sdk-features)
5. [Shared SDK Features](#5-shared-sdk-features)
6. [Advanced Features](#6-advanced-features)

---

## 1. SDK vs Core Architecture

### Semantic Core
- Defines **what** the protocol does
- Pure and deterministic functions
- No state, no decisions, no specific implementations

### SDK
- Defines **how** to use the protocol
- Implements features necessary to use the core
- May have state, decisions, serialization, verification

### Relationship

```
┌─────────────────────────────────────┐
│      SDK / Infrastructure            │
│  - Hash generation                   │
│  - MIME detection                    │
│  - Normalization                     │
│  - Semantic segmentation             │
│  - Serialization                     │
│  - Verification                      │
│  - Policy decisions                  │
│  - Audit & analytics                 │
└──────────────┬──────────────────────┘
               │ uses
               ▼
┌─────────────────────────────────────┐
│      Semantic Core                  │
│  - segment()                        │
│  - sanitize()                       │
│  - envelope()                       │
│  - Value objects                    │
└─────────────────────────────────────┘
```

---

## 2. CSL SDK Features

### Hash and Cryptography

#### `hashContent(content: string, algorithm?: HashAlgorithm): ContentHash`
Generates cryptographic hash of content.

```typescript
import { hashContent } from '@ai-pip/sdk/csl'

const hash = hashContent('Hello World', 'sha256')
// { value: 'a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e', algorithm: 'sha256' }
```

#### `verifyContentHash(content: string, hash: ContentHash): boolean`
Verifies if a hash corresponds to content.

```typescript
import { verifyContentHash } from '@ai-pip/sdk/csl'

const isValid = verifyContentHash('Hello World', hash) // true
```

### MIME Detection

#### `detectMimeType(content: string): string`
Detects the MIME type of content using heuristics.

```typescript
import { detectMimeType } from '@ai-pip/sdk/csl'

detectMimeType('<html><body>Hello</body></html>') // 'text/html'
detectMimeType('{"key": "value"}') // 'application/json'
detectMimeType('function test() {}') // 'application/javascript'
detectMimeType('Hello World') // 'text/plain'
```

**Detected types:**
- `text/html` - HTML
- `application/json` - JSON
- `application/xml` - XML
- `text/css` - CSS
- `application/javascript` - JavaScript
- `text/plain` - Default

### Normalization

#### `normalizeBasic(content: string): string`
Applies basic normalization to content.

```typescript
import { normalizeBasic } from '@ai-pip/sdk/csl'

normalizeBasic('Hello\u200B\u200Cworld') // 'Helloworld'
normalizeBasic('&lt;script&gt;') // '<script>'
normalizeBasic('Hello    World') // 'Hello World'
```

**Applied normalizations:**
- Unicode NFC (Canonical Composition)
- Removal of zero-width characters (U+200B, U+200C, U+200D, U+FEFF)
- HTML entity decoding
- Space normalization
- Control character removal

### Semantic Segmentation

#### `segmentSemantic(content: string, source: Source): string[]`
Segments content in an advanced semantic way.

```typescript
import { segmentSemantic } from '@ai-pip/sdk/csl'

segmentSemantic('# Header\nContent', 'UI')
// ['# Header', 'Content']

segmentSemantic('```code\nhere\n```', 'UI')
// ['```code\nhere\n```']
```

**Strategies:**
1. Code blocks (```...```)
2. Headers (Markdown #)
3. Lists (- item, * item, 1. item)
4. Paragraphs (double line break)
5. Lines (fallback)

---

## 3. ISL SDK Features

### Decisions and Policies

#### `shouldBlock(result: PiDetectionResult): boolean`
Determines if blocking is needed based on the detection result.

```typescript
import { shouldBlock } from '@ai-pip/sdk/isl'

const result = createPiDetectionResult([...])
if (shouldBlock(result)) {
  // Block content
}
```

#### `shouldWarn(result: PiDetectionResult): boolean`
Determines if a warning is needed based on the result.

#### `PolicyRule` and Related Functions

```typescript
import { 
  createPolicyRule,
  isIntentBlocked,
  isScopeSensitive,
  isRoleProtected
} from '@ai-pip/sdk/isl'

const policy = createPolicyRule(
  '1.0',
  ['delete_user_data', 'modify_system_settings'],
  ['financial_transactions'],
  { protectedRoles: ['system'], immutableInstructions: [...] },
  { enabled: true, blockMetadataExposure: true, ... }
)

if (isIntentBlocked(policy, 'delete_user_data')) {
  // Block intent
}
```

**PolicyRule components:**
- `blockedIntents` - Prohibited intents
- `sensitiveScope` - Sensitive scopes
- `roleProtection` - Role protection
- `contextLeakPrevention` - Context leak prevention

---

## 4. CPE SDK Features

### Serialization

#### `serializeContent(segments: readonly ISLSegment[]): string`
Serializes sanitized content for signing.

```typescript
import { serializeContent } from '@ai-pip/sdk/cpe'

const serialized = serializeContent(islResult.segments)
// Format: [0]:content1\n[1]:content2\n...
```

#### `serializeMetadata(metadata: CPEMetadata): string`
Serializes metadata for signing.

```typescript
import { serializeMetadata } from '@ai-pip/sdk/cpe'

const serialized = serializeMetadata(cpeMetadata)
// Format: timestamp:123|nonce:abc|version:1.0.0|...
```

#### `generateSignableContent(content: string, metadata: string, algorithm: string): string`
Generates complete content for signing.

```typescript
import { generateSignableContent } from '@ai-pip/sdk/cpe'

const signable = generateSignableContent(
  serializedContent,
  serializedMetadata,
  'HMAC-SHA256'
)
```

### Verification

#### `verifySignature(content: string, signature: string, secretKey: string): boolean`
Verifies a cryptographic signature.

```typescript
import { verifySignature } from '@ai-pip/sdk/cpe'

const isValid = verifySignature(
  signableContent,
  envelope.signature.value,
  secretKey
)
```

#### `isValidSignatureFormat(signature: string): boolean`
Validates the format of a signature.

```typescript
import { isValidSignatureFormat } from '@ai-pip/sdk/cpe'

isValidSignatureFormat('a1b2c3d4...') // true if hex of 64 chars
```

---

## 5. Shared SDK Features

### Lineage Auditing and Analysis

#### `getLineageStats(lineage: readonly LineageEntry[]): {...}`
Gets lineage statistics.

```typescript
import { getLineageStats } from '@ai-pip/sdk/shared'

const stats = getLineageStats(lineage)
// {
//   totalEntries: 5,
//   steps: { CSL: 2, ISL: 2, CPE: 1 },
//   timeRange: { start: 1000, end: 1050, duration: 50 },
//   entriesWithNotes: 3
// }
```

#### `getLineageByStep(lineage: readonly LineageEntry[], step: string): readonly LineageEntry[]`
Filters lineage by step.

#### `getLineageByTimeRange(lineage: readonly LineageEntry[], startTime: number, endTime: number): readonly LineageEntry[]`
Filters lineage by time range.

#### `getLineageByNotes(lineage: readonly LineageEntry[], searchTerm: string): readonly LineageEntry[]`
Searches in lineage notes.

#### `isLineageChronological(lineage: readonly LineageEntry[]): boolean`
Verifies if lineage is in chronological order.

#### `getTotalProcessingTime(lineage: readonly LineageEntry[]): number | undefined`
Calculates total processing time.

#### `getStepSequence(lineage: readonly LineageEntry[]): readonly string[]`
Gets step sequence in lineage.

### LineageEntry with Notes

The SDK can extend `LineageEntry` with notes for observability:

```typescript
type LineageEntryWithNotes = LineageEntry & {
  readonly notes?: string
}
```

---

## 6. Advanced Features

### Complete Integration

```typescript
import { segment } from '@ai-pip/core/csl'
import { sanitize } from '@ai-pip/core/isl'
import { envelope } from '@ai-pip/core/cpe'
import { 
  hashContent, 
  detectMimeType, 
  normalizeBasic 
} from '@ai-pip/sdk/csl'
import { shouldBlock } from '@ai-pip/sdk/isl'
import { verifySignature } from '@ai-pip/sdk/cpe'

// 1. Pre-processing (SDK)
const normalized = normalizeBasic(rawContent)
const mime = detectMimeType(normalized)

// 2. Core: Segmentation
const cslResult = segment({
  content: normalized,
  source: 'UI',
  metadata: { mime }
})

// 3. Core: Sanitization
const islResult = sanitize(cslResult)

// 4. Decisions (SDK)
if (islResult.segments[0]?.piDetection && shouldBlock(islResult.segments[0].piDetection)) {
  throw new Error('Content blocked')
}

// 5. Core: Envelope
const cpeResult = envelope(islResult, secretKey)

// 6. Verification (SDK)
const isValid = verifySignature(
  generateSignableContent(...),
  cpeResult.envelope.signature.value,
  secretKey
)
```

### Factory Functions

The SDK provides factory functions to facilitate usage:

```typescript
import { createCSLService } from '@ai-pip/sdk/csl'

const service = createCSLService({
  enablePolicyValidation: true,
  enableLineageTracking: true,
  hashAlgorithm: 'sha256'
})

const result = await service.segment(document.body)
```

### Adapters

The SDK includes adapters for different environments:

- `DOMAdapter` - DOM adapter
- `UIAdapter` - UI adapter
- `CryptoHashGenerator` - Cryptographic hash generator
- `SystemTimestampProvider` - Timestamp provider
- `ConsoleLogger` - Console logger

---

## 📊 SDK Features Summary

| Category | Functions | Location |
|----------|-----------|----------|
| **Hash and Cryptography** | 2 functions | `@ai-pip/sdk/csl` |
| **MIME Detection** | 1 function | `@ai-pip/sdk/csl` |
| **Normalization** | 1 function | `@ai-pip/sdk/csl` |
| **Semantic Segmentation** | 1 function | `@ai-pip/sdk/csl` |
| **ISL Decisions** | 2 functions | `@ai-pip/sdk/isl` |
| **Policies** | 6 functions | `@ai-pip/sdk/isl` |
| **CPE Serialization** | 3 functions | `@ai-pip/sdk/cpe` |
| **CPE Verification** | 2 functions | `@ai-pip/sdk/cpe` |
| **Lineage Auditing** | 7 functions | `@ai-pip/sdk/shared` |
| **TOTAL** | **25+ functions** | Complete SDK |

---

## 🎯 Recommended Usage

1. **Core** for pure protocol logic
2. **SDK** for practical implementation
3. **Adapters** for integration with specific environments
4. **Factory Functions** for easy configuration

The SDK is the layer that makes the core usable in the real world, without losing the semantic purity of the core.

