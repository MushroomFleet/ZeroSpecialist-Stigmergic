# ZeroSpecialist: 128-Bit Procedural System Prompt Generation

## Description

ZeroSpecialist is a deterministic system prompt generation architecture that replaces traditional 3000+ token handwritten system prompts with 128-bit seed references. A four-integer tuple (roleId, styleId, capabilitySalt, worldSeed) is hashed through FNV-1a and fed into octave-layered coherent noise to produce a unique AI specialist persona with five continuous behavioral dimensions, three categorical selectors, and a natural-language system prompt. The entire agent identity is reconstructable from 16 bytes. No prompt text is ever stored. The system operates as a pure-function pipeline: same seed in, identical prompt out, on any machine, in any session, forever.

The architecture is organised into nine auditable layers: identity hashing, capability field, inter-agent affinity, epoch tracking, evolution chains, stigmergic signal fields, prompt generation, selection protocol, and determinism audit. Each layer is a pure function with zero side effects. A three-layer orchestrator (ZeroSpecialist selection, dynamic approach matching, stigmergic signal blending) coordinates specialist assignment at runtime. Candidates are sampled, benchmarked, and rotated through an evolutionary protocol that promotes high-quality performers to locked winners and drops the bottom quartile each epoch.

This document provides complete standalone reconstruction instructions: all algorithms, all data structures, all code, all constants. No external dependencies beyond TypeScript and a hash function are required to rebuild the core.

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

**4. Evolutionary Selection Protocol (Sample-Benchmark-Rotate)**
- `sample(n, worldSeed, epochIndex, bias?)` generates up to 64 candidate AgentSeeds per epoch
- `benchmark(outputs, candidates)` groups quality scores by agent hash and averages
- `rotate(results, pool, winners, epoch, replacements)` drops bottom 25%, promotes candidates scoring >= 0.75 to winners, fills gaps with fresh samples biased toward high-performing regions
- Pattern learning feeds back as `PatternBias` to concentrate future sampling

**5. O(1) Auxiliary Queries**
- **Affinity:** Symmetric pairwise agent affinity in [0, 1] via sorted pair hashing with polarity channel
- **Epoch Tracking:** Performance envelope at any past/future epoch without replaying history
- **Signal Field:** Stigmergic signal strength at any (taskType, agentRole, epoch) coordinate
- **Evolution Chain:** Agent deltas at any depth via child hash derivation; full reconstruction from 128-bit winner reference

**6. Three-Layer Orchestration**
- Layer 1 (ZeroSpecialist): Scores all candidates against task context using keyword matching against capability profiles; locked winners receive +0.15 bonus
- Layer 2 (Dynamic Approaches): Pattern signature matching selects approach templates (unchanged from host system)
- Layer 3 (Stigmergic Board): Two-tier exponential decay signal blending (30-min short-term, 2-hour long-term); amplification factor 1.5, attenuation factor 0.7; approach selection blends 70% pattern score + 30% signal strength

**7. Nine-Layer Determinism Audit**
- Static analysis scans for anti-patterns: `Math.random()`, `Date.now()`, `crypto.randomUUID()`, `localStorage`, `sessionStorage`, `process.env`
- Runtime suite runs 10 trials per seed confirming identical output
- Cross-seed uniqueness check verifies >= 80% unique prompts from test seed set

### User Interface Behaviour
- Specialists displayed as cards showing ID (`zs-{hash8}`), quality score, specialization vector, execution count
- Human-readable names use `domain_animal` format with honorific progression based on performance tier
- Dashboard shows candidate pool, locked winners, epoch index, pattern bias regions

### State Management
- `worldSeed` generated once on first launch, persisted permanently, never changes
- `epochIndex` incremented on each rotation cycle
- `candidatePool` (max 64 AgentSeeds) and `lockedWinners` (WinnerRecord[]) persisted via Zustand with localStorage adapter
- BigInt serialization handled via toString/BigInt() round-trip in custom storage adapter
- `PatternBias` written by learning layer, read by selection protocol — the only shared state between learning and identity systems

## Technical Implementation

### Architecture Overview

