# CPE - Cryptographic Prompt Envelope

> **Envoltorio Criptográfico de Prompts** - Tercera capa del protocolo AI-PIP

## 📋 Descripción General

La **Cryptographic Prompt Envelope (CPE)** es la tercera capa del protocolo AI-PIP. Su función principal es generar un envoltorio criptográfico que garantiza la integridad y autenticidad del prompt procesado por las capas anteriores.

### Principios Fundamentales

- **Integridad Criptográfica**: Firma HMAC-SHA256 del contenido
- **No Repudio**: Timestamp y nonce únicos
- **Trazabilidad Completa**: Linaje completo preservado
- **Metadata de Seguridad**: Información de auditoría

## 🎯 Funcionalidades Principales

### 1. Generación de Metadata de Seguridad

CPE genera metadata que incluye:

- **Timestamp**: Momento de creación del envelope
- **Nonce**: Valor único para prevenir ataques de replay
- **Protocol Version**: Versión del protocolo AI-PIP
- **Previous Signatures**: Firmas opcionales de capas anteriores (CSL, ISL)

```typescript
import { createMetadata, CURRENT_PROTOCOL_VERSION } from '@ai-pip/core/cpe'

const metadata = createMetadata(
  Date.now(),
  nonce,
  CURRENT_PROTOCOL_VERSION,
  {
    csl: 'csl-signature-123',  // Opcional
    isl: 'isl-signature-456'   // Opcional
  }
)
```

### 2. Firma Criptográfica HMAC-SHA256

CPE genera una firma criptográfica del contenido usando HMAC-SHA256:

```typescript
import { createSignature, verifySignature } from '@ai-pip/core/cpe'

// Generar firma
const signature = createSignature(
  signableContent,
  secretKey
)

// Verificar firma
const isValid = verifySignature(
  content,
  signature.value,
  secretKey
)
```

### 3. Construcción del Envelope

CPE construye el envelope criptográfico completo:

```typescript
import { envelope } from '@ai-pip/core/cpe'

const cpeResult = envelope(islResult, secretKey)

// cpeResult.envelope contiene:
// - payload: contenido procesado (semántico)
// - metadata: metadata de seguridad
// - signature: firma criptográfica
// - lineage: linaje completo
```

## 📦 Componentes

### Funciones Principales

#### Generación de Envelope
- **`envelope(islResult: ISLResult, secretKey: string): CPEResult`** - Función principal de generación de envelope. Crea el envoltorio criptográfico completo con metadata, firma y linaje.

### Value Objects

#### Nonce
- **Tipo**: `Nonce` - Valor único para prevenir ataques de replay
- **Propiedades**:
  - `value: string` - Valor hexadecimal del nonce
- **Creación**: `createNonce(length?: number): Nonce` - Genera un nonce único (longitud por defecto: 16 bytes, mínimo: 8, máximo: 64)
- **Utilidades**:
  - `isValidNonce(value: string): boolean` - Valida que un string sea un nonce válido (16-128 caracteres hex)
  - `equalsNonce(nonce1: Nonce, nonce2: Nonce): boolean` - Compara dos nonces

#### Metadata
- **Tipo**: `CPEMetadata` - Metadata de seguridad del envelope inmutable
- **Propiedades**:
  - `timestamp: Timestamp` - Timestamp Unix en milisegundos
  - `nonce: NonceValue` - Valor del nonce (string)
  - `protocolVersion: ProtocolVersion` - Versión del protocolo
  - `previousSignatures?: { csl?: string; isl?: string }` - Firmas opcionales de capas anteriores
- **Constante**: `CURRENT_PROTOCOL_VERSION: ProtocolVersion = '0.1.4'` - Versión actual del protocolo
- **Creación**: `createMetadata(timestamp, nonce, protocolVersion?, previousSignatures?): CPEMetadata`
- **Utilidades**:
  - `isValidMetadata(metadata: CPEMetadata): boolean` - Valida que la metadata sea válida

#### Signature
- **Tipo**: `SignatureVO` - Firma criptográfica inmutable
- **Propiedades**:
  - `value: string` - Valor hexadecimal de la firma (64 caracteres para HMAC-SHA256)
  - `algorithm: SignatureAlgorithm` - Algoritmo usado ('HMAC-SHA256')
- **Creación**: `createSignature(content: string, secretKey: string): SignatureVO` - Genera firma HMAC-SHA256
- **Utilidades**:
  - `verifySignature(content: string, signature: string, secretKey: string): boolean` - Verifica una firma
  - `isValidSignatureFormat(signature: string): boolean` - Valida el formato de una firma (64 caracteres hex)

