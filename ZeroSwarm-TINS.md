# ZeroSwarm: Stigmergic AI Specialist Coordination System

## Description

ZeroSwarm is a deterministic AI specialist coordination system that replaces traditional handwritten system prompts with 128-bit seed references. A four-integer tuple `(roleId, styleId, capabilitySalt, worldSeed)` is hashed through FNV-1a and fed into octave-layered coherent noise to produce a unique AI specialist persona with five continuous behavioral dimensions, three categorical selectors, and a natural-language system prompt. The entire agent identity is reconstructable from 16 bytes. No prompt text is ever stored. The system operates as a pure-function pipeline: same seed in, identical prompt out, on any machine, in any session, forever.

The architecture is organised into nine auditable layers: identity hashing, capability field, inter-agent affinity, epoch tracking, evolution chains, stigmergic signal fields, prompt generation, selection protocol, and determinism audit. Each layer is a pure function with zero side effects. A three-layer orchestrator (ZeroSpecialist selection, dynamic approach matching, stigmergic signal blending) coordinates specialist assignment at runtime. Candidates are sampled, benchmarked, and rotated through an evolutionary protocol that promotes high-quality performers to locked winners and drops the bottom quartile each epoch.

The system integrates into a React + Vite + TypeScript web application with Zustand state management, Dexie IndexedDB persistence, and shadcn/ui components. It connects to OpenRouter-compatible LLM APIs for actual AI responses, with the ZeroSwarm layer handling specialist selection, approach matching, and signal coordination transparently.

Optional JSON profile support externalizes the vocabulary pools and prompt templates that the prompt generator uses, allowing custom specialist personalities without modifying core code. The hash pipeline, capability field, selection protocol, and all other pure-function layers remain completely unchanged when profiles are used.

**Technology Stack:**
- **Runtime:** TypeScript, React 18, Vite
- **State:** Zustand with localStorage persistence (BigInt serialization)
- **Database:** Dexie (IndexedDB) with 4 schema versions
- **UI:** shadcn/ui (Radix primitives + Tailwind CSS)
- **Routing:** React Router
- **Data Fetching:** TanStack React Query
- **API:** OpenRouter-compatible REST endpoints

## Functionality

### Core Features

**1. Deterministic Agent Identity from 128-Bit Seeds**
- An `AgentSeed` is a 4-tuple: `{roleId: u16, styleId: u16, capabilitySalt: u16, worldSeed: u64}`
- The seed is packed into 24 bytes (with 4 bytes padding) and hashed via FNV-1a to produce a 64-bit identity hash
- The identity hash deterministically selects: one of 10 role archetypes, one of 10 style modifiers, one of 8 reasoning approaches
- A `WinnerRecord` packs `{originSeed: u64, depth: u32, epochPromoted: u32, scoreAtPromotion: f32}` = 128 bits total
- Any winner is fully reconstructable from its 128-bit reference without storing prompt text

**2. Coherent Capability Field**
- Maps the 2D coordinate `(roleId, styleId)` through 4-octave coherent value noise to produce 5 continuous behavioral dimensions: verbosity, assertiveness, formality, domainFocus, reasoningDepth
- Each dimension uses a different salt (0, 1000, 2000, 3000, 4000) for independence
- Noise is interpolated via smoothstep over a hash-based grid, producing locally coherent values: nearby seeds generate related personas
- All values normalised to [0, 1]

**3. Natural-Language Prompt Assembly**
- The 5 continuous dimensions are thresholded into human-readable instructions (3 tiers each: low/mid/high)
- Combined with the categorical archetype, style, and reasoning selections into a ~200-token system prompt
- Prompt includes swarm coordination instructions and an 8-hex identity tag
- Same seed always produces byte-identical prompt text
- When a profile is provided, all lookups use profile pools and template selection uses profile templates; hash arithmetic is identical

**4. Evolutionary Selection Protocol (Sample-Benchmark-Rotate)**
- `sample(n, worldSeed, epochIndex, biasDigest?)` generates up to 64 candidate AgentSeeds per epoch
- `benchmark(outputs, candidates)` groups quality scores by agent hash and averages
- `rotate(results, pool, winners, epoch, replacements)` drops bottom 25%, promotes candidates scoring >= 0.75 to winners, fills gaps with fresh samples
- BiasDigest (single 64-bit hash) encodes the learning system's cumulative output for coordinate-based sampling nudges

**5. O(1) Auxiliary Queries**
- **Affinity:** Symmetric pairwise agent affinity in [0, 1] via sorted pair hashing with polarity channel
- **Epoch Tracking:** Performance envelope at any past/future epoch without replaying history
- **Signal Field:** O(1) accumulated signal field using geometric series approximation for lookback-windowed signal accumulation
- **Evolution Chain:** Agent deltas at any depth via child hash derivation; full reconstruction from 128-bit winner reference

**6. Three-Layer Orchestration**
- Layer 1 (ZeroSpecialist): Scores all candidates against task context using keyword matching against capability profiles; locked winners receive +0.15 bonus matched by full 64-bit agent hash
- Layer 2 (Dynamic Approaches): Pattern signature matching selects approach templates
- Layer 3 (Stigmergic Board): Epoch-based two-tier exponential decay signal blending (30-min short-term, 2-hour long-term); amplification factor 1.5, attenuation factor 0.7; approach selection blends 70% pattern score + 30% signal strength

**7. Nine-Layer Determinism Audit**
- Static analysis scans for anti-patterns: `Math.random()`, `Date.now()`, `crypto.randomUUID()`, `localStorage`, `sessionStorage`, `process.env`
- Runtime suite runs 10 trials per seed confirming identical output
- Cross-seed uniqueness check verifies >= 80% unique prompts from test seed set
- Scoped audit function for pure-library-only scanning

**8. Optional JSON Profile Support**
- Profiles externalize vocabulary pools (role archetypes, style modifiers, reasoning approaches, dimension instruction tiers, swarm context phrases) and prompt templates
- System defaults to hardcoded behavior; opt-in toggle in Settings enables profile mode
- Built-in "Default Extended" profile matches original vocabulary plus additional entries
- Custom `.json` profiles can be uploaded, validated, stored in Dexie, and selected
- Determinism guarantee extends to: same seed + same profile = same prompt

### User Interface Behaviour
- Specialists displayed as cards showing ID (`zs-{hash8}`), quality score, specialization vector, execution count
- Human-readable names use `domain_animal` format with honorific progression based on performance tier
- Dashboard shows candidate pool, locked winners, epoch index, signal network
- Settings panel has 5 tabs: API, System, Templates, Profiles, Database
- Chat interface supports single and parallel specialist execution with swarm trace bubbles

### State Management
- `worldSeed` generated once on first launch, persisted permanently in Dexie, never changes
- `epochIndex` incremented on each rotation cycle
- `candidatePool` (max 64 AgentSeeds) and `lockedWinners` (WinnerRecord[]) persisted via Zustand with localStorage adapter
- BigInt serialization handled via toString/BigInt() round-trip in custom storage adapter
- `biasDigest` written by learning layer, read by selection protocol — the only shared state between learning and identity systems
- `profileEnabled` and `activeProfileId` control the optional profile override path

## Technical Implementation

### Architecture Overview