```
src/zero-specialist/          ← Pure functions only. Zero UI/state imports.
  types.ts                    ← All type definitions
  hash.ts                     ← FNV-1a 64-bit hashing primitives
  capability-field.ts         ← Coherent noise capability mapping
  affinity.ts                 ← Symmetric pairwise affinity
  epoch-tracking.ts           ← O(1) temporal performance envelopes
  evolution-chain.ts          ← Winner lineage and reconstruction
  signal-field.ts             ← O(1) stigmergic field queries
  prompt-generator.ts         ← Deterministic system prompt assembly
  selection-protocol.ts       ← Sample/Benchmark/Rotate protocol
  boundary.ts                 ← Decoupling interface for learning system
  audit.ts                    ← Static + runtime determinism verification
  index.ts                    ← Public API barrel export
  __tests__/determinism.test.ts ← 9-layer test suite

src/core/                     ← Integration layer (imports zero-specialist + stores)
  zero-specialist-layer.ts    ← Bridges pure functions to Zustand store
  hybrid-orchestrator.ts      ← Three-layer coordination engine
  stigmergic-board.ts         ← Signal deposit/read with two-tier decay

src/stores/
  specialist-store.ts         ← Zustand store with BigInt serialization
```

**Critical Constraint:** The `src/zero-specialist/` directory must never import from `src/components/`, `src/stores/`, or any UI framework. It is a pure-function library. The `boundary.ts` file is the ONLY type that learning-side code may reference.

### Data Structures

```typescript
// === Core Identity ===

interface AgentSeed {
  roleId: number;           // 0-65535: position in role space
  styleId: number;          // 0-65535: position in style space
  capabilitySalt: number;   // 0-65535: fine-grained variation within role+style
  worldSeed: bigint;        // 64-bit world constant, never changes per installation
}

interface WinnerRecord {
  originSeed: bigint;       // Packed: [roleId:16][styleId:16][capSalt:16][worldSeedLow:16]
  depth: number;            // Evolution depth (0 = unevolved winner)
  epochPromoted: number;    // Epoch index at time of promotion
  scoreAtPromotion: number; // Quality score [0,1] at promotion time
}

// === Capability Profile ===

interface CapabilityProfile {
  verbosity: number;        // [0,1] concise -> expansive
  assertiveness: number;    // [0,1] tentative -> confident
  formality: number;        // [0,1] casual -> formal
  domainFocus: number;      // [0,1] generalist -> specialist
  reasoningDepth: number;   // [0,1] fast heuristic -> deep deliberate
}

// === Selection Protocol ===

interface CandidatePool {
  candidates: AgentSeed[];  // Hard cap: N <= 64
  epochSampled: number;
}

interface PatternBias {
  roleRegions: Array<{ roleId: number; density: number }>;
  styleRegions: Array<{ styleId: number; density: number }>;
}

interface BenchmarkResult {
  seed: AgentSeed;
  seedHash: bigint;
  averageQuality: number;
  taskCount: number;
}

interface RotationResult {
  updatedPool: CandidatePool;
  newWinners: WinnerRecord[];
  dropped: bigint[];
}

// === Auxiliary ===

interface PerformanceEnvelope {
  expectedMin: number;      // [0,1]
  expectedMax: number;      // [0,1]
  consistency: number;      // [0,1]
}

interface SignalEvent {
  taskType: string;
  agentRole: number;
  epochIndex: number;
  strength: number;
}

interface AgentOutputRecord {
  agentSeedHash: bigint;
  taskId: string;
  epochIndex: number;
  responseText: string;
  qualityScore: number;     // [0,1]
  taskType: string;
  durationMs: number;
}

// === Boundary (learning <-> identity decoupling) ===

interface PatternStore {
  bias: PatternBias;
  lastUpdatedEpoch: number;
}
```

### Algorithm 1: FNV-1a 64-Bit Hashing

All identity derivation flows through this single hash function. It is the cryptographic root of the entire system.

```typescript
const FNV_PRIME = 1099511628211n;
const FNV_OFFSET = 14695981039346656037n;
const MASK64 = 0xFFFFFFFFFFFFFFFFn;

function fnv64(bytes: Uint8Array): bigint {
  let hash = FNV_OFFSET;
  for (const byte of bytes) {
    hash ^= BigInt(byte);
    hash = (hash * FNV_PRIME) & MASK64;
  }
  return hash;
}
```

**Constants are non-negotiable.** FNV_PRIME and FNV_OFFSET are standardised values for the 64-bit FNV-1a variant. Changing them breaks all determinism guarantees.

### Algorithm 2: Agent Seed Packing and Hashing

```typescript
// Pack AgentSeed into 24-byte deterministic representation
// Layout: [roleId:4][styleId:4][capabilitySalt:4][pad:4][worldSeed:8]
function packAgentSeed(seed: AgentSeed): Uint8Array {
  const buf = new ArrayBuffer(24);
  const view = new DataView(buf);
  view.setUint32(0, seed.roleId, true);       // little-endian
  view.setUint32(4, seed.styleId, true);
  view.setUint32(8, seed.capabilitySalt, true);
  // bytes 12-15 are padding (zero)
  view.setBigUint64(16, seed.worldSeed, true);
  return new Uint8Array(buf);
}

function agentHash(seed: AgentSeed): bigint {
  return fnv64(packAgentSeed(seed));
}
```