### Tipos

#### Tipos Básicos
- **`ProtocolVersion`** - Versión del protocolo: `string`
- **`Timestamp`** - Timestamp Unix en milisegundos: `number`
- **`NonceValue`** - Valor del nonce: `string`
- **`SignatureAlgorithm`** - Algoritmo de firma: `'HMAC-SHA256'`
- **`Signature`** - Valor de la firma: `string`

#### Interfaces
- **`CPEMetadata`** - Metadata de seguridad:
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

- **`CPEEvelope`** - Envoltorio criptográfico completo:
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

- **`CPEResult`** - Resultado de generación del envelope:
  ```typescript
  {
    envelope: CPEEvelope
    processingTimeMs?: number
  }
  ```

### Excepciones

- **`EnvelopeError`** - Lanzada cuando la generación del envelope falla (clave secreta inválida, metadata inválida, etc.)

## 🔄 Flujo de Procesamiento

```
ISLResult (contenido sanitizado)
    ↓
Generar metadata (timestamp, nonce, versión)
    ↓
Preparar payload semántico
    ↓
Generar firma HMAC-SHA256
    ↓
Actualizar linaje con entrada CPE
    ↓
Construir envelope criptográfico
    ↓
CPEResult (envelope + metadata)
```

## ✅ Garantías

1. **Integridad**: Firma criptográfica garantiza integridad del contenido
2. **Autenticidad**: HMAC-SHA256 con clave secreta garantiza autenticidad
3. **No Repudio**: Timestamp y nonce únicos previenen replay attacks
4. **Trazabilidad**: Linaje completo preservado para auditoría

## 📝 Ejemplos de Uso

### Ejemplo Básico: Generación de Envelope

```typescript
import { envelope } from '@ai-pip/core'
import { sanitize } from '@ai-pip/core'
import { segment } from '@ai-pip/core'
import type { CPEResult } from '@ai-pip/core'

// 1. Procesar contenido a través de CSL e ISL
const cslResult = segment({
  content: 'User input here',
  source: 'UI',
  metadata: {}
})
const islResult = sanitize(cslResult)

// 2. Generar envelope criptográfico
const secretKey = 'your-secret-key' // Debe ser proporcionado por el SDK
const cpeResult: CPEResult = envelope(islResult, secretKey)

// cpeResult.envelope contiene:
// - payload: segmentos procesados
// - metadata: timestamp, nonce, protocolVersion
// - signature: firma HMAC-SHA256
// - lineage: linaje completo
```

### Ejemplo: Trabajar con Nonce

```typescript
import {
  createNonce,
  isValidNonce,
  equalsNonce
} from '@ai-pip/core'
import type { Nonce } from '@ai-pip/core'

// Crear nonce con longitud por defecto (16 bytes)
const nonce1: Nonce = createNonce()

// Crear nonce con longitud personalizada
const nonce2: Nonce = createNonce(32) // 32 bytes

// Validar nonce
console.log(isValidNonce(nonce1.value)) // true
console.log(isValidNonce('invalid'))     // false

// Comparar nonces
console.log(equalsNonce(nonce1, nonce2)) // false
console.log(equalsNonce(nonce1, nonce1))  // true
```

### Ejemplo: Metadata

```typescript
import {
  createMetadata,
  isValidMetadata,
  CURRENT_PROTOCOL_VERSION,
  createNonce
} from '@ai-pip/core'
import type { CPEMetadata } from '@ai-pip/core'

// Crear metadata básica
const nonce = createNonce()
const metadata: CPEMetadata = createMetadata(
  Date.now(),
  nonce,
  CURRENT_PROTOCOL_VERSION
)

// Crear metadata con firmas previas
const metadataWithSignatures: CPEMetadata = createMetadata(
  Date.now(),
  nonce,
  CURRENT_PROTOCOL_VERSION,
  {
    csl: 'csl-signature-123',
    isl: 'isl-signature-456'
  }
)

// Validar metadata
console.log(isValidMetadata(metadata)) // true
```

### Ejemplo: Firma Criptográfica

