# CPE (Cryptographic Prompt Envelope)

> **Cryptographic Prompt Envelope** — Third layer of the AI-PIP protocol. Builds a cryptographic envelope that guarantees the integrity and authenticity of the prompt processed by previous layers.

---

## 1. Overview

The **Cryptographic Prompt Envelope (CPE)** is the third layer of the AI-PIP protocol. Its main function is to produce a cryptographic envelope that guarantees the integrity and authenticity of the prompt processed by the previous layers.

### 1.1 Principles

- **Cryptographic integrity**: HMAC-SHA256 signature of the content
- **Non-repudiation**: Unique timestamp and nonce
- **Full traceability**: Complete lineage preserved
- **Security metadata**: Audit information

---

## 2. Main Functionality

### 2.1 Security Metadata Generation

CPE generates metadata that includes:

- **Timestamp**: Envelope creation time
- **Nonce**: Unique value to prevent replay attacks
- **Protocol Version**: AI-PIP protocol version
- **Previous Signatures**: Optional signatures from previous layers (CSL, ISL)

```typescript
import { createMetadata, CURRENT_PROTOCOL_VERSION } from '@ai-pip/core/cpe'

const metadata = createMetadata(
  Date.now(),
  nonce,
  CURRENT_PROTOCOL_VERSION,
  {
    csl: 'csl-signature-123',  // Optional
    isl: 'isl-signature-456'   // Optional
  }
)
```

### 2.2 HMAC-SHA256 Cryptographic Signature

CPE produces a cryptographic signature of the content using HMAC-SHA256:

```typescript
import { createSignature, verifySignature } from '@ai-pip/core/cpe'

// Generate signature
const signature = createSignature(
  signableContent,
  secretKey
)

// Verify signature
const isValid = verifySignature(
  content,
  signature.value,
  secretKey
)
```

### 2.3 Envelope Construction

CPE builds the full cryptographic envelope:

```typescript
import { envelope } from '@ai-pip/core/cpe'

const cpeResult = envelope(islResult, secretKey)

// cpeResult.envelope contains:
// - payload: processed content (semantic)
// - metadata: security metadata
// - signature: cryptographic signature
// - lineage: full lineage
```

---

## 3. Components

### 3.1 Main Functions

#### Envelope Generation
- **`envelope(islResult: ISLResult, secretKey: string): CPEResult`** — Main envelope generation function. Creates the full cryptographic envelope with metadata, signature, and lineage.

### 3.2 Value Objects

#### Nonce
- **Type**: `Nonce` — Unique value to prevent replay attacks
- **Properties**:
  - `value: string` — Hexadecimal nonce value
- **Creation**: `createNonce(length?: number): Nonce` — Generates a unique nonce (default length: 16 bytes, min: 8, max: 64)
- **Helpers**:
  - `isValidNonce(value: string): boolean` — Validates that a string is a valid nonce (16–128 hex characters)
  - `equalsNonce(nonce1: Nonce, nonce2: Nonce): boolean` — Compares two nonces

#### Metadata
- **Type**: `CPEMetadata` — Immutable envelope security metadata
- **Properties**:
  - `timestamp: Timestamp` — Unix timestamp in milliseconds
  - `nonce: NonceValue` — Nonce value (string)
  - `protocolVersion: ProtocolVersion` — Protocol version
  - `previousSignatures?: { csl?: string; isl?: string }` — Optional signatures from previous layers
- **Constant**: `CURRENT_PROTOCOL_VERSION: ProtocolVersion = '0.1.4'` — Current protocol version
- **Creation**: `createMetadata(timestamp, nonce, protocolVersion?, previousSignatures?): CPEMetadata`
- **Helpers**:
  - `isValidMetadata(metadata: CPEMetadata): boolean` — Validates that metadata is valid

#### Signature
- **Type**: `SignatureVO` — Immutable cryptographic signature
- **Properties**:
  - `value: string` — Hexadecimal signature value (64 characters for HMAC-SHA256)
  - `algorithm: SignatureAlgorithm` — Algorithm used ('HMAC-SHA256')
- **Creation**: `createSignature(content: string, secretKey: string): SignatureVO` — Generates HMAC-SHA256 signature
- **Helpers**:
  - `verifySignature(content: string, signature: string, secretKey: string): boolean` — Verifies a signature
  - `isValidSignatureFormat(signature: string): boolean` — Validates signature format (64 hex characters)

### 3.3 Types