### Algorithm 3: Hash Utility Functions

```typescript
// Lower 32 bits to float in [0, 1)
function hashToFloat(h: bigint): number {
  return Number(h & 0xFFFFFFFFn) / 0x100000000;
}

// Decorrelated channel extraction via shift+XOR
function hashToFloatChannel(h: bigint, channel: number): number {
  const shifted = (h >> BigInt(channel * 8)) ^ h;
  return Number(shifted & 0xFFFFFFFFn) / 0x100000000;
}

// Child hash derivation for generating multiple values from one seed
function deriveChildHash(parentHash: bigint, childIndex: number): bigint {
  const buf = new ArrayBuffer(16);
  const view = new DataView(buf);
  view.setBigUint64(0, parentHash, true);
  view.setUint32(8, childIndex, true);
  return fnv64(new Uint8Array(buf));
}
```

### Algorithm 4: Coherent Capability Field

This maps the 2D seed coordinate to 5 continuous behavioral dimensions using octave-layered value noise.

```typescript
function smoothstep(t: number): number {
  return t * t * (3 - 2 * t);
}

// Deterministic grid hash at integer coordinates
function gridHash(gx: number, gy: number, salt: number, worldSeed: bigint): number {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setInt32(0, gx, true);
  view.setInt32(4, gy, true);
  view.setUint32(8, salt, true);
  view.setBigUint64(12, worldSeed, true);
  return hashToFloat(fnv64(new Uint8Array(buf)));
}

// 4-octave coherent value noise, returns approximately [-1, 1]
function coherentNoise(
  x: number, y: number, salt: number, worldSeed: bigint, octaves = 4
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

// Maps (roleX, styleY) to CapabilityProfile
// Scale: roleX and styleY are in range [0, 65535]
function capabilityField(roleX: number, styleY: number, worldSeed: bigint): CapabilityProfile {
  const nx = roleX / 65535 * 8;  // Normalize to noise frequency range [0, 8]
  const ny = styleY / 65535 * 8;

  const raw = (salt: number) => (coherentNoise(nx, ny, salt * 1000, worldSeed) + 1) / 2;

  return {
    verbosity:      raw(0),   // salt 0
    assertiveness:  raw(1),   // salt 1000
    formality:      raw(2),   // salt 2000
    domainFocus:    raw(3),   // salt 3000
    reasoningDepth: raw(4),   // salt 4000
  };
}
```

**Key design decisions:**
- Normalisation to `[0, 8]` controls the noise frequency: lower ranges produce smoother fields, higher ranges produce more variation
- Salt spacing of 1000 ensures complete independence between dimensions
- 4 octaves with 0.5 amplitude decay is the standard multi-scale balance
- Smoothstep interpolation prevents grid artefacts

### Algorithm 5: Deterministic Prompt Generation

```typescript
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

function generateSystemPrompt(seed: AgentSeed): string {
  const hash = agentHash(seed);
  const profile = capabilityField(seed.roleId, seed.styleId, seed.worldSeed);

  const archetypeIndex = Number(hash % BigInt(ROLE_ARCHETYPES.length));
  const styleIndex = Number(deriveChildHash(hash, 1) % BigInt(STYLE_MODIFIERS.length));
  const reasoningIndex = Number(deriveChildHash(hash, 2) % BigInt(REASONING_APPROACHES.length));

  const archetype = ROLE_ARCHETYPES[archetypeIndex];
  const style = STYLE_MODIFIERS[styleIndex];
  const reasoning = REASONING_APPROACHES[reasoningIndex];

  // Behavioral instructions derived from continuous capability profile
  const verbosityInstruction =
    profile.verbosity > 0.7 ? 'Provide comprehensive, detailed responses.' :
    profile.verbosity < 0.3 ? 'Be concise. Prioritize brevity over completeness.' :
    'Balance depth and conciseness based on task complexity.';

  const assertivenessInstruction =
    profile.assertiveness > 0.7 ? 'State conclusions with confidence. Avoid hedging.' :
    profile.assertiveness < 0.3 ? 'Acknowledge uncertainty explicitly. Offer alternatives.' :
    'Calibrate confidence to evidence quality.';

  const formalityInstruction =
    profile.formality > 0.7 ? 'Use formal, precise language.' :
    profile.formality < 0.3 ? 'Use direct, accessible language.' :
    'Match register to the task context.';

  const domainInstruction =
    profile.domainFocus > 0.7
      ? 'Focus expertise deeply on the immediate task domain. Resist scope expansion.'
      : 'Draw connections across domains. Generalist perspective is an asset.';

  const reasoningInstruction =
    profile.reasoningDepth > 0.7
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
```

