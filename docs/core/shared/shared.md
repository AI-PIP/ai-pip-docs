# Shared - Funciones Compartidas

> **Utilidades Compartidas** - Funciones puras compartidas entre todas las capas del protocolo AI-PIP

## 📋 Descripción General

La capa **Shared** proporciona funciones puras compartidas que son utilizadas por todas las capas del protocolo AI-PIP (CSL, ISL, CPE). Su función principal es el manejo del linaje (lineage) que permite rastrear el procesamiento de contenido a través de todas las capas.

### Principios Fundamentales

- **Pureza**: Todas las funciones son puras (sin efectos secundarios)
- **Inmutabilidad**: Los arrays de linaje son inmutables, las funciones retornan nuevos arrays
- **Simplicidad**: Solo funciones básicas de manejo de linaje
- **Trazabilidad**: Preserva el historial completo de procesamiento

## 🎯 Funcionalidades Principales

### 1. Manejo de Linaje

El linaje (lineage) es un registro inmutable del procesamiento de contenido a través de las diferentes capas. Cada entrada registra:
- **Step**: La capa que procesó el contenido (ej: 'CSL', 'ISL', 'CPE')
- **Timestamp**: Momento en que se procesó

### 2. Operaciones sobre Linaje

Las funciones de Shared permiten:
- Agregar entradas al linaje
- Filtrar entradas por step
- Obtener la última entrada

## 📦 Componentes

### Funciones Principales

#### Agregar Entradas
- **`addLineageEntry(lineage: readonly LineageEntry[], entry: LineageEntry): LineageEntry[]`** - Agrega una entrada de linaje a un array existente. Retorna un nuevo array inmutable.

- **`addLineageEntries(lineage: readonly LineageEntry[], entries: readonly LineageEntry[]): LineageEntry[]`** - Agrega múltiples entradas de linaje a un array existente. Retorna un nuevo array inmutable.

#### Filtrar y Consultar
- **`filterLineageByStep(lineage: readonly LineageEntry[], step: string): LineageEntry[]`** - Filtra entradas de linaje por step. Retorna un nuevo array con solo las entradas que coinciden con el step.

- **`getLastLineageEntry(lineage: readonly LineageEntry[]): LineageEntry | undefined`** - Obtiene la última entrada de linaje. Retorna `undefined` si el linaje está vacío.

### Tipos

#### LineageEntry
El tipo `LineageEntry` es importado desde CSL y tiene la siguiente estructura:

```typescript
{
  step: string        // Paso del procesamiento (ej: 'CSL', 'ISL', 'CPE')
  timestamp: number   // Timestamp Unix en milisegundos
}
```

## 🔄 Flujo de Procesamiento

El linaje fluye a través de todas las capas:

```
CSL → Inicializa linaje con entrada 'CSL'
  ↓
ISL → Agrega entrada 'ISL' al linaje
  ↓
CPE → Agrega entrada 'CPE' al linaje
  ↓
Linaje completo con todas las capas
```

## ✅ Garantías

1. **Inmutabilidad**: Todas las funciones retornan nuevos arrays, nunca modifican el linaje original
2. **Pureza**: Sin efectos secundarios, funciones deterministas
3. **Preservación**: El linaje completo se preserva a través de todas las capas
4. **Trazabilidad**: Cada paso del procesamiento queda registrado

## 📝 Ejemplos de Uso

### Ejemplo Básico: Agregar Entrada

```typescript
import { addLineageEntry, createLineageEntry } from '@ai-pip/core'

// Linaje inicial (desde CSL)
const initialLineage = [
  { step: 'CSL', timestamp: Date.now() }
]

// Agregar entrada de ISL
const islEntry = createLineageEntry('ISL', Date.now())
const updatedLineage = addLineageEntry(initialLineage, islEntry)

// updatedLineage ahora contiene:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... }
// ]
```

### Ejemplo: Agregar Múltiples Entradas

```typescript
import { addLineageEntries, createLineageEntry } from '@ai-pip/core'

const initialLineage = [
  { step: 'CSL', timestamp: Date.now() }
]

// Crear múltiples entradas
const newEntries = [
  createLineageEntry('ISL', Date.now()),
  createLineageEntry('CPE', Date.now())
]

// Agregar todas las entradas de una vez
const fullLineage = addLineageEntries(initialLineage, newEntries)

// fullLineage ahora contiene:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... },
//   { step: 'CPE', timestamp: ... }
// ]
```

### Ejemplo: Filtrar por Step

