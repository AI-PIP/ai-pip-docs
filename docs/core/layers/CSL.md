# CSL (Context Segmentation Layer)

> **Context Segmentation Layer** — First layer of the AI-PIP protocol. Segments input content into semantic segments and classifies them by trust level based on origin.

---

## 1. Overview

The **Context Segmentation Layer (CSL)** is the first layer of the AI-PIP protocol. Its main function is to segment input content into semantic segments and classify them by trust level based solely on their origin.

### 1.1 Principles

- **Determinism**: Same origin → same trust level, always
- **Purity**: Functions with no side effects
- **Immutability**: All objects are immutable
- **Preservation**: Original content is never lost

---

## 2. Main Functionality

### 2.1 Content Segmentation

The `segment()` function splits input content into semantic segments based on context rules (line breaks, delimiters, etc.).

```typescript
import { segment } from '@ai-pip/core/csl'

const result = segment({
  content: 'Hello\nWorld\n---\nUser input',
  source: 'UI',
  metadata: {}
})

// result.segments contains the classified segments
```

### 2.2 Classification by Origin

CSL classifies each segment according to its origin (`source`) or origin type (`origin`):

#### Classification by Source

- **`UI`** → **TC** (Trusted Content)
- **`SYSTEM`** → **TC** (Trusted Content)
- **`DOM`** → **STC** (Semi-Trusted Content)
- **`API`** → **UC** (Untrusted Content)

#### Classification by Origin

- **`SYSTEM_GENERATED`** → **TC**
- **`DOM_VISIBLE`** → **STC**
- **`DOM_HIDDEN`** → **UC**
- **`USER`** → **UC**
- **`UNKNOWN`** → **UC** (fail-secure)

### 2.3 Lineage Initialization

Each segment receives an initial lineage entry that records:
- **Step**: `'CSL'`
- **Timestamp**: Creation time

---

## 3. Components

### 3.1 Main Functions

#### Segmentation
- **`segment(input: CSLInput): CSLResult`** — Main segmentation function. Splits content into semantic segments and classifies them by origin.

#### Classification
- **`classifySource(source: Source): TrustLevel`** — Classifies a source and returns its TrustLevel. Deterministic: same source → same trust level.
- **`classifyOrigin(origin: Origin): TrustLevel`** — Classifies an Origin and returns its TrustLevel based on the originMap.

#### Lineage
- **`initLineage(segment: CSLSegment): LineageEntry[]`** — Initializes a segment's lineage with a CSL entry.
- **`createLineageEntry(step: string, timestamp: number): LineageEntry`** — Creates a lineage entry with step and timestamp.

#### Utilities
- **`generateId(): string`** — Generates a unique ID for a segment (format: `seg-{timestamp}-{random}`).
- **`splitByContextRules(content: string): string[]`** — Splits content by context rules (line breaks). Pure basic segmentation function.

### 3.2 Value Objects

#### TrustLevel
- **Type**: `TrustLevel` — Immutable trust level (TC, STC, UC)
- **Creation**: `createTrustLevel(value: TrustLevelType): TrustLevel`
- **Helpers**:
  - `isTrusted(trust: TrustLevel): boolean` — Returns true if TC
  - `isSemiTrusted(trust: TrustLevel): boolean` — Returns true if STC
  - `isUntrusted(trust: TrustLevel): boolean` — Returns true if UC

#### Origin
- **Type**: `Origin` — Immutable content origin
- **Creation**: `createOrigin(type: OriginType): Origin`
- **Helpers**:
  - `isDom(origin: Origin): boolean` — Returns true if DOM origin
  - `isUser(origin: Origin): boolean` — Returns true if USER
  - `isSystem(origin: Origin): boolean` — Returns true if SYSTEM_GENERATED
  - `isInjected(origin: Origin): boolean` — Returns true if SCRIPT_INJECTED
  - `isUnknown(origin: Origin): boolean` — Returns true if UNKNOWN
  - `isNetworkFetched(origin: Origin): boolean` — Returns true if NETWORK_FETCHED
  - `isExternal(origin: Origin): boolean` — Returns true if external (NETWORK_FETCHED or SCRIPT_INJECTED)