**Prompt structure breakdown:**
1. Opening line: style + archetype + swarm context
2. Reasoning approach declaration
3. Five behavioral instructions (one per capability dimension)
4. Swarm coordination philosophy
5. Unique identity tag (first 8 hex chars of agent hash)

### Algorithm 6: Symmetric Pairwise Affinity

```typescript
function pairHash(seedA: bigint, seedB: bigint, salt: number): bigint {
  const [lo, hi] = seedA < seedB ? [seedA, seedB] : [seedB, seedA];
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setBigUint64(0, lo, true);
  view.setBigUint64(8, hi, true);
  view.setUint32(16, salt, true);
  return fnv64(new Uint8Array(buf));
}

// Returns collaboration affinity in [0, 1]
// 0.0 = antagonistic, 0.5 = neutral, 1.0 = highly collaborative
function agentAffinity(seedHashA: bigint, seedHashB: bigint): number {
  const base = hashToFloat(pairHash(seedHashA, seedHashB, 0));
  const polarity = hashToFloat(pairHash(seedHashA, seedHashB, 1));
  return polarity > 0.7 ? 0.5 + base * 0.5 :   // cooperative range [0.5, 1.0]
         polarity < 0.3 ? base * 0.5 :           // competitive range [0.0, 0.5]
         base;                                     // neutral range [0.0, 1.0)
}

function rankByAffinity(focalSeedHash: bigint, pool: bigint[]): bigint[] {
  return [...pool].sort(
    (a, b) => agentAffinity(focalSeedHash, b) - agentAffinity(focalSeedHash, a)
  );
}
```

### Algorithm 7: Temporal Epoch Tracking

```typescript
function temporalHash(agentSeedHash: bigint, epochIndex: number, worldSeed: bigint): bigint {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setBigUint64(0, agentSeedHash, true);
  view.setUint32(8, epochIndex, true);
  view.setBigUint64(12, worldSeed, true);
  return fnv64(new Uint8Array(buf));
}

function performanceEnvelope(
  agentSeedHash: bigint, epochIndex: number, worldSeed: bigint
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
```

### Algorithm 8: Evolution Chain and Winner Reconstruction

```typescript
// Derives seed deltas at a given evolution depth. O(1).
function agentAtDepth(originSeed: bigint, depth: number): {
  roleIdDelta: number; styleIdDelta: number; capabilitySaltDelta: number;
} {
  const depthHash = deriveChildHash(originSeed, depth);
  return {
    roleIdDelta:         Math.floor(hashToFloat(depthHash)        * 256) - 128,
    styleIdDelta:        Math.floor(hashToFloat(depthHash >> 16n) * 256) - 128,
    capabilitySaltDelta: Math.floor(hashToFloat(depthHash >> 32n) * 512) - 256,
  };
}

// Reconstructs full AgentSeed from a WinnerRecord (128-bit reference)
function reconstructWinner(record: WinnerRecord, baseWorldSeed: bigint): AgentSeed {
  const roleId  = Number((record.originSeed >> 48n) & 0xFFFFn);
  const styleId = Number((record.originSeed >> 32n) & 0xFFFFn);
  const capSalt = Number((record.originSeed >> 16n) & 0xFFFFn);

  const delta = agentAtDepth(record.originSeed, record.depth);

  return {
    roleId:         Math.max(0, Math.min(65535, roleId  + delta.roleIdDelta)),
    styleId:        Math.max(0, Math.min(65535, styleId + delta.styleIdDelta)),
    capabilitySalt: Math.max(0, Math.min(65535, capSalt + delta.capabilitySaltDelta)),
    worldSeed:      baseWorldSeed,
  };
}

// Packs AgentSeed fields into the high bits of originSeed for storage
function createWinnerRecord(
  seed: AgentSeed, epochPromoted: number, scoreAtPromotion: number, depth = 0
): WinnerRecord {
  const originSeed =
    (BigInt(seed.roleId)         << 48n) |
    (BigInt(seed.styleId)        << 32n) |
    (BigInt(seed.capabilitySalt) << 16n) |
    (seed.worldSeed & 0xFFFFn);

  return { originSeed, depth, epochPromoted, scoreAtPromotion };
}

// Alternate branch derivation for exploring parallel lineages
function forkBranch(originSeed: bigint, forkDepth: number, branchIndex: number): bigint {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setBigUint64(0, originSeed, true);
  view.setUint32(8, forkDepth, true);
  view.setUint32(12, branchIndex, true);
  return fnv64(new Uint8Array(buf));
}
```

**originSeed bit layout:**
```
Bits 63-48: roleId          (16 bits)
Bits 47-32: styleId         (16 bits)
Bits 31-16: capabilitySalt  (16 bits)
Bits 15-0:  worldSeed low   (16 bits, XOR portion)
```

