# Roadmap - AI-PIP Protocol

> Development and evolution plan for the AI-PIP protocol

**Last Updated**: 2026-01-01

---

## 📊 Current Status

### ✅ Phase 1: Core Layers

**Status**: Completed

#### Implemented Layers ✅

- **CSL (Context Segmentation Layer)**: ✅ Completed
  - Content segmentation
  - Classification by origin (source/origin)
  - Lineage initialization
  - Immutable value objects
  - Pure functions

- **ISL (Instruction Sanitization Layer)**: ✅ Completed
  - Differentiated sanitization by trust level
  - Prompt injection detection (basic structure)
  - Security policies (PolicyRule)
  - Anomaly scoring
  - Immutable value objects

- **CPE (Cryptographic Prompt Envelope)**: ✅ Completed
  - Security metadata generation
  - HMAC-SHA256 cryptographic signature
  - Envelope construction
  - Complete lineage preservation

- **Shared (Shared Features)**: ✅ Completed
  - Global and incremental lineage
  - Pure functions shared between layers
  - Complete processing auditing

---

## 🗺️ Development Phases

### 🔵 Phase 1: Protocol Core Layers

**Objective**: Implement all core layers of the AI-PIP protocol

**Status**: 100% completed (optimizations pending)

#### Completed Tasks ✅

- [x] CSL - Context Segmentation Layer
- [x] ISL - Instruction Sanitization Layer
- [x] CPE - Cryptographic Prompt Envelope
- [x] Shared - Shared features (global lineage)
- [x] Immutable value objects
- [x] Pure functions without side effects
- [x] Global and incremental lineage system
- [x] Unit tests (>80% coverage)
- [x] Complete layer documentation
- [x] Protocol whitepaper

#### Pending Tasks 🔄

- [ ] Reduce core size to make it lighter
- [ ] New pure functions to strengthen SDKs
- [ ] Improve code quality (refactoring, optimizations)
- [ ] Performance optimization
- [ ] Technical documentation improvements

**Estimation**: Q1-Q2 2026

---

### 🟢 Phase 2: SDK Implementation (TypeScript/JavaScript)

**Objective**: Implement functional and auditable beta SDK of the AI-PIP protocol for real environments

**Status**: Not started

#### SDK Beta - Main Objectives

Provide a functional and auditable implementation of the AI-PIP protocol capable of:
- Detecting, locating, and mitigating prompt injection
- Identifying attack surfaces in real environments (especially browser)
- Integrating complete DOM scanning
- Providing precise lineage per node
- Implementing configurable policies
- Supporting agent flows
- Enabling prevention, visualization, and exact risk auditing before interaction with AI models

#### TypeScript/JavaScript SDK Tasks

- [ ] Hexagonal architecture
- [ ] Classes and states for developer use
- [ ] Complete DOM scanning and content detection
- [ ] Precise lineage per DOM node
- [ ] Envelope serialization/deserialization
- [ ] Secret key management
- [ ] Configurable security policies
- [ ] Browser integration (extensions, injectables)
- [ ] Risk visualization and auditing
- [ ] Integrated agent flows
- [ ] Documentation and examples
- [ ] Integration tests in real environments

#### Python SDK

**Status**: Not planned until TypeScript/JavaScript SDK is completed

**Estimation**: Q3-Q4 2026 (after completing TypeScript SDK)

---

### 🟡 Phase 3: Integration and Testing in Real Environments

**Objective**: Integrate the protocol in real environments to validate its effectiveness and robustness

**Status**: Not started

#### Integration Tasks

- [ ] Integration with real browsers (Chrome, Firefox, Safari)
- [ ] Testing in real web applications
- [ ] Validation of prompt injection detection in real cases
- [ ] Performance testing in production
- [ ] Lineage validation in complex scenarios
- [ ] Configurable policy testing
- [ ] Integration with real AI agents
- [ ] Security auditing in real environments
- [ ] Metrics collection and feedback
- [ ] Optimization based on real results

**Estimation**: Q3-Q4 2026

---

### 🟠 Phase 4: Extensibility and Ecosystem

**Objective**: Create an ecosystem around the AI-PIP protocol and integrate with MCP agents

**Status**: Not started

#### Reference Implementations

- [ ] **SDK-browser**
  - SDK / browser extension based on AI-PIP
  - Implements CSL / ISL / CPE using the official SDK
  - Hidden context detection in DOM
  - Integration with browser agents
  - Action blocking through AAL (when available)
  - Use case: secure AI-assisted web navigation

#### Planned Tasks

- [ ] Plugin system
- [ ] Integrations with AI frameworks
- [ ] **Integration with MCP agents (Model Context Protocol)**
  - [ ] MCP adapter for AI-PIP
  - [ ] MCP context protection with CSL/ISL/CPE
  - [ ] MCP tool validation
  - [ ] MCP agent response sanitization
  - [ ] Integration with MCP servers
  - [ ] MCP integration documentation
- [ ] SDK improvements based on feedback
- [ ] Result caching
- [ ] Advanced performance optimization
- [ ] Advanced metrics and observability
- [ ] Development tools
- [ ] CLI tools
- [ ] Monitoring dashboard
- [ ] Community and contributions

**Estimation**: 2027

---

### 🔴 Phase 5: Scalability and Production

**Objective**: Optimize for large-scale production use

**Status**: Not started

#### Planned Tasks

- [ ] Performance optimization
- [ ] Horizontal scalability
- [ ] High availability
- [ ] Geographic distribution
- [ ] Compliance and certifications
- [ ] Security audits

**Estimation**: 2027-2028

---

## 📈 Progress Metrics

### Phase 1: Core Layers
- **Progress**: 100% (3/3 core layers completed + Shared)
- **Test Coverage**: 87%
- **Documentation**: 100% (layers + whitepaper)

### Phase 2: SDK Implementation
- **Progress**: 0%
- **Status**: Waiting to complete core optimizations

---

## 🎯 Short-Term Objectives

1. **Robust Core** (Q1-Q2 2026)
   - Reduce core size to make it lighter
   - New pure functions to strengthen SDKs
   - Improve code quality
   - Performance optimizations

2. **Protocol Beta SDK** (Q2-Q3 2026)
   - Provide a functional and auditable implementation of the AI-PIP protocol
   - Capable of detecting, locating, and mitigating prompt injection and attack surfaces in real environments (especially browser)
   - Integrating DOM scanning, precise lineage per node, configurable policies, and agent flows
   - To enable prevention, visualization, and exact risk auditing before interaction with AI models

3. **Testing in Real Environments** (Q3-Q4 2026)
   - Validate protocol effectiveness in production
   - Collect metrics and feedback

---

## 📝 Notes

- Estimates are approximate and may vary according to priorities
- Phases may overlap if resources are available
- Functionality may be adjusted according to community feedback

---

## 🤝 Contributions

Want to contribute? Review pending tasks in each phase and contact the team to coordinate.

**Repository**: https://github.com/AI-PIP/ai-pip-core  
**Issues**: https://github.com/AI-PIP/ai-pip-core/issues

