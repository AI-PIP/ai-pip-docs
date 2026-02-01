# Shared — Shared Utilities

> **Shared utilities** — Pure functions shared across all layers of the AI-PIP protocol.

---

## 1. Overview

The **Shared** layer provides pure functions shared by all layers of the AI-PIP protocol (CSL, ISL, CPE). Its main function is lineage (lineage) handling, which tracks content processing across all layers.

### 1.1 Principles

- **Purity**: All functions are pure (no side effects)
- **Immutability**: Lineage arrays are immutable; functions return new arrays
- **Simplicity**: Only basic lineage handling functions
- **Traceability**: Preserves full processing history

---

## 2. Main Functionality

### 2.1 Lineage Handling

Lineage is an immutable record of content processing through the different layers. Each entry records:
- **Step**: The layer that processed the content (e.g. 'CSL', 'ISL', 'CPE')
- **Timestamp**: When it was processed

### 2.2 Lineage Operations

Shared functions allow:
- Adding entries to lineage
- Filtering entries by step
- Getting the last entry

---

## 3. Components

### 3.1 Main Functions

#### Adding Entries
- **`addLineageEntry(lineage: readonly LineageEntry[], entry: LineageEntry): LineageEntry[]`** — Appends a lineage entry to an existing array. Returns a new immutable array.

- **`addLineageEntries(lineage: readonly LineageEntry[], entries: readonly LineageEntry[]): LineageEntry[]`** — Appends multiple lineage entries to an existing array. Returns a new immutable array.

#### Filtering and Querying
- **`filterLineageByStep(lineage: readonly LineageEntry[], step: string): LineageEntry[]`** — Filters lineage entries by step. Returns a new array with only entries matching the step.

- **`getLastLineageEntry(lineage: readonly LineageEntry[]): LineageEntry | undefined`** — Returns the last lineage entry. Returns `undefined` if lineage is empty.

### 3.2 Types

#### LineageEntry
The `LineageEntry` type is imported from CSL and has the following structure:

```typescript
{
  step: string        // Processing step (e.g. 'CSL', 'ISL', 'CPE')
  timestamp: number   // Unix timestamp in milliseconds
}
```

---

## 4. Processing Flow

Lineage flows through all layers:

```
CSL → Initializes lineage with 'CSL' entry
  ↓
ISL → Appends 'ISL' entry to lineage
  ↓
CPE → Appends 'CPE' entry to lineage
  ↓
Full lineage with all layers
```

---

## 5. Guarantees

1. **Immutability**: All functions return new arrays; they never mutate the original lineage
2. **Purity**: No side effects; deterministic functions
3. **Preservation**: Full lineage is preserved across all layers
4. **Traceability**: Every processing step is recorded

---

## 6. Usage Examples

### 6.1 Basic: Add entry

```typescript
import { addLineageEntry, createLineageEntry } from '@ai-pip/core'

// Initial lineage (from CSL)
const initialLineage = [
  { step: 'CSL', timestamp: Date.now() }
]

// Add ISL entry
const islEntry = createLineageEntry('ISL', Date.now())
const updatedLineage = addLineageEntry(initialLineage, islEntry)

// updatedLineage now contains:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... }
// ]
```

### 6.2 Add multiple entries

```typescript
import { addLineageEntries, createLineageEntry } from '@ai-pip/core'

const initialLineage = [
  { step: 'CSL', timestamp: Date.now() }
]

// Create multiple entries
const newEntries = [
  createLineageEntry('ISL', Date.now()),
  createLineageEntry('CPE', Date.now())
]

// Add all entries at once
const fullLineage = addLineageEntries(initialLineage, newEntries)

// fullLineage now contains:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... },
//   { step: 'CPE', timestamp: ... }
// ]
```

### 6.3 Filter by step

```typescript
import { filterLineageByStep } from '@ai-pip/core'

const lineage = [
  { step: 'CSL', timestamp: 1000 },
  { step: 'ISL', timestamp: 2000 },
  { step: 'CPE', timestamp: 3000 },
  { step: 'ISL', timestamp: 4000 }
]

// Filter only ISL entries
const islEntries = filterLineageByStep(lineage, 'ISL')

// islEntries contains:
// [
//   { step: 'ISL', timestamp: 2000 },
//   { step: 'ISL', timestamp: 4000 }
// ]
```