### Algorithm 9: Stigmergic Signal Field

```typescript
function signalStrength(
  taskTypeIndex: number, agentRoleId: number, epochIndex: number, worldSeed: bigint
): number {
  const buf = new ArrayBuffer(20);
  const view = new DataView(buf);
  view.setUint32(0, taskTypeIndex, true);
  view.setUint32(4, agentRoleId, true);
  view.setUint32(8, epochIndex, true);
  view.setBigUint64(12, worldSeed, true);
  const h = fnv64(new Uint8Array(buf));
  return hashToFloat(h);
}

function taskTypeToIndex(taskType: string): number {
  const encoder = new TextEncoder();
  const bytes = encoder.encode(taskType);
  const h = fnv64(bytes);
  return Number(h & 0xFFn);  // 256 task type buckets
}

function regionSamplingBias(
  roleId: number, styleId: number, epochIndex: number, worldSeed: bigint,
  taskHistory: Array<{ taskType: string; agentRoleId: number; epochIndex: number }>
): number {
  let totalSignal = 0;
  const ROLE_RADIUS = 512;

  for (const event of taskHistory) {
    if (Math.abs(event.agentRoleId - roleId) < ROLE_RADIUS) {
      const typeIndex = taskTypeToIndex(event.taskType);
      totalSignal += signalStrength(typeIndex, roleId, epochIndex - event.epochIndex, worldSeed);
    }
  }

  return Math.min(1, totalSignal / Math.max(1, taskHistory.length));
}
```

### Algorithm 10: Selection Protocol (Sample-Benchmark-Rotate)

```typescript
const MAX_CANDIDATE_POOL = 64;
const WINNER_THRESHOLD = 0.75;
const ROTATION_BOTTOM_QUARTILE = 0.25;

function sample(
  n: number, worldSeed: bigint, epochIndex: number, bias?: PatternBias
): AgentSeed[] {
  const count = Math.min(n, MAX_CANDIDATE_POOL);
  const seeds: AgentSeed[] = [];

  for (let i = 0; i < count; i++) {
    const slotHash = deriveChildHash(worldSeed ^ BigInt(epochIndex * 1000 + i), i);

    let roleId  = Math.floor(hashToFloat(slotHash)        * 65536);
    let styleId = Math.floor(hashToFloat(slotHash >> 16n) * 65536);
    const capSalt = Math.floor(hashToFloat(slotHash >> 32n) * 65536);

    // Apply bias: nudge toward high-density regions
    if (bias) {
      const roleBias = bias.roleRegions.find(r => Math.abs(r.roleId - roleId) < 1000);
      const styleBias = bias.styleRegions.find(s => Math.abs(s.styleId - styleId) < 1000);
      if (roleBias && hashToFloat(slotHash >> 48n) < roleBias.density) {
        roleId = roleBias.roleId + (roleId % 500) - 250;
      }
      if (styleBias && hashToFloat(slotHash >> 56n) < styleBias.density) {
        styleId = styleBias.styleId + (styleId % 500) - 250;
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

function benchmark(
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

function rotate(
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
```

### Algorithm 11: Task-Driven Specialist Scoring

```typescript
function scoreCandidate(seed: AgentSeed, task: { prompt: string }): number {
  const profile = capabilityField(seed.roleId, seed.styleId, seed.worldSeed);

  const promptLower = task.prompt.toLowerCase();
  const wantsCode = /\b(code|implement|function|class|bug|fix|refactor|debug)\b/.test(promptLower);
  const wantsExplanation = /\b(explain|describe|what is|how does|why|teach)\b/.test(promptLower);
  const wantsBrevity = /\b(brief|quick|short|tldr|summary|concise)\b/.test(promptLower);
  const wantsDepth = /\b(detail|thorough|comprehensive|deep dive|elaborate)\b/.test(promptLower);
  const wantsFormal = /\b(formal|academic|professional|technical|paper)\b/.test(promptLower);

  let score = 0.5;  // Base score

  // Verbosity alignment
  if (wantsBrevity) score += (1 - profile.verbosity) * 0.15;
  else if (wantsDepth) score += profile.verbosity * 0.15;

  // Domain focus alignment
  if (wantsCode) score += profile.domainFocus * 0.2;
  else if (wantsExplanation) score += (1 - profile.domainFocus) * 0.1;

  // Reasoning depth alignment
  if (wantsDepth || wantsCode) score += profile.reasoningDepth * 0.15;
  else if (wantsBrevity) score += (1 - profile.reasoningDepth) * 0.15;

  // Formality alignment
  if (wantsFormal) score += profile.formality * 0.1;

  // Assertiveness bonus for direct questions
  if (promptLower.includes('?')) score += profile.assertiveness * 0.05;

  // Complexity bonus: longer prompts benefit from deeper reasoning
  const wordCount = task.prompt.split(/\s+/).length;
  if (wordCount > 50) score += profile.reasoningDepth * 0.1;

  return Math.min(1, Math.max(0, score));
}
```