#### LineageEntry
- **Type**: `LineageEntry` — Immutable lineage entry
- **Properties**:
  - `step: string` — Processing step (e.g. 'CSL', 'ISL', 'CPE')
  - `timestamp: number` — Unix timestamp in milliseconds
- **Creation**: `createLineageEntry(step: string, timestamp: number): LineageEntry`

#### ContentHash
- **Type**: `ContentHash` — Immutable content hash
- **Properties**:
  - `value: string` — Hexadecimal hash value
  - `algorithm: HashAlgorithm` — Algorithm used ('sha256' | 'sha512')
- **Creation**: `createContentHash(value: string, algorithm?: HashAlgorithm): ContentHash`
- **Helpers**:
  - `isSha256(hash: ContentHash): boolean` — Returns true if SHA256
  - `isSha512(hash: ContentHash): boolean` — Returns true if SHA512

#### Origin Map
- **Constant**: `originMap: Map<OriginType, TrustLevelType>` — Deterministic mapping of OriginType to TrustLevelType
- **Validation**: `validateOriginMap(): void` — Validates that all OriginTypes are mapped

### 3.3 Types

#### Enums
- **`OriginType`** — Content origin type:
  - `USER` — Direct user input
  - `DOM_VISIBLE` — Visible DOM content
  - `DOM_HIDDEN` — Hidden DOM content
  - `DOM_ATTRIBUTE` — DOM attributes (data-*, aria-*)
  - `SCRIPT_INJECTED` — Script-injected content
  - `NETWORK_FETCHED` — Content from network/API
  - `SYSTEM_GENERATED` — System-generated content
  - `UNKNOWN` — Unknown origin

- **`TrustLevelType`** — Trust level:
  - `TC` — Trusted Content
  - `STC` — Semi-Trusted Content
  - `UC` — Untrusted Content

#### Basic Types
- **`Source`** — Content source: `'DOM' | 'UI' | 'SYSTEM' | 'API'`
- **`HashAlgorithm`** — Hash algorithm: `'sha256' | 'sha512'`

#### Interfaces
- **`CSLInput`** — Input for segmentation:
  ```typescript
  {
    content: string
    source: Source
    metadata?: Record<string, unknown>
  }
  ```

- **`CSLSegment`** — Classified segment:
  ```typescript
  {
    id: string
    content: string
    source: Source
    trust: TrustLevel
    lineage: LineageEntry[]
    hash?: ContentHash
    metadata?: Record<string, unknown>
  }
  ```

- **`CSLResult`** — Segmentation result:
  ```typescript
  {
    segments: readonly CSLSegment[]
    lineage: readonly LineageEntry[]
    processingTimeMs?: number
  }
  ```

### 3.4 Exceptions

- **`ClassificationError`** — Thrown when classification fails (unmapped origin, etc.)
- **`SegmentationError`** — Thrown when segmentation fails (invalid content, etc.)

---

## 4. Processing Flow

```
Input (content + source)
    ↓
Segmentation (splitByContextRules)
    ↓
Classification (classifySource / classifyOrigin)
    ↓
Lineage initialization (initLineage)
    ↓
CSLResult (segments + lineage)
```

---

## 5. Guarantees

1. **Integrity**: Original content is preserved in each segment
2. **Determinism**: Same input → same output
3. **Traceability**: Every segment has initialized lineage
4. **Fail-Secure**: Unknown origins are classified as UC

---

## 6. Usage Examples

### 6.1 Basic segmentation

```typescript
import { segment } from '@ai-pip/core'

// Segment content
const result = segment({
  content: 'System prompt\n---\nUser: Hello',
  source: 'UI',
  metadata: { sessionId: '123' }
})

// result.segments contains the classified segments
// result.lineage contains the initial lineage
```

### 6.2 Source classification