### 6.4 Get last entry

```typescript
import { getLastLineageEntry } from '@ai-pip/core'

const lineage = [
  { step: 'CSL', timestamp: 1000 },
  { step: 'ISL', timestamp: 2000 },
  { step: 'CPE', timestamp: 3000 }
]

// Get last entry
const lastEntry = getLastLineageEntry(lineage)

// lastEntry is: { step: 'CPE', timestamp: 3000 }

// With empty lineage
const emptyLineage: LineageEntry[] = []
const noEntry = getLastLineageEntry(emptyLineage)
// noEntry is: undefined
```

### 6.5 Full pipeline with lineage

```typescript
import {
  segment,
  sanitize,
  envelope,
  addLineageEntry,
  filterLineageByStep,
  getLastLineageEntry,
  createLineageEntry
} from '@ai-pip/core'
import type { LineageEntry } from '@ai-pip/core'

// 1. CSL initializes lineage
const cslResult = segment({
  content: 'Test content',
  source: 'UI',
  metadata: {}
})

// Initial lineage contains: [{ step: 'CSL', timestamp: ... }]

// 2. ISL adds entry to lineage (automatically in sanitize)
const islResult = sanitize(cslResult)

// Lineage now contains:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... }
// ]

// 3. CPE adds entry to lineage (automatically in envelope)
const cpeResult = envelope(islResult, 'secret-key')

// Final lineage contains:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... },
//   { step: 'CPE', timestamp: ... }
// ]

// 4. Query lineage
const fullLineage = cpeResult.envelope.lineage

// Get last entry
const lastEntry = getLastLineageEntry(fullLineage)
console.log('Last step:', lastEntry?.step) // 'CPE'

// Filter entries for a specific layer
const cslEntries = filterLineageByStep(fullLineage, 'CSL')
console.log('CSL entries:', cslEntries.length) // 1

// Add custom entry (e.g. for audit)
const auditEntry = createLineageEntry('AUDIT', Date.now())
const lineageWithAudit = addLineageEntry(fullLineage, auditEntry)
```

### 6.6 Audit and analysis

```typescript
import {
  filterLineageByStep,
  getLastLineageEntry
} from '@ai-pip/core'
import type { LineageEntry } from '@ai-pip/core'

function analyzeLineage(lineage: readonly LineageEntry[]) {
  // Get last entry
  const lastEntry = getLastLineageEntry(lineage)
  console.log('Last processing step:', lastEntry?.step)

  // Count entries by step
  const steps = ['CSL', 'ISL', 'CPE'] as const
  steps.forEach(step => {
    const entries = filterLineageByStep(lineage, step)
    console.log(`${step} entries: ${entries.length}`)
  })

  // Calculate total processing time
  if (lineage.length >= 2) {
    const first = lineage[0]
    const last = lastEntry
    if (first && last) {
      const totalTime = last.timestamp - first.timestamp
      console.log(`Total processing time: ${totalTime}ms`)
    }
  }
}

// Usage
const lineage = cpeResult.envelope.lineage
analyzeLineage(lineage)
```

---

## 7. Integration with Other Layers

### CSL
- CSL initializes lineage with `initLineage()`, which creates the first entry with step 'CSL'.

### ISL
- ISL appends an entry to lineage with step 'ISL' during sanitization.

### CPE
- CPE appends an entry to lineage with step 'CPE' during envelope generation.

---

## 8. Core Limitations

The Shared core **does not include**:
- Advanced lineage analysis (handled by the SDK)
- Processing statistics (handled by the SDK)
- Lineage serialization (handled by the SDK)
- Complex lineage queries (handled by the SDK)

These functionalities are implemented in the SDK or in audit tools.

---

## 9. References

- **LineageEntry**: Available from `@ai-pip/core`
- **createLineageEntry**: Available from `@ai-pip/core`
- **initLineage**: Available from `@ai-pip/core`

> **Note**: Shared functions are available from the main entry point `@ai-pip/core`, not from a specific subpath.