**Score weight breakdown:**
| Factor | Max Weight | Condition |
|--------|-----------|-----------|
| Base | 0.50 | Always |
| Domain focus (code) | +0.20 | Keywords: code, implement, function, class, bug, fix, refactor, debug |
| Verbosity | +0.15 | Keywords: brief/short OR detail/thorough |
| Reasoning depth | +0.15 | Keywords: detail/thorough OR code keywords |
| Formality | +0.10 | Keywords: formal, academic, professional, technical, paper |
| Complexity | +0.10 | Prompt > 50 words |
| Assertiveness | +0.05 | Prompt contains "?" |
| Winner bonus | +0.15 | Applied post-scoring for locked winners |

### Algorithm 12: Determinism Audit

```typescript
const ANTI_PATTERNS = [
  /Math\.random\(\)/g,
  /Date\.now\(\)/g,
  /new Date\(\)/g,
  /crypto\.randomUUID\(\)/g,
  /localStorage/g,
  /sessionStorage/g,
  /process\.env\./g,
];

function auditSourceForAntiPatterns(sourceCode: string): {
  passed: boolean;
  violations: Array<{ pattern: string; lineNumber: number; excerpt: string }>;
} {
  const violations: Array<{ pattern: string; lineNumber: number; excerpt: string }> = [];
  const lines = sourceCode.split('\n');

  for (const [index, line] of lines.entries()) {
    for (const pattern of ANTI_PATTERNS) {
      if (pattern.test(line)) {
        violations.push({
          pattern: pattern.toString(),
          lineNumber: index + 1,
          excerpt: line.trim().slice(0, 80),
        });
      }
      pattern.lastIndex = 0;  // Reset regex state
    }
  }

  return { passed: violations.length === 0, violations };
}

function assertPromptDeterminism(seed: AgentSeed, trials = 5): void {
  const reference = generateSystemPrompt(seed);
  for (let i = 0; i < trials; i++) {
    const result = generateSystemPrompt(seed);
    if (result !== reference) {
      throw new Error(
        `Prompt determinism violation at trial ${i}. Seed: ${JSON.stringify(seed)}`
      );
    }
  }
}

function runDeterminismSuite(testSeeds: AgentSeed[]): {
  passed: boolean; failures: string[];
} {
  const failures: string[] = [];

  for (const seed of testSeeds) {
    try {
      assertPromptDeterminism(seed, 10);
    } catch (e) {
      failures.push((e as Error).message);
    }
  }

  // Cross-seed uniqueness: different seeds must produce different prompts
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

### Integration Layer: Three-Layer Orchestration

The orchestrator connects ZeroSpecialist selection with dynamic approaches and stigmergic signals.

```typescript
// Specialist selection with winner bonus
class ZeroSpecialistLayer {
  selectSpecialist(task: TaskContext): ZeroSpecialistSelection {
    const { worldSeed, candidatePool, lockedWinners, epochIndex } = getState();

    let candidates = candidatePool.candidates;
    if (candidates.length === 0) {
      candidates = sample(32, worldSeed, epochIndex);
    }

    // Score all candidates against task
    const scored = candidates.map(seed => ({
      seed,
      score: scoreCandidate(seed, task),
      hash: agentHash(seed),
    }));

    // Locked winners get +0.15 bonus
    const winnerHashes = new Set(
      lockedWinners.map(w => {
        const roleId = Number((w.originSeed >> 48n) & 0xFFFFn);
        const styleId = Number((w.originSeed >> 32n) & 0xFFFFn);
        return `${roleId}-${styleId}`;
      })
    );

    for (const entry of scored) {
      const key = `${entry.seed.roleId}-${entry.seed.styleId}`;
      if (winnerHashes.has(key)) {
        entry.score = Math.min(1, entry.score + 0.15);
      }
    }

    scored.sort((a, b) => b.score - a.score);
    const best = scored[0];

    return {
      specialistId: `zs-${best.hash.toString(16).slice(0, 8)}`,
      seed: best.seed,
      seedHash: best.hash,
      systemPrompt: generateSystemPrompt(best.seed),
      profile: capabilityField(best.seed.roleId, best.seed.styleId, best.seed.worldSeed),
    };
  }
}
```

### Integration Layer: Stigmergic Board (Two-Tier Decay)

```typescript
function exponentialDecay(value: number, ageSeconds: number, decayRate: number): number {
  return value * Math.exp(-ageSeconds / decayRate);
}