```typescript
import { classifySource, isTrusted, isSemiTrusted, isUntrusted } from '@ai-pip/core'

// Classify different sources
const uiTrust = classifySource('UI')   // { value: 'TC' }
const domTrust = classifySource('DOM') // { value: 'STC' }
const apiTrust = classifySource('API') // { value: 'UC' }

// Check trust levels
console.log(isTrusted(uiTrust))      // true
console.log(isSemiTrusted(domTrust)) // true
console.log(isUntrusted(apiTrust))   // true
```

### 6.3 Working with Origins

```typescript
import {
  createOrigin,
  OriginType,
  classifyOrigin,
  isDom,
  isSystem,
  isExternal
} from '@ai-pip/core'

// Create an Origin
const origin = createOrigin(OriginType.DOM_VISIBLE)

// Classify the Origin
const trust = classifyOrigin(origin) // { value: 'STC' }

// Check origin type
console.log(isDom(origin))      // true
console.log(isSystem(origin))   // false
console.log(isExternal(origin))  // false
```

### 6.4 ContentHash

```typescript
import { createContentHash, isSha256, isSha512 } from '@ai-pip/core'

// Create SHA256 hash
const hash256 = createContentHash(
  'a1b2c3d4e5f6...', // 64 hex characters
  'sha256'
)

// Create SHA512 hash (default is sha256)
const hash512 = createContentHash(
  'a1b2c3d4e5f6...', // 128 hex characters
  'sha512'
)

// Check algorithm
console.log(isSha256(hash256)) // true
console.log(isSha512(hash512)) // true
```

### 6.5 Lineage

```typescript
import { createLineageEntry, initLineage } from '@ai-pip/core'

// Create lineage entry manually
const entry = createLineageEntry('CSL', Date.now())

// Initialize lineage for a segment
const segment = {
  id: 'seg-123',
  content: 'Test',
  source: 'UI' as const,
  trust: { value: 'TC' as const },
  lineage: []
}

const lineage = initLineage(segment)
// Returns: [{ step: 'CSL', timestamp: ... }]
```

### 6.6 Utilities

```typescript
import { generateId, splitByContextRules } from '@ai-pip/core'

// Generate unique ID
const id = generateId() // 'seg-1234567890-abc123'

// Split content by context rules
const segments = splitByContextRules('Line 1\nLine 2\n\nLine 3')
// Returns: ['Line 1', 'Line 2', 'Line 3']
```

### 6.7 Full pipeline: CSL

```typescript
import {
  segment,
  classifySource,
  createOrigin,
  OriginType,
  isTrusted
} from '@ai-pip/core'
import type { CSLResult, CSLSegment } from '@ai-pip/core'

// 1. Segment content
const cslResult: CSLResult = segment({
  content: 'System: You are a helpful assistant\n---\nUser: Hello',
  source: 'UI',
  metadata: { sessionId: 'abc123' }
})

// 2. Process each segment
cslResult.segments.forEach((seg: CSLSegment) => {
  console.log(`Segment ID: ${seg.id}`)
  console.log(`Content: ${seg.content}`)
  console.log(`Trust Level: ${seg.trust.value}`)
  console.log(`Is Trusted: ${isTrusted(seg.trust)}`)
  console.log(`Lineage entries: ${seg.lineage.length}`)
})

// 3. Classify a specific source
const trust = classifySource('API')
console.log(`API trust level: ${trust.value}`) // 'UC'
```

---

## 7. Integration with ISL

CSL passes its result to ISL via the contract:

```typescript
CSLResult {
  segments: CSLSegment[]  // Classified segments
  lineage: LineageEntry[] // Initial lineage
}
```

ISL receives this result and applies sanitization according to each segment's `trust` level.

---

## 8. Core Limitations

The CSL core **does not include**:
- Aggressive content normalization
- Prompt-injection detection (handled by ISL)
- Security policies (handled by ISL)
- Stateful services (handled by the SDK)

These functionalities are implemented in upper layers or in the SDK.