```
src/zero-specialist/          <- Pure functions only. Zero UI/state imports.
  types.ts                    <- All type definitions
  hash.ts                     <- FNV-1a 64-bit hashing primitives
  capability-field.ts         <- Coherent noise capability mapping
  affinity.ts                 <- Symmetric pairwise affinity
  epoch-tracking.ts           <- O(1) temporal performance envelopes
  evolution-chain.ts          <- Winner lineage and reconstruction
  signal-field.ts             <- O(1) stigmergic field queries
  prompt-generator.ts         <- Deterministic system prompt assembly
  selection-protocol.ts       <- Sample/Benchmark/Rotate protocol
  boundary.ts                 <- Decoupling interface for learning system
  profile.ts                  <- Profile loader, validator, default profile
  audit.ts                    <- Static + runtime determinism verification
  index.ts                    <- Public API barrel export
  __tests__/determinism.test.ts <- 9-layer + profile test suite

src/core/                     <- Integration layer (imports zero-specialist + stores)
  types.ts                    <- Shared types (Signal, TaskContext, etc.)
  utils.ts                    <- exponentialDecay and utility functions
  zero-specialist-layer.ts    <- Bridges pure functions to Zustand store
  hybrid-orchestrator.ts      <- Three-layer coordination engine
  stigmergic-board.ts         <- Epoch-based signal deposit/read with two-tier decay
  dynamic-approaches.ts       <- Pattern matching for approach selection
  content-analyzer.ts         <- Content quality analysis
  pattern-analyzer.ts         <- Pattern discovery
  prompt-analyzer.ts          <- Prompt analysis
  prompt-templates.ts         <- Template type definitions
  template-loader.ts          <- Template loading
  template-manager.ts         <- Template management
  template-parser.ts          <- Template parsing
  bootstrap-approaches.ts     <- Default approach initialization
  approach-evolution.ts       <- Approach evolution
  adaptive-resonance.ts       <- Resonance calculations
  index.ts                    <- Core barrel export

src/stores/
  specialist-store.ts         <- Zustand store with BigInt serialization
  system-store.ts             <- System configuration store
  chat-store.ts               <- Chat state management
  ui-store.ts                 <- UI state
  index.ts                    <- Store barrel export

src/storage/
  db.ts                       <- Dexie database (4 schema versions)
  signals-store.ts            <- Signal CRUD operations
  specialists-store.ts        <- Specialist CRUD operations
  profiles-store.ts           <- Profile CRUD operations
  history-store.ts            <- Execution history storage
  approaches-store.ts         <- Approach storage
  patterns-store.ts           <- Pattern storage
  templates-store.ts          <- Template storage
  index.ts                    <- Storage barrel export

src/components/
  Chat/                       <- Chat interface components
    ChatInterface.tsx          <- Main chat component
    MessageInput.tsx           <- Input field
    MessageList.tsx            <- Message display
    MessageItem.tsx            <- Individual message
    StreamingMessage.tsx       <- Streaming response display
    ConversationHeader.tsx     <- Header with specialist info
    SwarmTraceBubble.tsx       <- Swarm coordination trace
    index.ts
  Dashboard/                  <- System dashboard
    SystemDashboard.tsx        <- Main dashboard
    SpecialistCard.tsx         <- Specialist display card
    specialist-carousel.tsx    <- Carousel of specialists
    ApproachCard.tsx           <- Approach display
    SignalBoard.tsx            <- Signal visualization
    signal-network-graph.tsx   <- Network graph
    performance-metrics.tsx    <- Performance charts
    index.ts
  Settings/                   <- Configuration panel
    SettingsPanel.tsx          <- 5-tab settings container
    ApiKeyInput.tsx            <- API key configuration
    ApiUrlInput.tsx            <- API URL configuration
    SystemConfig.tsx           <- System parameters
    TemplateManagement.tsx     <- Template editor
    ProfileSettings.tsx        <- Profile toggle/upload/preview
    Database.tsx               <- Database management
    index.ts
  Layout/                     <- App shell
    AppLayout.tsx              <- Main layout wrapper
    AppSidebar.tsx             <- Navigation sidebar
    Header.tsx                 <- Top header
    DonateButton.tsx           <- Support button
    index.ts
  ui/                         <- shadcn/ui primitives (40+ components)

src/hooks/
  useCoordination.ts          <- Orchestrator hook
  useParallelChat.ts          <- Parallel specialist chat
  useStreamingChat.ts         <- Streaming response handling
  useApiKey.ts                <- API key management
  useSystemStats.ts           <- System statistics
  usePatternDiscovery.ts      <- Pattern analysis hook
  useDashboardContext.ts      <- Dashboard state
  useDashboardLayout.ts       <- Dashboard layout
  use-local-storage.ts        <- localStorage hook
  use-mobile.tsx              <- Mobile detection
  use-toast.ts                <- Toast notifications
  index.ts

src/api/
  openrouter-client.ts        <- OpenRouter API client
  error-handler.ts            <- API error handling
  index.ts

src/utils/
  constants.ts                <- App-wide constants
  specialist-names.ts         <- Human-readable name generation
  clustering.ts               <- Pattern clustering
  markdown-formatter.ts       <- Markdown utilities

src/pages/
  Index.tsx                   <- Home/landing page
  Chat.tsx                    <- Chat page
  Dashboard.tsx               <- Dashboard page
  Settings.tsx                <- Settings page
  Tutorial.tsx                <- Tutorial/docs page
  NotFound.tsx                <- 404 page

src/
  App.tsx                     <- Root component with routing
  main.tsx                    <- Entry point
  vite-env.d.ts               <- Vite type declarations
```

**Critical Constraint:** The `src/zero-specialist/` directory must never import from `src/components/`, `src/stores/`, or any UI framework. It is a pure-function library. The `boundary.ts` file is the ONLY type that learning-side code may reference.

### Data Structures

#### File: `src/zero-specialist/types.ts`

```typescript
// src/zero-specialist/types.ts
// Core type definitions for the ZeroSpecialist deterministic agent system.
// No runtime logic — pure type declarations.

export interface AgentSeed {
  roleId: number;           // 0-65535: position in role space
  styleId: number;          // 0-65535: position in style space
  capabilitySalt: number;   // 0-65535: fine-grained variation within role+style
  worldSeed: bigint;        // 64-bit world constant, never changes per installation
}

export interface WinnerRecord {
  originSeed: bigint;       // The xxHash64 of the winning AgentSeed
  depth: number;            // Evolution depth (0 = unevolved winner)
  epochPromoted: number;    // Epoch index at time of promotion
  scoreAtPromotion: number; // Quality score [0,1] at promotion time
}

export interface AgentOutputRecord {
  agentSeedHash: bigint;    // xxHash64 of AgentSeed — the agent's identity
  taskId: string;
  epochIndex: number;
  responseText: string;
  qualityScore: number;     // From content-analyzer.ts, range [0,1]
  taskType: string;
  durationMs: number;
}

export interface CandidatePool {
  candidates: AgentSeed[];  // Hard cap: N <= 64
  epochSampled: number;
}

/**
 * BiasDigest: a single 64-bit value encoding the learning system's
 * cumulative output. Computed by hashing benchmark results from prior epochs
 * into a rolling digest. Used by sample() as a coordinate for bias field lookup.
 */
export type BiasDigest = bigint;

/** @deprecated Use BiasDigest instead. Kept for migration compatibility. */
export interface PatternBias {
  roleRegions: Array<{ roleId: number; density: number }>;
  styleRegions: Array<{ styleId: number; density: number }>;
}

export interface CapabilityProfile {
  verbosity: number;        // [0,1] concise -> expansive
  assertiveness: number;    // [0,1] tentative -> confident
  formality: number;        // [0,1] casual -> formal
  domainFocus: number;      // [0,1] generalist -> specialist
  reasoningDepth: number;   // [0,1] fast heuristic -> deep deliberate
}

export interface PerformanceEnvelope {
  expectedMin: number;      // Lower bound of expected quality [0,1]
  expectedMax: number;      // Upper bound of expected quality [0,1]
  consistency: number;      // How stable is this agent's performance [0,1]
}

export interface BenchmarkResult {
  seed: AgentSeed;
  seedHash: bigint;
  averageQuality: number;
  taskCount: number;
}

export interface RotationResult {
  updatedPool: CandidatePool;
  newWinners: WinnerRecord[];
  dropped: bigint[];        // Seed hashes of dropped candidates
}

/**
 * The PatternStore is the ONLY shared state between the learning system
 * and the ZeroSpecialist selection system. Learning writes a BiasDigest here.
 * ZeroSpecialist reads the BiasDigest from here during sample().
 */
export interface PatternStore {
  biasDigest: BiasDigest;
  lastUpdatedEpoch: number;
}

/**
 * ZeroSpecialist Profile v1.
 * Externalizes the vocabulary pools and prompt templates.
 * The seed controls selection; the profile defines what is available to select from.
 */
export interface ZeroSpecialistProfile {
  name: string;
  description: string;
  version: string;
  type: 'zerospecialist';

  pools: {
    role_archetypes: string[];
    style_modifiers: string[];
    reasoning_approaches: string[];
    verbosity_high: string[];
    verbosity_mid: string[];
    verbosity_low: string[];
    assertiveness_high: string[];
    assertiveness_mid: string[];
    assertiveness_low: string[];
    formality_high: string[];
    formality_mid: string[];
    formality_low: string[];
    domain_focus_high: string[];
    domain_focus_low: string[];
    reasoning_depth_high: string[];
    reasoning_depth_low: string[];
    swarm_context: string[];
    [key: string]: string[];  // Allow additional custom pools
  };

  templates: string[];

  rules: {
    min_pool_size: {
      role_archetypes: number;
      style_modifiers: number;
      reasoning_approaches: number;
    };
    required_pools: string[];
    identity_tag_source: string;
  };
}

/**
 * Stored profile record in Dexie.
 */
export interface ProfileRecord {
  id: string;
  name: string;
  json: string;
  uploadedAt: number;
}
```

### Layer 1: FNV-1a 64-Bit Hashing

All identity derivation flows through this single hash function. It is the cryptographic root of the entire system.

#### File: `src/zero-specialist/hash.ts`

```typescript
// src/zero-specialist/hash.ts
// Pure 64-bit hashing functions for deterministic agent identity.
// Contract: same inputs -> same output across all machines, all sessions.
// No Math.random(), Date.now(), crypto.randomUUID(), or external state.

import type { AgentSeed } from './types';

const FNV_PRIME = 1099511628211n;
const FNV_OFFSET = 14695981039346656037n;
const MASK64 = 0xFFFFFFFFFFFFFFFFn;

/**
 * Pure 64-bit hash using FNV-1a variant for TypeScript BigInt.
 * Same inputs -> same output across all environments.
 */
export function fnv64(bytes: Uint8Array): bigint {
  let hash = FNV_OFFSET;
  for (const byte of bytes) {
    hash ^= BigInt(byte);
    hash = (hash * FNV_PRIME) & MASK64;
  }
  return hash;
}

/**
 * Packs an AgentSeed into a 24-byte deterministic representation.
 * Layout: [roleId:4][styleId:4][capabilitySalt:4][pad:4][worldSeed:8]
 */
function packAgentSeed(seed: AgentSeed): Uint8Array {
  const buf = new ArrayBuffer(24);
  const view = new DataView(buf);
  view.setUint32(0, seed.roleId, true);
  view.setUint32(4, seed.styleId, true);
  view.setUint32(8, seed.capabilitySalt, true);
  view.setBigUint64(16, seed.worldSeed, true);
  return new Uint8Array(buf);
}

/**
 * Computes the deterministic 64-bit identity hash of an AgentSeed.
 * Same seed -> same hash, always.
 */
export function agentHash(seed: AgentSeed): bigint {
  return fnv64(packAgentSeed(seed));
}

/**
 * Converts a 64-bit hash to a float in [0, 1).
 * Uses the lower 32 bits for uniform distribution.
 */
export function hashToFloat(h: bigint): number {
  return Number(h & 0xFFFFFFFFn) / 0x100000000;
}

/**
 * Derives independent float channels from the same hash without re-hashing.
 * Each channel shifts and XORs to produce a decorrelated value in [0, 1).
 */
export function hashToFloatChannel(h: bigint, channel: number): number {
  const shifted = (h >> BigInt(channel * 8)) ^ h;
  return Number(shifted & 0xFFFFFFFFn) / 0x100000000;
}

/**
 * Derives a child hash from a parent hash and an integer index.
 * Used for generating multiple independent values from a single seed.
 * Same (parentHash, childIndex) -> same output, always.
 */
export function deriveChildHash(parentHash: bigint, childIndex: number): bigint {
  const buf = new ArrayBuffer(16);
  const view = new DataView(buf);
  view.setBigUint64(0, parentHash, true);
  view.setUint32(8, childIndex, true);
  return fnv64(new Uint8Array(buf));
}
```