```typescript
import {
  createSignature,
  verifySignature,
  isValidSignatureFormat
} from '@ai-pip/core'
import type { SignatureVO } from '@ai-pip/core'

const secretKey = 'my-secret-key'
const content = 'content to sign'

// Generar firma
const signature: SignatureVO = createSignature(content, secretKey)
console.log(signature.value)        // 'a1b2c3d4...' (64 caracteres hex)
console.log(signature.algorithm)     // 'HMAC-SHA256'

// Validar formato
console.log(isValidSignatureFormat(signature.value)) // true
console.log(isValidSignatureFormat('invalid'))       // false

// Verificar firma
const isValid = verifySignature(content, signature.value, secretKey)
console.log(isValid) // true

// Verificar con contenido diferente
const isValid2 = verifySignature('different content', signature.value, secretKey)
console.log(isValid2) // false
```

### Ejemplo Completo: Pipeline CSL → ISL → CPE

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
  CPEEvelope
} from '@ai-pip/core'

// 1. Segmentar contenido (CSL)
const cslResult: CSLResult = segment({
  content: 'System: You are helpful\n---\nUser: Hello',
  source: 'UI',
  metadata: {}
})

// 2. Sanitizar contenido (ISL)
const islResult: ISLResult = sanitize(cslResult)

// 3. Generar envelope criptográfico (CPE)
const secretKey = 'my-secret-key-12345'
const cpeResult: CPEResult = envelope(islResult, secretKey)

// 4. Acceder al envelope
const envelope: CPEEvelope = cpeResult.envelope

console.log('Payload segments:', envelope.payload.segments.length)
console.log('Metadata timestamp:', envelope.metadata.timestamp)
console.log('Metadata nonce:', envelope.metadata.nonce)
console.log('Metadata version:', envelope.metadata.protocolVersion)
console.log('Signature:', envelope.signature.value.substring(0, 20) + '...')
console.log('Lineage entries:', envelope.lineage.length)

// 5. Verificar firma (en producción, esto se haría al recibir el envelope)
// Nota: En producción, necesitarías serializar el contenido y metadata
// de la misma manera que se hizo durante la firma
const isValid = verifySignature(
  'serialized-content-and-metadata',
  envelope.signature.value,
  secretKey
)
console.log('Signature valid:', isValid)
```

### Ejemplo: Validación de Envelope

```typescript
import {
  isValidMetadata,
  isValidNonce,
  isValidSignatureFormat,
  verifySignature
} from '@ai-pip/core'
import type { CPEEvelope } from '@ai-pip/core'

function validateEnvelope(
  envelope: CPEEvelope,
  secretKey: string,
  expectedContent: string
): boolean {
  // 1. Validar metadata
  if (!isValidMetadata(envelope.metadata)) {
    console.error('Invalid metadata')
    return false
  }

  // 2. Validar nonce
  if (!isValidNonce(envelope.metadata.nonce)) {
    console.error('Invalid nonce')
    return false
  }

  // 3. Validar formato de firma
  if (!isValidSignatureFormat(envelope.signature.value)) {
    console.error('Invalid signature format')
    return false
  }

  // 4. Verificar firma
  if (!verifySignature(expectedContent, envelope.signature.value, secretKey)) {
    console.error('Invalid signature')
    return false
  }

  return true
}

// Uso
const isValid = validateEnvelope(cpeResult.envelope, secretKey, 'serialized-content')
console.log('Envelope is valid:', isValid)
```

## 🔗 Integración con ISL y ModelGateway

### Entrada desde ISL

CPE recibe `ISLResult` con contenido sanitizado y linaje actualizado.

### Salida hacia ModelGateway

CPE produce `CPEResult` que contiene el envelope criptográfico completo listo para ser enviado al modelo.

## 🔐 Seguridad

### Algoritmo de Firma

- **HMAC-SHA256**: Algoritmo estándar para garantizar integridad y autenticidad
- **Clave Secreta**: Debe ser proporcionada por el SDK o aplicación
- **Validación de Formato**: Verificación de formato de firma (64 caracteres hex)

### Prevención de Replay Attacks

- **Nonce Único**: Cada envelope tiene un nonce único
- **Timestamp**: Validación de timestamp para prevenir ataques de replay
- **Validación de Futuro**: Timestamps del futuro son rechazados (con margen de 5 minutos)

## ⚠️ Limitaciones del Core

El core de CPE **NO incluye**:
- Serialización del envelope (va al SDK)
- Deserialización del envelope (va al SDK)
- Gestión de claves secretas (va al SDK)
- Validación de timestamps en tiempo real (va al SDK)

Estas funcionalidades se implementan en el SDK o en la aplicación.