```typescript
import { filterLineageByStep } from '@ai-pip/core'

const lineage = [
  { step: 'CSL', timestamp: 1000 },
  { step: 'ISL', timestamp: 2000 },
  { step: 'CPE', timestamp: 3000 },
  { step: 'ISL', timestamp: 4000 }
]

// Filtrar solo entradas de ISL
const islEntries = filterLineageByStep(lineage, 'ISL')

// islEntries contiene:
// [
//   { step: 'ISL', timestamp: 2000 },
//   { step: 'ISL', timestamp: 4000 }
// ]
```

### Ejemplo: Obtener Última Entrada

```typescript
import { getLastLineageEntry } from '@ai-pip/core'

const lineage = [
  { step: 'CSL', timestamp: 1000 },
  { step: 'ISL', timestamp: 2000 },
  { step: 'CPE', timestamp: 3000 }
]

// Obtener última entrada
const lastEntry = getLastLineageEntry(lineage)

// lastEntry es: { step: 'CPE', timestamp: 3000 }

// Con linaje vacío
const emptyLineage: LineageEntry[] = []
const noEntry = getLastLineageEntry(emptyLineage)
// noEntry es: undefined
```

### Ejemplo Completo: Pipeline con Linaje

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

// 1. CSL inicializa el linaje
const cslResult = segment({
  content: 'Test content',
  source: 'UI',
  metadata: {}
})

// El linaje inicial contiene: [{ step: 'CSL', timestamp: ... }]

// 2. ISL agrega entrada al linaje (automáticamente en sanitize)
const islResult = sanitize(cslResult)

// El linaje ahora contiene:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... }
// ]

// 3. CPE agrega entrada al linaje (automáticamente en envelope)
const cpeResult = envelope(islResult, 'secret-key')

// El linaje final contiene:
// [
//   { step: 'CSL', timestamp: ... },
//   { step: 'ISL', timestamp: ... },
//   { step: 'CPE', timestamp: ... }
// ]

// 4. Consultar el linaje
const fullLineage = cpeResult.envelope.lineage

// Obtener última entrada
const lastEntry = getLastLineageEntry(fullLineage)
console.log('Last step:', lastEntry?.step) // 'CPE'

// Filtrar entradas de una capa específica
const cslEntries = filterLineageByStep(fullLineage, 'CSL')
console.log('CSL entries:', cslEntries.length) // 1

// Agregar entrada personalizada (ej: para auditoría)
const auditEntry = createLineageEntry('AUDIT', Date.now())
const lineageWithAudit = addLineageEntry(fullLineage, auditEntry)
```

### Ejemplo: Auditoría y Análisis

```typescript
import {
  filterLineageByStep,
  getLastLineageEntry
} from '@ai-pip/core'
import type { LineageEntry } from '@ai-pip/core'

function analyzeLineage(lineage: readonly LineageEntry[]) {
  // Obtener última entrada
  const lastEntry = getLastLineageEntry(lineage)
  console.log('Last processing step:', lastEntry?.step)

  // Contar entradas por step
  const steps = ['CSL', 'ISL', 'CPE'] as const
  steps.forEach(step => {
    const entries = filterLineageByStep(lineage, step)
    console.log(`${step} entries: ${entries.length}`)
  })

  // Calcular tiempo total de procesamiento
  if (lineage.length >= 2) {
    const first = lineage[0]
    const last = lastEntry
    if (first && last) {
      const totalTime = last.timestamp - first.timestamp
      console.log(`Total processing time: ${totalTime}ms`)
    }
  }
}

// Uso
const lineage = cpeResult.envelope.lineage
analyzeLineage(lineage)
```

## 🔗 Integración con Otras Capas

### CSL
- CSL inicializa el linaje con `initLineage()` que crea la primera entrada con step 'CSL'.

### ISL
- ISL agrega una entrada al linaje con step 'ISL' durante la sanitización.

### CPE
- CPE agrega una entrada al linaje con step 'CPE' durante la generación del envelope.

## ⚠️ Limitaciones del Core

El core de Shared **NO incluye**:
- Análisis avanzado de linaje (va al SDK)
- Estadísticas de procesamiento (van al SDK)
- Serialización de linaje (va al SDK)
- Búsqueda compleja en linaje (va al SDK)

Estas funcionalidades se implementan en el SDK o en herramientas de auditoría.

## 📚 Referencias

- **LineageEntry**: Disponible desde `@ai-pip/core`
- **createLineageEntry**: Disponible desde `@ai-pip/core`
- **initLineage**: Disponible desde `@ai-pip/core`

> **Nota**: Las funciones de Shared están disponibles desde el entry point principal `@ai-pip/core`, no desde un subpath específico.