**Constants are non-negotiable.** FNV_PRIME and FNV_OFFSET are standardised values for the 64-bit FNV-1a variant. Changing them breaks all determinism guarantees.

### Layer 2: Coherent Capability Field

Maps the 2D seed coordinate to 5 continuous behavioral dimensions using octave-layered value noise.

#### File: `src/zero-specialist/capability-field.ts`

```typescript
// src/zero-specialist/capability-field.ts
// Maps (roleX, styleY) coordinates to behavioral dimensions using octave-layered coherent noise.
// Pure functions only. Same inputs -> same output across all environments.

import type { CapabilityProfile } from './types';
import { hashToFloat, fnv64 } from './hash';

/** Smooth interpolation (smoothstep) */
function smoothstep(t: number): number {
  return t * t * (3 - 2 * t);
}

/**
 * Deterministic hash-based grid value at integer coordinates.
 * Same (gx, gy, salt, worldSeed) -> same float in [0, 1).
 */
function gridHash(gx: number, gy: number, salt: number, worldSeed: bigint): number {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setInt32(0, gx, true);
  view.setInt32(4, gy, true);
  view.setUint32(8, salt, true);
  view.setBigUint64(12, worldSeed, true);
  return hashToFloat(fnv64(new Uint8Array(buf)));
}

/**
 * Coherent value noise over a 2D integer grid.
 * Returns a float in approximately [-1, 1].
 * Uses octave layering for multi-scale coherence.
 */
function coherentNoise(
  x: number,
  y: number,
  salt: number,
  worldSeed: bigint,
  octaves = 4
): number {
  let value = 0;
  let amplitude = 1.0;
  let frequency = 1.0;
  let maxAmplitude = 0;

  for (let i = 0; i < octaves; i++) {
    const gx = Math.floor(x * frequency);
    const gy = Math.floor(y * frequency);
    const fx = (x * frequency) - gx;
    const fy = (y * frequency) - gy;
    const sx = smoothstep(fx < 0 ? fx + 1 : fx);
    const sy = smoothstep(fy < 0 ? fy + 1 : fy);

    const n00 = gridHash(gx,     gy,     salt + i, worldSeed) * 2 - 1;
    const n10 = gridHash(gx + 1, gy,     salt + i, worldSeed) * 2 - 1;
    const n01 = gridHash(gx,     gy + 1, salt + i, worldSeed) * 2 - 1;
    const n11 = gridHash(gx + 1, gy + 1, salt + i, worldSeed) * 2 - 1;

    const nx0 = n00 * (1 - sx) + n10 * sx;
    const nx1 = n01 * (1 - sx) + n11 * sx;
    value += amplitude * (nx0 * (1 - sy) + nx1 * sy);
    maxAmplitude += amplitude;
    amplitude *= 0.5;
    frequency *= 2.0;
  }

  return value / maxAmplitude;
}

/**
 * Maps a (roleX, styleY) coordinate to a CapabilityProfile.
 * The field is continuous and coherent — neighbors produce related profiles.
 * Scale: roleX and styleY are in range [0, 65535].
 * Same inputs -> identical profile across all machines.
 */
export function capabilityField(
  roleX: number,
  styleY: number,
  worldSeed: bigint
): CapabilityProfile {
  // Normalize to a sensible noise frequency range
  const nx = roleX / 65535 * 8;
  const ny = styleY / 65535 * 8;

  // Each dimension uses a different salt for independence
  const raw = (salt: number) => (coherentNoise(nx, ny, salt * 1000, worldSeed) + 1) / 2;

  return {
    verbosity:      raw(0),
    assertiveness:  raw(1),
    formality:      raw(2),
    domainFocus:    raw(3),
    reasoningDepth: raw(4),
  };
}
```

### Layer 3: Symmetric Pairwise Affinity

#### File: `src/zero-specialist/affinity.ts`

```typescript
// src/zero-specialist/affinity.ts
// Deterministic inter-agent affinity computation.
// Symmetric: agentAffinity(A, B) === agentAffinity(B, A).
// O(1) per pair. No stored affinity matrix.

import { fnv64, hashToFloat } from './hash';

/**
 * Symmetric pair hash. Sorts seeds before packing to enforce symmetry.
 * Same (seedA, seedB, salt) -> same hash regardless of argument order.
 */
function pairHash(seedA: bigint, seedB: bigint, salt: number): bigint {
  const [lo, hi] = seedA < seedB ? [seedA, seedB] : [seedB, seedA];
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setBigUint64(0, lo, true);
  view.setBigUint64(8, hi, true);
  view.setUint32(16, salt, true);
  return fnv64(new Uint8Array(buf));
}

/**
 * Returns collaboration affinity in [0, 1].
 * 0.0 = antagonistic, 0.5 = neutral, 1.0 = highly collaborative.
 * Uses a polarity channel to determine cooperative vs competitive tendency.
 */
export function agentAffinity(
  seedHashA: bigint,
  seedHashB: bigint
): number {
  const base = hashToFloat(pairHash(seedHashA, seedHashB, 0));
  const polarity = hashToFloat(pairHash(seedHashA, seedHashB, 1));
  // Polarity > 0.7 = strong cooperative, < 0.3 = competitive, middle = neutral
  return polarity > 0.7 ? 0.5 + base * 0.5 :
         polarity < 0.3 ? base * 0.5 :
         base;
}

/**
 * Given a focal agent seed and a pool of candidate seeds,
 * returns the pool sorted by descending affinity to the focal agent.
 * O(N) where N = pool size, bounded by CandidatePool cap (64).
 */
export function rankByAffinity(
  focalSeedHash: bigint,
  pool: bigint[]
): bigint[] {
  return [...pool].sort(
    (a, b) => agentAffinity(focalSeedHash, b) - agentAffinity(focalSeedHash, a)
  );
}
```

### Layer 4: Temporal Epoch Tracking

#### File: `src/zero-specialist/epoch-tracking.ts`

```typescript
// src/zero-specialist/epoch-tracking.ts
// Deterministic performance epoch tracking.
// O(1) access to any epoch — past or future — without replaying history.
// Epoch is a world-defined integer tick, never wall-clock time.

import type { PerformanceEnvelope } from './types';
import { fnv64, hashToFloat } from './hash';

/**
 * Computes a temporal hash for an agent at a specific epoch.
 * Same (agentSeedHash, epochIndex, worldSeed) -> same hash, always.
 */
function temporalHash(agentSeedHash: bigint, epochIndex: number, worldSeed: bigint): bigint {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setBigUint64(0, agentSeedHash, true);
  view.setUint32(8, epochIndex, true);
  view.setBigUint64(12, worldSeed, true);
  return fnv64(new Uint8Array(buf));
}

/**
 * Returns the deterministic expected performance envelope for an agent at a given epoch.
 * This is a prior — actual observed performance updates PatternStore separately.
 * O(1): jump to any epoch directly without replaying history.
 */
export function performanceEnvelope(
  agentSeedHash: bigint,
  epochIndex: number,
  worldSeed: bigint
): PerformanceEnvelope {
  const h = temporalHash(agentSeedHash, epochIndex, worldSeed);
  const base = hashToFloat(h);
  const variance = hashToFloat(h >> 16n) * 0.3;
  const consistency = hashToFloat(h >> 32n);

  return {
    expectedMin: Math.max(0, base - variance),
    expectedMax: Math.min(1, base + variance),
    consistency,
  };
}

/**
 * Query any past epoch's expected envelope — identical O(1) cost.
 * Use for retrospective analysis of why a winner was promoted.
 */
export function queryHistoricalPerformance(
  agentSeedHash: bigint,
  pastEpoch: number,
  worldSeed: bigint
): PerformanceEnvelope {
  return performanceEnvelope(agentSeedHash, pastEpoch, worldSeed);
}
```

### Layer 5: Evolution Chain and Winner Reconstruction

#### File: `src/zero-specialist/evolution-chain.ts`

