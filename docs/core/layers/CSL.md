# CSL - Context Segmentation Layer

> **Capa de Segmentación de Contexto** - Primera capa del protocolo AI-PIP

## 📋 Descripción General

La **Context Segmentation Layer (CSL)** es la primera capa del protocolo AI-PIP. Su función principal es segmentar el contenido de entrada en segmentos semánticos y clasificarlos según su nivel de confianza basándose únicamente en su origen.

### Principios Fundamentales

- **Determinismo**: Mismo origen → mismo nivel de confianza, siempre
- **Pureza**: Funciones sin efectos secundarios
- **Inmutabilidad**: Todos los objetos son inmutables
- **Preservación**: El contenido original nunca se pierde

## 🎯 Funcionalidades Principales

### 1. Segmentación de Contenido

La función `segment()` divide el contenido de entrada en segmentos semánticos basándose en reglas de contexto (saltos de línea, delimitadores, etc.).

```typescript
import { segment } from '@ai-pip/core/csl'

const result = segment({
  content: 'Hello\nWorld\n---\nUser input',
  source: 'UI',
  metadata: {}
})

// result.segments contiene los segmentos clasificados
```

### 2. Clasificación por Origen

CSL clasifica cada segmento según su origen (`source`) o tipo de origen (`origin`):

#### Clasificación por Source

- **`UI`** → **TC** (Trusted Content)
- **`SYSTEM`** → **TC** (Trusted Content)
- **`DOM`** → **STC** (Semi-Trusted Content)
- **`API`** → **UC** (Untrusted Content)

#### Clasificación por Origin

- **`SYSTEM_GENERATED`** → **TC**
- **`DOM_VISIBLE`** → **STC**
- **`DOM_HIDDEN`** → **UC**
- **`USER`** → **UC**
- **`UNKNOWN`** → **UC** (fail-secure)

### 3. Inicialización de Linaje

Cada segmento recibe una entrada inicial de linaje que registra:
- **Step**: `'CSL'`
- **Timestamp**: Momento de creación

## 📦 Componentes

### Funciones Principales

#### Segmentación
- **`segment(input: CSLInput): CSLResult`** - Función principal de segmentación. Divide el contenido en segmentos semánticos y los clasifica según su origen.

#### Clasificación
- **`classifySource(source: Source): TrustLevel`** - Clasifica un source y retorna su TrustLevel. Determinista: mismo source → mismo trust level.
- **`classifyOrigin(origin: Origin): TrustLevel`** - Clasifica un Origin y retorna su TrustLevel basándose en el originMap.

#### Linaje
- **`initLineage(segment: CSLSegment): LineageEntry[]`** - Inicializa el linaje de un segmento con una entrada CSL.
- **`createLineageEntry(step: string, timestamp: number): LineageEntry`** - Crea una entrada de linaje con step y timestamp.

#### Utilidades
- **`generateId(): string`** - Genera un ID único para un segmento (formato: `seg-{timestamp}-{random}`).
- **`splitByContextRules(content: string): string[]`** - Divide contenido por reglas de contexto (saltos de línea). Función pura de segmentación básica.

### Value Objects

#### TrustLevel
- **Tipo**: `TrustLevel` - Nivel de confianza inmutable (TC, STC, UC)
- **Creación**: `createTrustLevel(value: TrustLevelType): TrustLevel`
- **Utilidades**:
  - `isTrusted(trust: TrustLevel): boolean` - Verifica si es TC
  - `isSemiTrusted(trust: TrustLevel): boolean` - Verifica si es STC
  - `isUntrusted(trust: TrustLevel): boolean` - Verifica si es UC

#### Origin
- **Tipo**: `Origin` - Origen del contenido inmutable
- **Creación**: `createOrigin(type: OriginType): Origin`
- **Utilidades**:
  - `isDom(origin: Origin): boolean` - Verifica si es origen DOM
  - `isUser(origin: Origin): boolean` - Verifica si es USER
  - `isSystem(origin: Origin): boolean` - Verifica si es SYSTEM_GENERATED
  - `isInjected(origin: Origin): boolean` - Verifica si es SCRIPT_INJECTED
  - `isUnknown(origin: Origin): boolean` - Verifica si es UNKNOWN
  - `isNetworkFetched(origin: Origin): boolean` - Verifica si es NETWORK_FETCHED
  - `isExternal(origin: Origin): boolean` - Verifica si es externo (NETWORK_FETCHED o SCRIPT_INJECTED)