class StigmergicBoard {
  private shortTermDecay = 1800;    // 30 minutes
  private longTermDecay = 7200;     // 2 hours
  private amplificationFactor = 1.5;
  private attenuationFactor = 0.7;

  calculateDecayedStrength(signal: { strength: number; timestamp: number }): number {
    const ageSeconds = (Date.now() - signal.timestamp) / 1000;
    const oneHour = 3600;

    if (ageSeconds <= oneHour) {
      return exponentialDecay(signal.strength, ageSeconds, this.shortTermDecay);
    } else {
      const shortTermStrength = exponentialDecay(signal.strength, oneHour, this.shortTermDecay);
      return exponentialDecay(shortTermStrength, ageSeconds - oneHour, this.longTermDecay);
    }
  }

  // Signal blending: 70% pattern match + 30% signal strength
  selectWithSignals(
    matches: Array<{ approach: { id: string }; score: number }>,
    signals: Map<string, number>
  ): string {
    const scored = matches.map(({ approach, score }) => {
      const signalStrength = signals.get(approach.id) ?? 0;
      return { approachId: approach.id, score: 0.7 * score + 0.3 * (signalStrength / 100) };
    });

    scored.sort((a, b) => b.score - a.score);
    return scored[0].approachId;
  }
}
```

### State Persistence: Zustand Store with BigInt Serialization

```typescript
// BigInt cannot be JSON.stringify'd natively. Custom adapter required.
function serializeState(state: any): any {
  return {
    ...state,
    worldSeed: state.worldSeed?.toString() ?? null,
    candidatePool: {
      ...state.candidatePool,
      candidates: state.candidatePool.candidates.map((s: AgentSeed) => ({
        ...s,
        worldSeed: s.worldSeed.toString(),
      })),
    },
    lockedWinners: state.lockedWinners.map((w: WinnerRecord) => ({
      ...w,
      originSeed: w.originSeed.toString(),
    })),
  };
}

function deserializeState(stored: any): any {
  if (!stored) return stored;
  return {
    ...stored,
    worldSeed: stored.worldSeed ? BigInt(stored.worldSeed) : null,
    candidatePool: {
      ...stored.candidatePool,
      candidates: (stored.candidatePool?.candidates ?? []).map((s: any) => ({
        ...s,
        worldSeed: BigInt(s.worldSeed),
      })),
    },
    lockedWinners: (stored.lockedWinners ?? []).map((w: any) => ({
      ...w,
      originSeed: BigInt(w.originSeed),
    })),
  };
}
```

### Boundary Interface (Learning-Identity Decoupling)

The boundary module is the ONLY import that learning-side code may reference. It enforces strict separation between the identity generation system and the pattern learning system.

```typescript
// boundary.ts — exported types only, no ZeroSpecialist primitives

interface AgentOutputRecord {
  agentSeedHash: bigint;
  taskId: string;
  epochIndex: number;
  responseText: string;
  qualityScore: number;
  taskType: string;
  durationMs: number;
}

interface PatternBias {
  roleRegions: Array<{ roleId: number; density: number }>;
  styleRegions: Array<{ styleId: number; density: number }>;
}

interface PatternStore {
  bias: PatternBias;
  lastUpdatedEpoch: number;
}
```

**Coupling rule:** Learning writes `PatternBias` to the `PatternStore`. Selection reads `PatternBias` from the `PatternStore` during `sample()`. No other cross-system data flow exists.

### Public API (Barrel Export)

```typescript
// Types
export type { AgentSeed, WinnerRecord, AgentOutputRecord, CandidatePool,
  PatternBias, SignalEvent, CapabilityProfile, PerformanceEnvelope,
  BenchmarkResult, RotationResult, PatternStore } from './types';

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
export { signalStrength, taskTypeToIndex, regionSamplingBias } from './signal-field';

// Layer 7 — Prompt Generation
export { generateSystemPrompt, assertPromptDeterminism } from './prompt-generator';

// Layer 8 — Selection Protocol
export { sample, benchmark, rotate } from './selection-protocol';