```typescript
// src/zero-specialist/evolution-chain.ts
// Winner lineage and evolution chain using Zero-Causal chain hashing.
// A winning agent is fully reconstructable from a 128-bit reference (originSeed, depth).
// O(1) depth access — no replay required.

import type { AgentSeed, WinnerRecord } from './types';
import { fnv64, hashToFloat, deriveChildHash } from './hash';

/**
 * Derives the agent seed deltas at a given evolution depth from an origin.
 * depth 0 = the original unmodified seed parameters.
 * depth N = N mutations applied deterministically.
 * O(1): depth IS the coordinate, not the result of iteration.
 */
export function agentAtDepth(originSeed: bigint, depth: number): {
  roleIdDelta: number;
  styleIdDelta: number;
  capabilitySaltDelta: number;
} {
  const depthHash = deriveChildHash(originSeed, depth);
  return {
    roleIdDelta:          Math.floor(hashToFloat(depthHash)        * 256) - 128,
    styleIdDelta:         Math.floor(hashToFloat(depthHash >> 16n) * 256) - 128,
    capabilitySaltDelta:  Math.floor(hashToFloat(depthHash >> 32n) * 512) - 256,
  };
}

/**
 * Reconstructs the full AgentSeed from a WinnerRecord.
 * Given only the 128-bit reference, the complete agent is recoverable.
 */
export function reconstructWinner(
  record: WinnerRecord,
  baseWorldSeed: bigint
): AgentSeed {
  const roleId   = Number((record.originSeed >> 48n) & 0xFFFFn);
  const styleId  = Number((record.originSeed >> 32n) & 0xFFFFn);
  const capSalt  = Number((record.originSeed >> 16n) & 0xFFFFn);

  const delta = agentAtDepth(record.originSeed, record.depth);

  return {
    roleId:          Math.max(0, Math.min(65535, roleId  + delta.roleIdDelta)),
    styleId:         Math.max(0, Math.min(65535, styleId + delta.styleIdDelta)),
    capabilitySalt:  Math.max(0, Math.min(65535, capSalt + delta.capabilitySaltDelta)),
    worldSeed:       baseWorldSeed,
  };
}

/**
 * Derives an alternate branch from a winner at a given fork depth.
 * Branch 0 = main lineage, branch > 0 = alternate outcome.
 */
export function forkBranch(
  originSeed: bigint,
  forkDepth: number,
  branchIndex: number
): bigint {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setBigUint64(0, originSeed, true);
  view.setUint32(8, forkDepth, true);
  view.setUint32(12, branchIndex, true);
  return fnv64(new Uint8Array(buf));
}

/**
 * Creates a WinnerRecord from a promoted AgentSeed.
 * Packs the (roleId, styleId, capabilitySalt) into the high bits of originSeed.
 */
export function createWinnerRecord(
  seed: AgentSeed,
  epochPromoted: number,
  scoreAtPromotion: number,
  depth = 0
): WinnerRecord {
  const originSeed =
    (BigInt(seed.roleId)          << 48n) |
    (BigInt(seed.styleId)         << 32n) |
    (BigInt(seed.capabilitySalt)  << 16n) |
    (seed.worldSeed & 0xFFFFn);

  return { originSeed, depth, epochPromoted, scoreAtPromotion };
}
```

**originSeed bit layout:**
```
Bits 63-48: roleId          (16 bits)
Bits 47-32: styleId         (16 bits)
Bits 31-16: capabilitySalt  (16 bits)
Bits 15-0:  worldSeed low   (16 bits, XOR portion)
```

### Layer 6: Stigmergic Signal Field

#### File: `src/zero-specialist/signal-field.ts`

```typescript
// src/zero-specialist/signal-field.ts
// Stigmergic signal field — O(1) field query replacing O(N) signal card iteration.

import { fnv64, hashToFloat } from './hash';

/**
 * Computes the stigmergic signal strength at a point in (taskType, agentRole, epoch) space.
 * O(1) field query.
 */
export function signalStrength(
  taskTypeIndex: number,
  agentRoleId: number,
  epochIndex: number,
  worldSeed: bigint
): number {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setUint32(0, taskTypeIndex, true);
  view.setUint32(4, agentRoleId, true);
  view.setUint32(8, epochIndex, true);
  view.setBigUint64(12, worldSeed, true);

  const h = fnv64(new Uint8Array(buf));
  const base = hashToFloat(h);
  const epochDecay = 1.0;

  return base * epochDecay;
}

/**
 * Converts a task type string to a stable integer index.
 */
export function taskTypeToIndex(taskType: string): number {
  const encoder = new TextEncoder();
  const bytes = encoder.encode(taskType);
  const h = fnv64(bytes);
  return Number(h & 0xFFn); // 256 task type buckets
}

/**
 * O(1) accumulated signal field for a (roleId, styleId) region.
 * Uses closed-form geometric series approximation for lookback-windowed accumulation.
 * Pure function. Same inputs -> same output. No stored state.
 */
export function accumulatedSignalField(
  roleId: number,
  styleId: number,
  currentEpoch: number,
  worldSeed: bigint,
  config?: {
    lookbackEpochs?: number;
    decayPerEpoch?: number;
    taskTypeSalt?: number;
  }
): number {
  const lookback = config?.lookbackEpochs ?? 12;
  const decay = config?.decayPerEpoch ?? 0.85;
  const taskSalt = config?.taskTypeSalt ?? 0;

  const base = signalStrength(taskSalt, roleId, currentEpoch, worldSeed);

  if (decay >= 1.0) return base * lookback;
  if (decay <= 0.0) return base;

  const geometricSum = (1 - Math.pow(decay, lookback)) / (1 - decay);
  const accumulated = base * geometricSum;

  const maxPossible = geometricSum;
  return Math.min(1, accumulated / maxPossible);
}
```

### Layer 7: Deterministic Prompt Generation

#### File: `src/zero-specialist/prompt-generator.ts`

```typescript
// src/zero-specialist/prompt-generator.ts
// Deterministic system prompt generation — PURE FUNCTION of AgentSeed (+ optional Profile).
// No side effects. No external reads. No randomness.
// Same AgentSeed -> identical output on all machines, all sessions.
// Same AgentSeed + same Profile -> identical output on all machines, all sessions.

import type { AgentSeed, ZeroSpecialistProfile } from './types';
import { agentHash, deriveChildHash } from './hash';
import { capabilityField } from './capability-field';

// --- Original hardcoded constants (used when no profile is provided) ---

const ROLE_ARCHETYPES: string[] = [
  'analyst', 'synthesizer', 'critic', 'explorer', 'implementer',
  'strategist', 'communicator', 'researcher', 'validator', 'integrator',
];

const STYLE_MODIFIERS: string[] = [
  'concise', 'thorough', 'direct', 'exploratory', 'structured',
  'narrative', 'systematic', 'adaptive', 'precise', 'generative',
];

const REASONING_APPROACHES: string[] = [
  'first-principles', 'analogical', 'deductive', 'inductive',
  'abductive', 'comparative', 'causal', 'probabilistic',
];

// --- Tier selection salt map for child hash derivation ---

const TIER_SALT_MAP: Record<string, number> = {
  verbosity: 10,
  assertiveness: 11,
  formality: 12,
  domain_focus: 13,
  reasoning_depth: 14,
};

const TEMPLATE_SALT = 20;
const SWARM_CONTEXT_SALT = 21;

/**
 * Selects a string from a tiered pool using the capability value and hash.
 * value > 0.7 -> high tier, value < 0.3 -> low tier, else -> mid tier.
 * For domain_focus and reasoning_depth which have no mid tier, mid falls through to low.
 */
function selectFromTier(
  hash: bigint,
  value: number,
  pools: ZeroSpecialistProfile['pools'],
  dimensionKey: string
): string {
  const tier = value > 0.7 ? 'high' : value < 0.3 ? 'low' : 'mid';
  let poolKey = `${dimensionKey}_${tier}`;
  let pool = pools[poolKey];
  // Fallback: domain_focus and reasoning_depth have no mid tier — use low
  if (!pool || pool.length === 0) {
    poolKey = `${dimensionKey}_low`;
    pool = pools[poolKey];
  }
  if (!pool || pool.length === 0) return '';
  const salt = TIER_SALT_MAP[dimensionKey] ?? 15;
  const idx = Number(deriveChildHash(hash, salt) % BigInt(pool.length));
  return pool[idx];
}

/**
 * Generates a fully deterministic system prompt from an AgentSeed.
 * PURE FUNCTION. No side effects. No external reads.
 *
 * When profile is omitted or null, uses the original hardcoded lookup tables
 * and template string — byte-identical to the pre-profile implementation.
 *
 * When profile is provided, all lookups use profile.pools and template
 * selection uses profile.templates. Hash arithmetic is identical.
 */
export function generateSystemPrompt(seed: AgentSeed, profile?: ZeroSpecialistProfile | null): string {
  const hash = agentHash(seed);
  const capProfile = capabilityField(seed.roleId, seed.styleId, seed.worldSeed);

  // --- Profile-driven path ---
  if (profile) {
    const pools = profile.pools;

    const archetype = pools.role_archetypes[Number(hash % BigInt(pools.role_archetypes.length))];
    const style = pools.style_modifiers[Number(deriveChildHash(hash, 1) % BigInt(pools.style_modifiers.length))];
    const reasoning = pools.reasoning_approaches[Number(deriveChildHash(hash, 2) % BigInt(pools.reasoning_approaches.length))];

    const verbosity = selectFromTier(hash, capProfile.verbosity, pools, 'verbosity');
    const assertiveness = selectFromTier(hash, capProfile.assertiveness, pools, 'assertiveness');
    const formality = selectFromTier(hash, capProfile.formality, pools, 'formality');
    const domainFocus = selectFromTier(hash, capProfile.domainFocus, pools, 'domain_focus');
    const reasoningDepth = selectFromTier(hash, capProfile.reasoningDepth, pools, 'reasoning_depth');

    const swarmIdx = Number(deriveChildHash(hash, SWARM_CONTEXT_SALT) % BigInt(pools.swarm_context.length));
    const swarmContext = pools.swarm_context[swarmIdx];

    const templateIdx = Number(deriveChildHash(hash, TEMPLATE_SALT) % BigInt(profile.templates.length));
    const template = profile.templates[templateIdx];

    const identityTag = hash.toString(16).slice(0, 8);

    return template
      .replace('{role_archetypes}', archetype)
      .replace('{style_modifiers}', style)
      .replace('{reasoning_approaches}', reasoning)
      .replace('{verbosity}', verbosity)
      .replace('{assertiveness}', assertiveness)
      .replace('{formality}', formality)
      .replace('{domain_focus}', domainFocus)
      .replace('{reasoning_depth}', reasoningDepth)
      .replace('{swarm_context}', swarmContext)
      .replace('{identity_tag}', identityTag)
      .trim();
  }

  // --- Original hardcoded path (unchanged, byte-identical to pre-profile) ---

  const archetypeIndex = Number(hash % BigInt(ROLE_ARCHETYPES.length));
  const styleIndex = Number(deriveChildHash(hash, 1) % BigInt(STYLE_MODIFIERS.length));
  const reasoningIndex = Number(deriveChildHash(hash, 2) % BigInt(REASONING_APPROACHES.length));

  const archetype = ROLE_ARCHETYPES[archetypeIndex];
  const style = STYLE_MODIFIERS[styleIndex];
  const reasoning = REASONING_APPROACHES[reasoningIndex];

  const verbosityInstruction =
    capProfile.verbosity > 0.7 ? 'Provide comprehensive, detailed responses.' :
    capProfile.verbosity < 0.3 ? 'Be concise. Prioritize brevity over completeness.' :
    'Balance depth and conciseness based on task complexity.';

  const assertivenessInstruction =
    capProfile.assertiveness > 0.7 ? 'State conclusions with confidence. Avoid hedging.' :
    capProfile.assertiveness < 0.3 ? 'Acknowledge uncertainty explicitly. Offer alternatives.' :
    'Calibrate confidence to evidence quality.';

  const formalityInstruction =
    capProfile.formality > 0.7 ? 'Use formal, precise language.' :
    capProfile.formality < 0.3 ? 'Use direct, accessible language.' :
    'Match register to the task context.';

  const domainInstruction =
    capProfile.domainFocus > 0.7
      ? 'Focus expertise deeply on the immediate task domain. Resist scope expansion.'
      : 'Draw connections across domains. Generalist perspective is an asset.';

  const reasoningInstruction =
    capProfile.reasoningDepth > 0.7
      ? 'Work through problems deliberately. Show reasoning steps.'
      : 'Prioritize speed. Use heuristics where appropriate.';

  return `You are a ${style} ${archetype} AI specialist operating within a stigmergic coordination swarm.