#### LineageEntry
- **Tipo**: `LineageEntry` - Entrada de linaje inmutable
- **Propiedades**:
  - `step: string` - Paso del procesamiento (ej: 'CSL', 'ISL', 'CPE')
  - `timestamp: number` - Timestamp Unix en milisegundos
- **Creación**: `createLineageEntry(step: string, timestamp: number): LineageEntry`

#### ContentHash
- **Tipo**: `ContentHash` - Hash del contenido inmutable
- **Propiedades**:
  - `value: string` - Valor hexadecimal del hash
  - `algorithm: HashAlgorithm` - Algoritmo usado ('sha256' | 'sha512')
- **Creación**: `createContentHash(value: string, algorithm?: HashAlgorithm): ContentHash`
- **Utilidades**:
  - `isSha256(hash: ContentHash): boolean` - Verifica si usa SHA256
  - `isSha512(hash: ContentHash): boolean` - Verifica si usa SHA512

#### Origin Map
- **Constante**: `originMap: Map<OriginType, TrustLevelType>` - Mapeo determinista de OriginType a TrustLevelType
- **Validación**: `validateOriginMap(): void` - Valida que todos los OriginType estén mapeados

### Tipos

#### Enums
- **`OriginType`** - Tipo de origen del contenido:
  - `USER` - Entrada directa del usuario
  - `DOM_VISIBLE` - Contenido DOM visible
  - `DOM_HIDDEN` - Contenido DOM oculto
  - `DOM_ATTRIBUTE` - Atributos DOM (data-*, aria-*)
  - `SCRIPT_INJECTED` - Contenido inyectado por scripts
  - `NETWORK_FETCHED` - Contenido desde red/API
  - `SYSTEM_GENERATED` - Contenido generado por el sistema
  - `UNKNOWN` - Origen desconocido

- **`TrustLevelType`** - Nivel de confianza:
  - `TC` - Trusted Content (Contenido confiable)
  - `STC` - Semi-Trusted Content (Contenido semi-confiable)
  - `UC` - Untrusted Content (Contenido no confiable)

#### Tipos Básicos
- **`Source`** - Source del contenido: `'DOM' | 'UI' | 'SYSTEM' | 'API'`
- **`HashAlgorithm`** - Algoritmo de hash: `'sha256' | 'sha512'`

#### Interfaces
- **`CSLInput`** - Input para segmentación:
  ```typescript
  {
    content: string
    source: Source
    metadata?: Record<string, unknown>
  }
  ```

- **`CSLSegment`** - Segmento clasificado:
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

- **`CSLResult`** - Resultado de segmentación:
  ```typescript
  {
    segments: readonly CSLSegment[]
    lineage: readonly LineageEntry[]
    processingTimeMs?: number
  }
  ```

### Excepciones

- **`ClassificationError`** - Lanzada cuando la clasificación falla (origen no mapeado, etc.)
- **`SegmentationError`** - Lanzada cuando la segmentación falla (contenido inválido, etc.)

## 🔄 Flujo de Procesamiento

```
Input (content + source)
    ↓
Segmentación (splitByContextRules)
    ↓
Clasificación (classifySource/classifyOrigin)
    ↓
Inicialización de Linaje (initLineage)
    ↓
CSLResult (segmentos + linaje)
```

## ✅ Garantías

1. **Integridad**: El contenido original se preserva en cada segmento
2. **Determinismo**: Mismo input → mismo output
3. **Trazabilidad**: Todo segmento tiene linaje inicializado
4. **Fail-Secure**: Orígenes desconocidos se clasifican como UC

## 📝 Ejemplos de Uso

### Ejemplo Básico: Segmentación

```typescript
import { segment } from '@ai-pip/core'

// Segmentar contenido
const result = segment({
  content: 'System prompt\n---\nUser: Hello',
  source: 'UI',
  metadata: { sessionId: '123' }
})

// result.segments contiene los segmentos clasificados
// result.lineage contiene el linaje inicial
```

### Ejemplo: Clasificación de Sources