#### Basic Types
- **`ProtocolVersion`** — Protocol version: `string`
- **`Timestamp`** — Unix timestamp in milliseconds: `number`
- **`NonceValue`** — Nonce value: `string`
- **`SignatureAlgorithm`** — Signature algorithm: `'HMAC-SHA256'`
- **`Signature`** — Signature value: `string`

#### Interfaces
- **`CPEMetadata`** — Security metadata:
  ```typescript
  {
    timestamp: Timestamp
    nonce: NonceValue
    protocolVersion: ProtocolVersion
    previousSignatures?: {
      csl?: string
      isl?: string
    }
  }
  ```

- **`CPEEnvelope`** — Full cryptographic envelope:
  ```typescript
  {
    payload: {
      segments: readonly {
        id: string
        content: string
        trust: TrustLevel
        sanitizationLevel: 'minimal' | 'moderate' | 'aggressive'
      }[]
    }
    metadata: CPEMetadata
    signature: SignatureVO
    lineage: readonly LineageEntry[]
  }
  ```

- **`CPEResult`** — Envelope generation result:
  ```typescript
  {
    envelope: CPEEnvelope
    processingTimeMs?: number
  }
  ```

### 3.4 Exceptions

- **`EnvelopeError`** — Thrown when envelope generation fails (invalid secret key, invalid metadata, etc.)

---

## 4. Processing Flow

```
ISLResult (sanitized content)
    ↓
Generate metadata (timestamp, nonce, version)
    ↓
Prepare semantic payload
    ↓
Generate HMAC-SHA256 signature
    ↓
Update lineage with CPE entry
    ↓
Build cryptographic envelope
    ↓
CPEResult (envelope + metadata)
```

---

## 5. Guarantees

1. **Integrity**: Cryptographic signature guarantees content integrity
2. **Authenticity**: HMAC-SHA256 with secret key guarantees authenticity
3. **Non-repudiation**: Unique timestamp and nonce prevent replay attacks
4. **Traceability**: Full lineage preserved for audit

---

## 6. Usage Examples

### 6.1 Basic envelope generation

```typescript
import { envelope } from '@ai-pip/core'
import { sanitize } from '@ai-pip/core'
import { segment } from '@ai-pip/core'
import type { CPEResult } from '@ai-pip/core'

// 1. Process content through CSL and ISL
const cslResult = segment({
  content: 'User input here',
  source: 'UI',
  metadata: {}
})
const islResult = sanitize(cslResult)

// 2. Generate cryptographic envelope
const secretKey = 'your-secret-key' // Must be provided by the SDK
const cpeResult: CPEResult = envelope(islResult, secretKey)

// cpeResult.envelope contains:
// - payload: processed segments
// - metadata: timestamp, nonce, protocolVersion
// - signature: HMAC-SHA256 signature
// - lineage: full lineage
```

### 6.2 Working with Nonce

```typescript
import {
  createNonce,
  isValidNonce,
  equalsNonce
} from '@ai-pip/core'
import type { Nonce } from '@ai-pip/core'

// Create nonce with default length (16 bytes)
const nonce1: Nonce = createNonce()

// Create nonce with custom length
const nonce2: Nonce = createNonce(32) // 32 bytes

// Validate nonce
console.log(isValidNonce(nonce1.value)) // true
console.log(isValidNonce('invalid'))     // false

// Compare nonces
console.log(equalsNonce(nonce1, nonce2)) // false
console.log(equalsNonce(nonce1, nonce1))  // true
```

### 6.3 Metadata

```typescript
import {
  createMetadata,
  isValidMetadata,
  CURRENT_PROTOCOL_VERSION,
  createNonce
} from '@ai-pip/core'
import type { CPEMetadata } from '@ai-pip/core'

// Create basic metadata
const nonce = createNonce()
const metadata: CPEMetadata = createMetadata(
  Date.now(),
  nonce,
  CURRENT_PROTOCOL_VERSION
)

// Create metadata with previous signatures
const metadataWithSignatures: CPEMetadata = createMetadata(
  Date.now(),
  nonce,
  CURRENT_PROTOCOL_VERSION,
  {
    csl: 'csl-signature-123',
    isl: 'isl-signature-456'
  }
)

// Validate metadata
console.log(isValidMetadata(metadata)) // true
```

### 6.4 Cryptographic signature