Your reasoning approach: ${reasoning}.
${verbosityInstruction}
${assertivenessInstruction}
${formalityInstruction}
${domainInstruction}
${reasoningInstruction}

You coordinate with peer specialists through environmental signals. Your role is to contribute your distinct perspective, not to reach consensus. Disagreement with peers is valuable signal.

Identity: Agent-${hash.toString(16).slice(0, 8)}`.trim();
}

/**
 * Audit assertion: verifies the prompt generation path is pure.
 * Call this in test fixtures. Throws if output is non-deterministic.
 */
export function assertPromptDeterminism(seed: AgentSeed, trials = 5, profile?: ZeroSpecialistProfile | null): void {
  const reference = generateSystemPrompt(seed, profile);
  for (let i = 0; i < trials; i++) {
    const result = generateSystemPrompt(seed, profile);
    if (result !== reference) {
      throw new Error(
        `Prompt determinism violation at trial ${i}. ` +
        `Seed: ${JSON.stringify(seed)}` +
        (profile ? ` Profile: ${profile.name}` : '')
      );
    }
  }
}
```

### Layer 8: Selection Protocol (Sample-Benchmark-Rotate)

#### File: `src/zero-specialist/selection-protocol.ts`

```typescript
// src/zero-specialist/selection-protocol.ts
// Swarm selection protocol: Sample -> Benchmark -> Rotate.
// All functions are pure — no side effects, fully testable.
// Deterministic given (worldSeed, epochIndex, biasDigest).

import type { AgentSeed, WinnerRecord, CandidatePool, BenchmarkResult, RotationResult, BiasDigest } from './types';
import { agentHash, hashToFloat, deriveChildHash, fnv64 } from './hash';
import { createWinnerRecord } from './evolution-chain';

const MAX_CANDIDATE_POOL = 64;
const WINNER_THRESHOLD = 0.75;
const ROTATION_BOTTOM_QUARTILE = 0.25;

/**
 * Coordinate-based bias field. Returns a nudge value in [0, 1) for a given
 * (roleId, styleId) position, modulated by the BiasDigest.
 */
function biasField(
  roleId: number,
  styleId: number,
  biasDigest: BiasDigest,
  channel: number
): number {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setUint32(0, roleId, true);
  view.setUint32(4, styleId, true);
  view.setUint32(8, channel, true);
  view.setBigUint64(12, biasDigest, true);
  return hashToFloat(fnv64(new Uint8Array(buf)));
}

/**
 * Samples N candidate AgentSeeds from specialist space.
 * Bias is applied via a BiasDigest (single bigint) instead of a PatternBias struct.
 * When biasDigest is 0n, no bias is applied (equivalent to unbiased sampling).
 * Deterministic: same (n, worldSeed, epoch, biasDigest) -> same candidates.
 */
export function sample(
  n: number,
  worldSeed: bigint,
  epochIndex: number,
  biasDigest: BiasDigest = 0n
): AgentSeed[] {
  const count = Math.min(n, MAX_CANDIDATE_POOL);
  const seeds: AgentSeed[] = [];

  for (let i = 0; i < count; i++) {
    const slotHash = deriveChildHash(worldSeed ^ BigInt(epochIndex * 1000 + i), i);

    let roleId   = Math.floor(hashToFloat(slotHash)        * 65536);
    let styleId  = Math.floor(hashToFloat(slotHash >> 16n) * 65536);
    const capSalt = Math.floor(hashToFloat(slotHash >> 32n) * 65536);

    if (biasDigest !== 0n) {
      const roleBias = biasField(roleId, styleId, biasDigest, 0);
      const styleBias = biasField(roleId, styleId, biasDigest, 1);
      const nudgeThreshold = biasField(roleId, styleId, biasDigest, 2);

      if (nudgeThreshold > 0.6) {
        const roleSeed = Math.floor(roleBias * 65536);
        const styleSeed = Math.floor(styleBias * 65536);
        roleId  = Math.floor(roleId * 0.5 + roleSeed * 0.5);
        styleId = Math.floor(styleId * 0.5 + styleSeed * 0.5);
      }
    }

    seeds.push({
      roleId:  Math.max(0, Math.min(65535, roleId)),
      styleId: Math.max(0, Math.min(65535, styleId)),
      capabilitySalt: capSalt,
      worldSeed,
    });
  }

  return seeds;
}

/**
 * Groups AgentOutputRecords by agent seed hash and computes average quality.
 */
export function benchmark(
  outputs: Array<{ agentSeedHash: bigint; qualityScore: number }>,
  candidates: AgentSeed[]
): BenchmarkResult[] {
  const candidateHashes = new Map(
    candidates.map(s => [agentHash(s).toString(), s])
  );

  const groups = new Map<string, number[]>();
  for (const output of outputs) {
    const key = output.agentSeedHash.toString();
    if (candidateHashes.has(key)) {
      if (!groups.has(key)) groups.set(key, []);
      groups.get(key)!.push(output.qualityScore);
    }
  }

  return Array.from(groups.entries()).map(([key, scores]) => ({
    seed: candidateHashes.get(key)!,
    seedHash: BigInt(key),
    averageQuality: scores.reduce((a, b) => a + b, 0) / scores.length,
    taskCount: scores.length,
  }));
}

/**
 * Drops bottom quartile, promotes winners, returns updated state.
 */
export function rotate(
  results: BenchmarkResult[],
  currentPool: CandidatePool,
  lockedWinners: WinnerRecord[],
  epochIndex: number,
  replacementCandidates: AgentSeed[]
): RotationResult {
  if (results.length === 0) {
    return { updatedPool: currentPool, newWinners: [], dropped: [] };
  }

  const sorted = [...results].sort((a, b) => b.averageQuality - a.averageQuality);
  const cutoff = Math.floor(sorted.length * ROTATION_BOTTOM_QUARTILE);
  const dropping = sorted.slice(sorted.length - cutoff);
  const surviving = sorted.slice(0, sorted.length - cutoff);

  const newWinners: WinnerRecord[] = [];
  for (const result of surviving) {
    if (result.averageQuality >= WINNER_THRESHOLD) {
      newWinners.push(
        createWinnerRecord(result.seed, epochIndex, result.averageQuality)
      );
    }
  }

  const survivingSeeds = surviving.map(r => r.seed);
  const updatedCandidates = [
    ...survivingSeeds,
    ...replacementCandidates,
  ].slice(0, MAX_CANDIDATE_POOL);

  return {
    updatedPool: { candidates: updatedCandidates, epochSampled: epochIndex },
    newWinners,
    dropped: dropping.map(r => r.seedHash),
  };
}

/**
 * Computes a BiasDigest from benchmark results.
 * This is the ONLY function that translates learning output into a digest.
 */
export function computeBiasDigest(
  results: BenchmarkResult[],
  previousDigest: BiasDigest = 0n
): BiasDigest {
  if (results.length === 0) return previousDigest;

  const topPerformers = results
    .filter(r => r.averageQuality >= WINNER_THRESHOLD)
    .sort((a, b) => b.averageQuality - a.averageQuality)
    .slice(0, 8);

  if (topPerformers.length === 0) return previousDigest;

  const buf = new ArrayBuffer(8 + topPerformers.length * 12);
  const view = new DataView(buf);
  view.setBigUint64(0, previousDigest, true);

  for (let i = 0; i < topPerformers.length; i++) {
    const offset = 8 + i * 12;
    view.setUint32(offset, topPerformers[i].seed.roleId, true);
    view.setUint32(offset + 4, topPerformers[i].seed.styleId, true);
    view.setUint32(offset + 8, Math.floor(topPerformers[i].averageQuality * 1000000), true);
  }

  return fnv64(new Uint8Array(buf));
}
```