```typescript
import { classifySource, isTrusted, isSemiTrusted, isUntrusted } from '@ai-pip/core'

// Clasificar diferentes sources
const uiTrust = classifySource('UI')        // { value: 'TC' }
const domTrust = classifySource('DOM')      // { value: 'STC' }
const apiTrust = classifySource('API')      // { value: 'UC' }

// Verificar niveles de confianza
console.log(isTrusted(uiTrust))      // true
console.log(isSemiTrusted(domTrust)) // true
console.log(isUntrusted(apiTrust))   // true
```

### Ejemplo: Trabajar con Origins

```typescript
import {
  createOrigin,
  OriginType,
  classifyOrigin,
  isDom,
  isSystem,
  isExternal
} from '@ai-pip/core'

// Crear un Origin
const origin = createOrigin(OriginType.DOM_VISIBLE)

// Clasificar el Origin
const trust = classifyOrigin(origin) // { value: 'STC' }

// Verificar tipo de origen
console.log(isDom(origin))      // true
console.log(isSystem(origin))   // false
console.log(isExternal(origin)) // false
```

### Ejemplo: ContentHash

```typescript
import { createContentHash, isSha256, isSha512 } from '@ai-pip/core'

// Crear hash SHA256
const hash256 = createContentHash(
  'a1b2c3d4e5f6...', // 64 caracteres hex
  'sha256'
)

// Crear hash SHA512 (por defecto es sha256)
const hash512 = createContentHash(
  'a1b2c3d4e5f6...', // 128 caracteres hex
  'sha512'
)

// Verificar algoritmo
console.log(isSha256(hash256)) // true
console.log(isSha512(hash512)) // true
```

### Ejemplo: Linaje

```typescript
import { createLineageEntry, initLineage } from '@ai-pip/core'

// Crear entrada de linaje manualmente
const entry = createLineageEntry('CSL', Date.now())

// Inicializar linaje para un segmento
const segment = {
  id: 'seg-123',
  content: 'Test',
  source: 'UI' as const,
  trust: { value: 'TC' as const },
  lineage: []
}

const lineage = initLineage(segment)
// Retorna: [{ step: 'CSL', timestamp: ... }]
```

### Ejemplo: Utilidades

```typescript
import { generateId, splitByContextRules } from '@ai-pip/core'

// Generar ID único
const id = generateId() // 'seg-1234567890-abc123'

// Dividir contenido por reglas de contexto
const segments = splitByContextRules('Line 1\nLine 2\n\nLine 3')
// Retorna: ['Line 1', 'Line 2', 'Line 3']
```

### Ejemplo Completo: Pipeline CSL

```typescript
import {
  segment,
  classifySource,
  createOrigin,
  OriginType,
  isTrusted
} from '@ai-pip/core'
import type { CSLResult, CSLSegment } from '@ai-pip/core'

// 1. Segmentar contenido
const cslResult: CSLResult = segment({
  content: 'System: You are a helpful assistant\n---\nUser: Hello',
  source: 'UI',
  metadata: { sessionId: 'abc123' }
})

// 2. Procesar cada segmento
cslResult.segments.forEach((seg: CSLSegment) => {
  console.log(`Segment ID: ${seg.id}`)
  console.log(`Content: ${seg.content}`)
  console.log(`Trust Level: ${seg.trust.value}`)
  console.log(`Is Trusted: ${isTrusted(seg.trust)}`)
  console.log(`Lineage entries: ${seg.lineage.length}`)
})

// 3. Clasificar un source específico
const trust = classifySource('API')
console.log(`API trust level: ${trust.value}`) // 'UC'
```

## 🔗 Integración con ISL

CSL pasa su resultado a ISL mediante el contrato:

```typescript
CSLResult {
  segments: CSLSegment[]  // Segmentos clasificados
  lineage: LineageEntry[] // Linaje inicial
}
```

ISL recibe este resultado y aplica sanitización según el `trust` level de cada segmento.

## ⚠️ Limitaciones del Core

El core de CSL **NO incluye**:
- Normalización agresiva de contenido
- Detección de prompt injection (va a ISL)
- Políticas de seguridad (van a ISL)
- Servicios con estado (van al SDK)

Estas funcionalidades se implementan en capas superiores o en el SDK.