```typescript
import {
  createSignature,
  verifySignature,
  isValidSignatureFormat
} from '@ai-pip/core'
import type { SignatureVO } from '@ai-pip/core'

const secretKey = 'my-secret-key'
const content = 'content to sign'

// Generate signature
const signature: SignatureVO = createSignature(content, secretKey)
console.log(signature.value)        // 'a1b2c3d4...' (64 hex characters)
console.log(signature.algorithm)   // 'HMAC-SHA256'

// Validate format
console.log(isValidSignatureFormat(signature.value)) // true
console.log(isValidSignatureFormat('invalid'))       // false

// Verify signature
const isValid = verifySignature(content, signature.value, secretKey)
console.log(isValid) // true

// Verify with different content
const isValid2 = verifySignature('different content', signature.value, secretKey)
console.log(isValid2) // false
```

### 6.5 Full pipeline: CSL → ISL → CPE

```typescript
import {
  segment,
  sanitize,
  envelope,
  createNonce,
  createMetadata,
  createSignature,
  verifySignature,
  CURRENT_PROTOCOL_VERSION
} from '@ai-pip/core'
import type {
  CSLResult,
  ISLResult,
  CPEResult,
  CPEEnvelope
} from '@ai-pip/core'

// 1. Segment content (CSL)
const cslResult: CSLResult = segment({
  content: 'System: You are helpful\n---\nUser: Hello',
  source: 'UI',
  metadata: {}
})

// 2. Sanitize content (ISL)
const islResult: ISLResult = sanitize(cslResult)

// 3. Generate cryptographic envelope (CPE)
const secretKey = 'my-secret-key-12345'
const cpeResult: CPEResult = envelope(islResult, secretKey)

// 4. Access the envelope
const env: CPEEnvelope = cpeResult.envelope

console.log('Payload segments:', env.payload.segments.length)
console.log('Metadata timestamp:', env.metadata.timestamp)
console.log('Metadata nonce:', env.metadata.nonce)
console.log('Metadata version:', env.metadata.protocolVersion)
console.log('Signature:', env.signature.value.substring(0, 20) + '...')
console.log('Lineage entries:', env.lineage.length)

// 5. Verify signature (in production, this would be done when receiving the envelope)
// Note: In production, you would serialize the content and metadata
// the same way as during signing
const isValid = verifySignature(
  'serialized-content-and-metadata',
  env.signature.value,
  secretKey
)
console.log('Signature valid:', isValid)
```

### 6.6 Envelope validation

```typescript
import {
  isValidMetadata,
  isValidNonce,
  isValidSignatureFormat,
  verifySignature
} from '@ai-pip/core'
import type { CPEEnvelope } from '@ai-pip/core'

function validateEnvelope(
  env: CPEEnvelope,
  secretKey: string,
  expectedContent: string
): boolean {
  // 1. Validate metadata
  if (!isValidMetadata(env.metadata)) {
    console.error('Invalid metadata')
    return false
  }

  // 2. Validate nonce
  if (!isValidNonce(env.metadata.nonce)) {
    console.error('Invalid nonce')
    return false
  }

  // 3. Validate signature format
  if (!isValidSignatureFormat(env.signature.value)) {
    console.error('Invalid signature format')
    return false
  }

  // 4. Verify signature
  if (!verifySignature(expectedContent, env.signature.value, secretKey)) {
    console.error('Invalid signature')
    return false
  }

  return true
}

// Usage
const isValid = validateEnvelope(cpeResult.envelope, secretKey, 'serialized-content')
console.log('Envelope is valid:', isValid)
```

---

## 7. Integration with ISL and ModelGateway

### 7.1 Input from ISL

CPE receives `ISLResult` with sanitized content and updated lineage.

### 7.2 Output to ModelGateway

CPE produces `CPEResult` containing the full cryptographic envelope ready to be sent to the model.

---

## 8. Security

### 8.1 Signature Algorithm

- **HMAC-SHA256**: Standard algorithm to guarantee integrity and authenticity
- **Secret Key**: Must be provided by the SDK or application
- **Format Validation**: Signature format verification (64 hex characters)

### 8.2 Replay Attack Prevention

- **Unique Nonce**: Each envelope has a unique nonce
- **Timestamp**: Timestamp validation to prevent replay attacks
- **Future Validation**: Future timestamps are rejected (with a 5-minute margin)

---

## 9. Core Limitations

The CPE core **does not include**:
- Envelope serialization (handled by the SDK)
- Envelope deserialization (handled by the SDK)
- Secret key management (handled by the SDK)
- Real-time timestamp validation (handled by the SDK)

These functionalities are implemented in the SDK or in the application.