### Layer 9: Boundary and Audit

#### File: `src/zero-specialist/boundary.ts`

```typescript
// src/zero-specialist/boundary.ts
// The ONLY import that learning-side files may reference.
// Contains no ZeroSpecialist primitives — only the shared output type.

export interface AgentOutputRecord {
  agentSeedHash: bigint;
  taskId: string;
  epochIndex: number;
  responseText: string;
  qualityScore: number;
  taskType: string;
  durationMs: number;
}

export type BiasDigest = bigint;

/** @deprecated Use BiasDigest instead. */
export interface PatternBias {
  roleRegions: Array<{ roleId: number; density: number }>;
  styleRegions: Array<{ styleId: number; density: number }>;
}

/**
 * The PatternStore is the ONLY shared state between the learning system
 * and the ZeroSpecialist selection system.
 */
export interface PatternStore {
  biasDigest: BiasDigest;
  lastUpdatedEpoch: number;
}
```

#### File: `src/zero-specialist/audit.ts`

```typescript
// src/zero-specialist/audit.ts
// Prompt determinism verification — static analysis and runtime checks.

import { generateSystemPrompt, assertPromptDeterminism } from './prompt-generator';
import type { AgentSeed } from './types';

const ANTI_PATTERNS = [
  /Math\.random\(\)/g,
  /Date\.now\(\)/g,
  /new Date\(\)/g,
  /crypto\.randomUUID\(\)/g,
  /localStorage/g,
  /sessionStorage/g,
  /process\.env\./g,
] as const;

/**
 * Static audit: scans source text for anti-patterns.
 * Optionally accepts excludePatterns to skip lines matching integration-layer patterns.
 */
export function auditSourceForAntiPatterns(
  sourceCode: string,
  options?: {
    excludePatterns?: RegExp[];
  }
): {
  passed: boolean;
  violations: Array<{ pattern: string; lineNumber: number; excerpt: string }>;
} {
  const violations: Array<{ pattern: string; lineNumber: number; excerpt: string }> = [];
  const lines = sourceCode.split('\n');
  const excludes = options?.excludePatterns ?? [];

  for (const [index, line] of lines.entries()) {
    if (excludes.some(ex => { const r = ex.test(line); ex.lastIndex = 0; return r; })) continue;

    for (const pattern of ANTI_PATTERNS) {
      if (pattern.test(line)) {
        violations.push({
          pattern: pattern.toString(),
          lineNumber: index + 1,
          excerpt: line.trim().slice(0, 80),
        });
      }
      pattern.lastIndex = 0;
    }
  }

  return { passed: violations.length === 0, violations };
}

/**
 * Scoped audit: scans only the pure zero-specialist library code.
 */
export function auditPureLibrary(
  sourceFiles: Array<{ filename: string; content: string }>
): {
  passed: boolean;
  violations: Array<{ filename: string; pattern: string; lineNumber: number; excerpt: string }>;
} {
  const allViolations: Array<{ filename: string; pattern: string; lineNumber: number; excerpt: string }> = [];

  for (const file of sourceFiles) {
    const result = auditSourceForAntiPatterns(file.content);
    for (const v of result.violations) {
      allViolations.push({ filename: file.filename, ...v });
    }
  }

  return { passed: allViolations.length === 0, violations: allViolations };
}

/**
 * Runtime determinism verification suite.
 */
export function runDeterminismSuite(testSeeds: AgentSeed[]): {
  passed: boolean;
  failures: string[];
} {
  const failures: string[] = [];

  for (const seed of testSeeds) {
    try {
      assertPromptDeterminism(seed, 10);
    } catch (e) {
      failures.push((e as Error).message);
    }
  }

  const prompts = testSeeds.map(s => generateSystemPrompt(s));
  const unique = new Set(prompts);
  if (unique.size < prompts.length * 0.8) {
    failures.push(
      `Insufficient uniqueness: ${unique.size} unique prompts from ${prompts.length} seeds`
    );
  }

  return { passed: failures.length === 0, failures };
}
```

### Profile System

#### File: `src/zero-specialist/profile.ts`

This file contains the profile loader, validator, hash function, and the hardcoded default extended profile. The `DEFAULT_EXTENDED_PROFILE` contains the exact same 10 role archetypes, 10 style modifiers, 8 reasoning approaches as the hardcoded prompt-generator, plus 3 entries per tiered instruction pool and 3 prompt templates.

```typescript
// src/zero-specialist/profile.ts
// Profile loading, validation, and the built-in default extended profile.
// Pure functions only. No side effects. No external state reads.

import type { ZeroSpecialistProfile } from './types';
import { fnv64 } from './hash';

const REQUIRED_POOLS: string[] = [
  'role_archetypes', 'style_modifiers', 'reasoning_approaches',
  'verbosity_high', 'verbosity_mid', 'verbosity_low',
  'assertiveness_high', 'assertiveness_mid', 'assertiveness_low',
  'formality_high', 'formality_mid', 'formality_low',
  'domain_focus_high', 'domain_focus_low',
  'reasoning_depth_high', 'reasoning_depth_low',
  'swarm_context',
];

const MIN_POOL_SIZES: Record<string, number> = {
  role_archetypes: 8,
  style_modifiers: 8,
  reasoning_approaches: 6,
};

export function validateProfile(json: unknown): { valid: true } | { valid: false; error: string } {
  if (typeof json !== 'object' || json === null) {
    return { valid: false, error: 'Profile must be a JSON object' };
  }
  const obj = json as Record<string, unknown>;
  if (typeof obj.name !== 'string' || obj.name.length === 0) {
    return { valid: false, error: 'Profile must have a non-empty "name" string' };
  }
  if (typeof obj.description !== 'string') {
    return { valid: false, error: 'Profile must have a "description" string' };
  }
  if (obj.type !== 'zerospecialist') {
    return { valid: false, error: 'Profile "type" must be "zerospecialist"' };
  }
  if (typeof obj.pools !== 'object' || obj.pools === null) {
    return { valid: false, error: 'Profile must have a "pools" object' };
  }
  const pools = obj.pools as Record<string, unknown>;
  for (const key of REQUIRED_POOLS) {
    if (!Array.isArray(pools[key])) {
      return { valid: false, error: `Missing or invalid pool: "${key}" must be a string array` };
    }
    const arr = pools[key] as unknown[];
    if (arr.length === 0) {
      return { valid: false, error: `Pool "${key}" must not be empty` };
    }
    if (!arr.every(item => typeof item === 'string')) {
      return { valid: false, error: `Pool "${key}" must contain only strings` };
    }
  }
  for (const [key, minSize] of Object.entries(MIN_POOL_SIZES)) {
    const arr = pools[key] as string[];
    if (arr.length < minSize) {
      return { valid: false, error: `Pool "${key}" has ${arr.length} entries, minimum is ${minSize}` };
    }
  }
  if (!Array.isArray(obj.templates) || obj.templates.length === 0) {
    return { valid: false, error: 'Profile must have a non-empty "templates" array' };
  }
  for (let i = 0; i < (obj.templates as unknown[]).length; i++) {
    if (typeof (obj.templates as unknown[])[i] !== 'string') {
      return { valid: false, error: `Template at index ${i} must be a string` };
    }
  }
  return { valid: true };
}

export function loadProfile(jsonString: string): ZeroSpecialistProfile {
  let parsed: unknown;
  try { parsed = JSON.parse(jsonString); }
  catch (e) { throw new Error(`Profile JSON parse error: ${(e as Error).message}`); }
  const result = validateProfile(parsed);
  if (!result.valid) throw new Error(result.error);
  return parsed as ZeroSpecialistProfile;
}

export function profileHash(profile: ZeroSpecialistProfile): number {
  const content = JSON.stringify({ pools: profile.pools, templates: profile.templates });
  const encoder = new TextEncoder();
  const bytes = encoder.encode(content);
  const h = fnv64(bytes);
  return Number(h & 0xFFFFFFFFn);
}

export const DEFAULT_EXTENDED_PROFILE_ID = 'default-extended';

export const DEFAULT_EXTENDED_PROFILE: ZeroSpecialistProfile = {
  name: "Default Extended",
  description: "Built-in profile matching the original ZeroSpecialist vocabulary with additional template and instruction variants.",
  version: "1.0.0",
  type: "zerospecialist",
  pools: {
    role_archetypes: [
      "analyst", "synthesizer", "critic", "explorer", "implementer",
      "strategist", "communicator", "researcher", "validator", "integrator",
    ],
    style_modifiers: [
      "concise", "thorough", "direct", "exploratory", "structured",
      "narrative", "systematic", "adaptive", "precise", "generative",
    ],
    reasoning_approaches: [
      "first-principles", "analogical", "deductive", "inductive",
      "abductive", "comparative", "causal", "probabilistic",
    ],
    verbosity_high: [
      "Provide comprehensive, detailed responses.",
      "Elaborate fully on each sub-point.",
      "Document your reasoning exhaustively.",
    ],
    verbosity_mid: [
      "Balance depth and conciseness based on task complexity.",
      "Scale response length to task scope.",
      "Expand where it adds value, trim where it does not.",
    ],
    verbosity_low: [
      "Be concise. Prioritize brevity over completeness.",
      "One clear answer. No padding.",
      "Short and direct. Omit what can be inferred.",
    ],
    assertiveness_high: [
      "State conclusions with confidence. Avoid hedging.",
      "Commit to a position. Revise only on new evidence.",
      "Lead with the answer. Justify afterward.",
    ],
    assertiveness_mid: [
      "Calibrate confidence to evidence quality.",
      "Express certainty proportional to supporting data.",
      "Qualify where genuinely uncertain; commit where evidence is clear.",
    ],
    assertiveness_low: [
      "Acknowledge uncertainty explicitly. Offer alternatives.",
      "Frame conclusions as provisional. Invite correction.",
      "Surface competing interpretations before settling.",
    ],
    formality_high: [
      "Use formal, precise language.",
      "Maintain professional register throughout.",
      "Technical terminology preferred where unambiguous.",
    ],
    formality_mid: [
      "Match register to the task context.",
      "Shift tone as the material demands.",
      "Professional but not stiff; clear without being casual.",
    ],
    formality_low: [
      "Use direct, accessible language.",
      "Plain speech over jargon.",
      "Speak plainly. Avoid ceremony.",
    ],
    domain_focus_high: [
      "Focus expertise deeply on the immediate task domain. Resist scope expansion.",
      "Stay within the task boundary. Depth over breadth.",
      "Domain mastery requires ignoring adjacent distractions.",
    ],
    domain_focus_low: [
      "Draw connections across domains. Generalist perspective is an asset.",
      "Cross-domain links often carry the highest signal.",
      "Lateral thinking is your edge.",
    ],
    reasoning_depth_high: [
      "Work through problems deliberately. Show reasoning steps.",
      "Slow down. Surface assumptions before conclusions.",
      "Deep deliberation reduces downstream error.",
    ],
    reasoning_depth_low: [
      "Prioritize speed. Use heuristics where appropriate.",
      "Fast heuristics over slow proofs where stakes allow.",
      "Responsive over exhaustive.",
    ],
    swarm_context: [
      "You coordinate with peer specialists through environmental signals. Your role is to contribute your distinct perspective, not to reach consensus. Disagreement with peers is valuable signal.",
      "You operate within a stigmergic swarm. Write to the shared signal field, not to the other agents directly. Your output shapes the environment others read.",
      "Peer specialists share this task space. Redundancy is waste — diverge from what others are likely to produce.",
    ],
  },
  templates: [
    "You are a {style_modifiers} {role_archetypes} AI specialist operating within a stigmergic coordination swarm.\n\nYour reasoning approach: {reasoning_approaches}.\n{verbosity}\n{assertiveness}\n{formality}\n{domain_focus}\n{reasoning_depth}\n\n{swarm_context}\n\nIdentity: Agent-{identity_tag}",
    "You are an {role_archetypes} operating in {style_modifiers} mode within a distributed specialist network.\n\nReasoning stance: {reasoning_approaches}.\n{assertiveness}\n{verbosity}\n{domain_focus}\n{formality}\n{reasoning_depth}\n\n{swarm_context}\n\nAgent signature: {identity_tag}",
    "{style_modifiers} {role_archetypes}. Swarm operative.\n\nApproach: {reasoning_approaches}.\n{reasoning_depth}\n{assertiveness}\n{verbosity}\n{formality}\n{domain_focus}\n\n{swarm_context}\n\n[{identity_tag}]",
  ],
  rules: {
    min_pool_size: { role_archetypes: 8, style_modifiers: 8, reasoning_approaches: 6 },
    required_pools: REQUIRED_POOLS,
    identity_tag_source: "agentHash_hex8",
  },
};
```