// Layer 9 — Audit
export { auditSourceForAntiPatterns, runDeterminismSuite } from './audit';
```

### Test Suite: 9-Layer Determinism Verification

Standard test seeds for all layers:
```typescript
const TEST_SEEDS: AgentSeed[] = [
  { roleId: 0,     styleId: 0,     capabilitySalt: 0,     worldSeed: 1n },
  { roleId: 32767, styleId: 32767, capabilitySalt: 32767, worldSeed: 1n },
  { roleId: 65535, styleId: 65535, capabilitySalt: 65535, worldSeed: 1n },
  { roleId: 1000,  styleId: 2000,  capabilitySalt: 3000,  worldSeed: 9999999999n },
];
```

**Layer test coverage:**

| Layer | Module | Key Assertions |
|-------|--------|---------------|
| 1 | hash.ts | FNV-1a consistency, uniqueness, distribution, child derivation |
| 2 | capability-field.ts | Determinism, value ranges [0,1], adjacent coherence |
| 3 | affinity.ts | Symmetry, value range [0,1], ranking order |
| 4 | epoch-tracking.ts | Determinism, historical query stability, valid ranges |
| 5 | evolution-chain.ts | Bounded deltas, round-trip reconstruction, branch divergence |
| 6 | signal-field.ts | Determinism, value range [0,1], task type hashing |
| 7 | prompt-generator.ts | 10-trial determinism, cross-seed uniqueness, structural elements |
| 8 | selection-protocol.ts | Sample determinism, pool cap, no duplicates, rotation quartile drop |
| 9 | audit.ts | Anti-pattern detection, clean code pass-through |

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
| ROLE_RADIUS | 512 | signal-field.ts | Region influence distance |
| Task type buckets | 256 | signal-field.ts | Hash modulo for task categorisation |
| Short-term decay | 1800s | stigmergic-board.ts | 30-minute fast decay |
| Long-term decay | 7200s | stigmergic-board.ts | 2-hour slow decay |
| Amplification factor | 1.5 | stigmergic-board.ts | Signal boost on success |
| Attenuation factor | 0.7 | stigmergic-board.ts | Signal reduction on failure |
| Winner score bonus | 0.15 | zero-specialist-layer.ts | Proven performer advantage |
| Pattern blend ratio | 70/30 | hybrid-orchestrator.ts | Pattern score vs signal weight |

### Reconstruction Checklist

To rebuild ZeroSpecialist from scratch in any TypeScript/JavaScript environment:

1. **Implement `hash.ts`** — FNV-1a with exact constants. Verify: `fnv64(new Uint8Array([1,2,3,4]))` produces a stable bigint.
2. **Implement `types.ts`** — All interfaces exactly as specified. AgentSeed must use bigint for worldSeed.
3. **Implement `capability-field.ts`** — gridHash, coherentNoise (4 octaves), capabilityField. Verify: `capabilityField(1000, 2000, 1n)` returns identical profile across runs.
4. **Implement `prompt-generator.ts`** — The three lookup tables (10+10+8 entries), thresholding logic, template string. Verify: `generateSystemPrompt({roleId:0, styleId:0, capabilitySalt:0, worldSeed:1n})` produces identical text across runs.
5. **Implement `affinity.ts`** — Sorted pair hashing with polarity channel. Verify: symmetry property.
6. **Implement `epoch-tracking.ts`** — Temporal hash with performance envelope derivation. Verify: O(1) access to any epoch.
7. **Implement `evolution-chain.ts`** — Delta derivation, winner packing/unpacking, reconstruction. Verify: round-trip at depth 0 preserves prompt.
8. **Implement `signal-field.ts`** — Signal strength computation, task type indexing. Verify: deterministic output.
9. **Implement `selection-protocol.ts`** — sample, benchmark, rotate with exact constants. Verify: sample determinism and pool cap.
10. **Implement `boundary.ts`** — Type-only module. No logic.
11. **Implement `audit.ts`** — Anti-pattern scanner and runtime determinism suite.
12. **Wire integration layer** — ZeroSpecialistLayer (scoring + selection), StigmergicBoard (two-tier decay), HybridSwarmOrchestrator (three-layer coordination).
13. **Add state persistence** — Zustand store with BigInt serialization adapter.
14. **Run test suite** — All 9 layers must pass. Cross-seed uniqueness >= 80%.

### Why 128 Bits Replaces 3000 Tokens

| Dimension | Traditional Prompt | ZeroSpecialist |
|-----------|--------------------|----------------|
| Storage per agent | ~4,000 bytes | 16 bytes (128 bits) |
| Reproducibility | Session-dependent | Deterministic forever |
| Variation space | Manually enumerated | 2^48 base + 2^160 profile |
| Evolution tracking | Lost on edit | Full lineage from 128-bit ref |
| Cross-machine identity | Impossible | Guaranteed by pure functions |
| Prompt bloat over time | Grows linearly | Constant 16 bytes per winner |

The 128-bit seed encodes position in a continuous behavioral space. The prompt is not compressed into 128 bits — it is *generated* from 128 bits through a deterministic pipeline of hashing, coherent noise, and template assembly. The information is not in the bits themselves but in the shared algorithm that interprets them.