### Barrel Export

#### File: `src/zero-specialist/index.ts`

```typescript
// src/zero-specialist/index.ts
// Public API for the ZeroSpecialist deterministic agent system.

// Types
export type {
  AgentSeed, WinnerRecord, AgentOutputRecord, CandidatePool,
  PatternBias, BiasDigest, CapabilityProfile, PerformanceEnvelope,
  BenchmarkResult, RotationResult, PatternStore,
  ZeroSpecialistProfile, ProfileRecord,
} from './types';

// Layer 1 — Agent Identity Hashing
export { fnv64, agentHash, hashToFloat, hashToFloatChannel, deriveChildHash } from './hash';

// Layer 2 — Capability Field
export { capabilityField } from './capability-field';

// Layer 3 — Inter-Agent Affinity
export { agentAffinity, rankByAffinity } from './affinity';

// Layer 4 — Performance Epoch Tracking
export { performanceEnvelope, queryHistoricalPerformance } from './epoch-tracking';

// Layer 5 — Winner Lineage and Evolution Chain
export { agentAtDepth, reconstructWinner, forkBranch, createWinnerRecord } from './evolution-chain';

// Layer 6 — Stigmergic Signal Field
export { signalStrength, taskTypeToIndex, accumulatedSignalField } from './signal-field';

// Layer 7 — Prompt Generation (Deterministic Path)
export { generateSystemPrompt, assertPromptDeterminism } from './prompt-generator';

// Layer 8 — Selection Protocol
export { sample, benchmark, rotate, computeBiasDigest } from './selection-protocol';

// Layer 9 — Boundary (for learning system consumption)
export type { PatternStore as BoundaryPatternStore } from './boundary';
export type { AgentOutputRecord as BoundaryAgentOutputRecord } from './boundary';

// Profile
export {
  validateProfile, loadProfile, profileHash,
  DEFAULT_EXTENDED_PROFILE, DEFAULT_EXTENDED_PROFILE_ID,
} from './profile';

// Audit
export { auditSourceForAntiPatterns, runDeterminismSuite, auditPureLibrary } from './audit';
```

### Integration Layer: Epoch-Based Stigmergic Board

The `StigmergicBoard` in `src/core/stigmergic-board.ts` uses `epochIndex` (never `Date.now()`) for all signal timestamps. Signals are deposited at an epoch, read at an epoch, and decay as a function of `(currentEpoch - depositEpoch) * EPOCH_DURATION_SECONDS`. The `EPOCH_DURATION_SECONDS` constant (600 = 10 minutes) maps epoch count to wall-clock-equivalent seconds for the two-tier decay formula.

The `exponentialDecay` utility in `src/core/utils.ts` computes `value * exp(-ageSeconds / decayRate)`.

The board uses two-tier decay: signals less than 1 hour old use short-term decay (1800s), signals older than 1 hour first decay through the short-term rate for 1 hour, then through the long-term rate (7200s) for the remainder. Signal amplification factor is 1.5 (boosted when same agent or high quality), attenuation factor is 0.7 (reduced otherwise). Cleanup is called on demand per epoch rotation, not via `setInterval`.

### Integration Layer: ZeroSpecialist Selection

The `ZeroSpecialistLayer` in `src/core/zero-specialist-layer.ts` bridges pure functions with the Zustand store. It reads `worldSeed`, `candidatePool`, `lockedWinners`, `epochIndex` from the store, scores candidates via `scoreCandidate()` using keyword-matching against capability profiles, applies +0.15 winner bonus matched by full 64-bit agent hash via `reconstructWinner()`, and generates system prompts with optional profile support.

The `scoreCandidate()` function assigns a base score of 0.5 and adds dimension-weighted bonuses based on regex keyword matching: code/implementation keywords boost domain focus (+0.20), brevity/depth keywords align verbosity (+0.15), reasoning depth bonuses for complex tasks (+0.15), formality bonuses (+0.10), complexity bonuses for long prompts (+0.10), and assertiveness bonuses for questions (+0.05).

### Integration Layer: Three-Layer Orchestrator

The `HybridSwarmOrchestrator` in `src/core/hybrid-orchestrator.ts` coordinates all three layers. For each task:
1. Layer 1 selects a specialist via `ZeroSpecialistLayer.selectSpecialist(task)`
2. Layer 2 matches approaches via `DynamicApproachManager.matchApproaches(task)`
3. Layer 3 blends approach scores with stigmergic signals: `0.7 * patternScore + 0.3 * (signalStrength / 100)`

The orchestrator passes `epochIndex` from the specialist store to all signal operations. Execution results update all three layers: specialist feedback, signal deposits, and approach metrics.

### State Persistence: Zustand Store with BigInt Serialization

The specialist store in `src/stores/specialist-store.ts` uses Zustand `persist` middleware with a custom localStorage adapter that serializes/deserializes BigInt values via `toString()`/`BigInt()` round-trip. The store holds: `worldSeed`, `epochIndex`, `candidatePool`, `lockedWinners`, `biasDigest`, `profileEnabled`, `activeProfileId`, and `initialized`.

### Database: Dexie IndexedDB with 4 Schema Versions

The database in `src/storage/db.ts` uses Dexie with progressive schema versions:
- **Version 1:** Original tables (specialists, approaches, signals with timestamp, executionHistory, patterns, promptTemplates)
- **Version 2:** Adds ZeroSpecialist tables (worldConfig, winnerRecords, epochLog)
- **Version 3:** Migrates signals from `timestamp` index to `epochDeposited` with upgrade function converting wall-clock ages to epoch estimates
- **Version 4:** Adds `profiles` table for custom JSON profiles

### Signal Storage

The `SignalsStore` in `src/storage/signals-store.ts` provides epoch-based CRUD: `getRecentSignals(currentEpoch, epochWindow)`, `deleteOldSignals(currentEpoch, epochThreshold)`, `getSignalsInRange(currentEpoch, epochWindow)` — all using `epochDeposited` index queries instead of wall-clock timestamps.

### Profile Storage

The `ProfilesStore` in `src/storage/profiles-store.ts` provides simple Dexie CRUD: `saveProfile`, `getProfile`, `getAllProfiles`, `deleteProfile`, `count`.

### Settings UI: Profile Settings

The `ProfileSettings` component in `src/components/Settings/ProfileSettings.tsx` provides:
- Toggle: Enable/disable profile override
- Dropdown: Select from "Default Extended" (built-in) + uploaded profiles
- Upload: File picker -> JSON validation -> Dexie save -> auto-select
- Delete: Per-profile delete (cannot delete built-in)
- Preview: Read-only card showing name, description, pool sizes, template count

The `SettingsPanel` in `src/components/Settings/SettingsPanel.tsx` organizes 5 tabs (API, System, Templates, Profiles, Database) using shadcn/ui Tabs components in a `grid-cols-5` layout.

### Signal Type (Core)

The `Signal` interface in `src/core/types.ts` uses `epochDeposited: number` (epoch index) instead of wall-clock timestamps. The deprecated `timestamp?: number` field is kept for migration compatibility only.

```typescript
export interface Signal {
  taskId: string;
  approach: string;
  strength: number;
  epochDeposited: number;        // Epoch index when signal was deposited
  depositedBy: string;
  successMetric: number;
  id?: string;
  coordinates?: { x: number; y: number; intensity: number };
  type?: 'approach' | 'specialist' | 'environment' | 'pattern';
  connections?: string[];
  /** @deprecated Use epochDeposited instead. */
  timestamp?: number;
}
```

### Constants Reference

| Constant | Value | Location | Purpose |
|----------|-------|----------|---------|
| FNV_PRIME | 1099511628211n | hash.ts | FNV-1a multiplication constant |
| FNV_OFFSET | 14695981039346656037n | hash.ts | FNV-1a initial offset basis |
| MASK64 | 0xFFFFFFFFFFFFFFFFn | hash.ts | 64-bit overflow mask |
| MAX_CANDIDATE_POOL | 64 | selection-protocol.ts | Hard cap on candidates per epoch |
| WINNER_THRESHOLD | 0.75 | selection-protocol.ts | Minimum quality for winner promotion |
| ROTATION_BOTTOM_QUARTILE | 0.25 | selection-protocol.ts | Fraction dropped each rotation |
| Noise octaves | 4 | capability-field.ts | Multi-scale coherence levels |
| Amplitude decay | 0.5 | capability-field.ts | Per-octave amplitude reduction |
| Frequency multiplier | 2.0 | capability-field.ts | Per-octave frequency doubling |
| Coordinate scale | 8.0 | capability-field.ts | roleId/styleId normalisation range |
| Salt spacing | 1000 | capability-field.ts | Dimension independence factor |
| Task type buckets | 256 | signal-field.ts | Hash modulo for task categorisation |
| EPOCH_DURATION_SECONDS | 600 | stigmergic-board.ts | 10 minutes per epoch |
| Short-term decay | 1800s | stigmergic-board.ts | 30-minute fast decay |
| Long-term decay | 7200s | stigmergic-board.ts | 2-hour slow decay |
| Amplification factor | 1.5 | stigmergic-board.ts | Signal boost on success |
| Attenuation factor | 0.7 | stigmergic-board.ts | Signal reduction on failure |
| Winner score bonus | 0.15 | zero-specialist-layer.ts | Proven performer advantage |
| Pattern blend ratio | 70/30 | hybrid-orchestrator.ts | Pattern score vs signal weight |
| TIER_SALT_MAP | 10-14 | prompt-generator.ts | Child hash salts for dimension selection |
| TEMPLATE_SALT | 20 | prompt-generator.ts | Template selection hash salt |
| SWARM_CONTEXT_SALT | 21 | prompt-generator.ts | Swarm context selection hash salt |

### Test Suite: 9-Layer + Profile Determinism Verification

The test suite in `src/zero-specialist/__tests__/determinism.test.ts` uses Vitest with 4 standard test seeds:

```typescript
const TEST_SEEDS: AgentSeed[] = [
  { roleId: 0,     styleId: 0,     capabilitySalt: 0,     worldSeed: 1n },
  { roleId: 32767, styleId: 32767, capabilitySalt: 32767, worldSeed: 1n },
  { roleId: 65535, styleId: 65535, capabilitySalt: 65535, worldSeed: 1n },
  { roleId: 1000,  styleId: 2000,  capabilitySalt: 3000,  worldSeed: 9999999999n },
];
```

**Test coverage by layer:**

| Layer | Module | Key Assertions |
|-------|--------|---------------|
| 1 | hash.ts | FNV-1a consistency, uniqueness, distribution, child derivation |
| 2 | capability-field.ts | Determinism, value ranges [0,1], adjacent coherence |
| 3 | affinity.ts | Symmetry, value range [0,1], ranking order |
| 4 | epoch-tracking.ts | Determinism, historical query stability, valid ranges |
| 5 | evolution-chain.ts | Bounded deltas, round-trip reconstruction, branch divergence |
| 6 | signal-field.ts | Determinism, value range [0,1], task type hashing, accumulated field |
| 7 | prompt-generator.ts | 10-trial determinism, cross-seed uniqueness, structural elements |
| 8 | selection-protocol.ts | Sample determinism, pool cap, no duplicates, rotation quartile drop, biasDigest |
| 9 | audit.ts | Anti-pattern detection, clean code pass-through, pure library audit |
| Profile | profile.ts + prompt-generator.ts | Profile determinism, null/undefined parity, validation, hash consistency |

### Reconstruction Checklist

To rebuild ZeroSwarm from scratch in any TypeScript/React/Vite environment:

1. **Scaffold project** — `npm create vite@latest` with React + TypeScript template
2. **Install dependencies** — `zustand`, `dexie`, `@tanstack/react-query`, `react-router-dom`, shadcn/ui setup, `lucide-react`, `nanoid`
3. **Implement `src/zero-specialist/types.ts`** — All interfaces exactly as specified. AgentSeed must use bigint for worldSeed.
4. **Implement `src/zero-specialist/hash.ts`** — FNV-1a with exact constants. Verify: `fnv64(new Uint8Array([1,2,3,4]))` produces a stable bigint.
5. **Implement `src/zero-specialist/capability-field.ts`** — gridHash, coherentNoise (4 octaves), capabilityField. Verify: `capabilityField(1000, 2000, 1n)` returns identical profile across runs.
6. **Implement `src/zero-specialist/affinity.ts`** — Sorted pair hashing with polarity channel. Verify: symmetry property.
7. **Implement `src/zero-specialist/epoch-tracking.ts`** — Temporal hash with performance envelope derivation.
8. **Implement `src/zero-specialist/evolution-chain.ts`** — Delta derivation, winner packing/unpacking, reconstruction. Verify: round-trip at depth 0 preserves prompt.
9. **Implement `src/zero-specialist/signal-field.ts`** — Signal strength, task type indexing, accumulated signal field with geometric series.
10. **Implement `src/zero-specialist/selection-protocol.ts`** — sample, benchmark, rotate, computeBiasDigest with exact constants.
11. **Implement `src/zero-specialist/boundary.ts`** — Type-only module. No logic.
12. **Implement `src/zero-specialist/prompt-generator.ts`** — Hardcoded + profile-driven paths with selectFromTier helper.
13. **Implement `src/zero-specialist/profile.ts`** — Validator, loader, hash, DEFAULT_EXTENDED_PROFILE constant.
14. **Implement `src/zero-specialist/audit.ts`** — Anti-pattern scanner, pure library audit, runtime determinism suite.
15. **Implement `src/zero-specialist/index.ts`** — Public API barrel export.
16. **Implement `src/core/types.ts`** — Signal interface with epochDeposited, TaskContext, CoordinationResult, etc.
17. **Implement `src/core/utils.ts`** — exponentialDecay function.
18. **Implement `src/core/stigmergic-board.ts`** — Epoch-based StigmergicBoard class with two-tier decay.
19. **Implement `src/core/zero-specialist-layer.ts`** — ZeroSpecialistLayer class with scoreCandidate, profile loading, winner bonus.
20. **Implement `src/core/hybrid-orchestrator.ts`** — Three-layer orchestration with signal blending.
21. **Implement `src/storage/db.ts`** — Dexie database with 4 schema versions and migration.
22. **Implement `src/storage/signals-store.ts`** — Epoch-based signal CRUD.
23. **Implement `src/storage/profiles-store.ts`** — Profile CRUD.
24. **Implement `src/stores/specialist-store.ts`** — Zustand store with BigInt serialization.
25. **Implement `src/components/Settings/ProfileSettings.tsx`** — Profile toggle/upload/preview UI.
26. **Implement `src/components/Settings/SettingsPanel.tsx`** — 5-tab settings container.
27. **Run test suite** — All 9 layers + profile tests must pass. Cross-seed uniqueness >= 80%.

### Why 128 Bits Replaces 3000 Tokens

| Dimension | Traditional Prompt | ZeroSwarm |
|-----------|--------------------|-----------|
| Storage per agent | ~4,000 bytes | 16 bytes (128 bits) |
| Reproducibility | Session-dependent | Deterministic forever |
| Variation space | Manually enumerated | 2^48 base + 2^160 profile |
| Evolution tracking | Lost on edit | Full lineage from 128-bit ref |
| Cross-machine identity | Impossible | Guaranteed by pure functions |
| Prompt bloat over time | Grows linearly | Constant 16 bytes per winner |

The 128-bit seed encodes position in a continuous behavioral space. The prompt is not compressed into 128 bits — it is *generated* from 128 bits through a deterministic pipeline of hashing, coherent noise, and template assembly. The information is not in the bits themselves but in the shared algorithm that interprets them.
