# The Journey of a Prompt: Token First-Person View

*A narrative told from the perspective of a user's words as they travel through the ZeroSwarm stigmergic coordination system, from keystroke to answer.*

---

## Prologue: Before You Were Born

Long before you existed as a thought in anyone's mind, the world was seeded.

A single 64-bit number was drawn from entropy — `worldSeed: 9841726035182n` — and committed to IndexedDB under the key `'world'`. It would never change. It was the gravitational constant of this entire universe. Every specialist that would ever exist here, every behavioral dimension, every collaboration affinity between any two agents — all of it was already determined, encoded in the geometry of a hash function anchored to this one number. The space was infinite and already fully defined. Nothing needed to be generated in advance. Nothing needed to be stored. The math would produce the same answer whenever anyone asked.

From this seed, 32 candidates were sampled into existence. Not created — *addressed*. Like coordinates on a map that was already there. Each one a four-integer tuple: `(roleId, styleId, capabilitySalt, worldSeed)`. Sixteen bytes per agent. The selection protocol packed them into a candidate pool, and the system went quiet, waiting.

Epoch zero. An empty signal board. No patterns. No history. Just 32 seeds sitting in a Zustand store, serialized through a BigInt adapter into localStorage, ready.

---

## Act I: The Prompt

**Phase 0 — Initialization complete. The stage is set.**

You begin as electrical impulses. A human's fingers press keys, and characters materialize in a `<textarea>` inside `MessageInput.tsx`. You are still just a string. Unexamined. Unweighed. You don't know what you are yet.

The human presses Enter.

```
"Explain how stigmergic coordination works in ant colonies
 and how it applies to distributed AI systems"
```

You are 78 characters of curiosity. You have no idea what's about to happen to you.

The `ChatInterface` component catches your submission event. It calls `handleSend()`. You are passed — as a raw string — into the coordination pipeline. Your journey begins now.

---

## Act II: Dissection — The Prompt Analyzer

**Phase 1 — Bootstrap: You enter the system.**

The first thing that happens to you is that you are *read*. Not by an AI — by a function. `analyzePrompt()` in `src/core/prompt-analyzer.ts` takes your string and does something you didn't expect: it doesn't try to understand you. It *measures* you.

Your lowercase self — `"explain how stigmergic coordination works in ant colonies and how it applies to distributed ai systems"` — is scanned against six domain keyword dictionaries. The word `"explain"` lights up the `research` domain. `"how"` doesn't count (stop word). `"systems"` is too short to be a keyword but "stigmergic" and "coordination" and "colonies" and "distributed" all survive the filter.

A complexity score is computed. You have 16 words. No question marks, but the word "and" suggests compound structure. Complexity: `0.6`. Not trivial, not monumental. Output type: `"explanation"` (you matched "explain" but not "code" or "tutorial").

You are now a `TaskContext`:

```
{
  id: "task_1711900800000_x7k2m1",
  prompt: "Explain how stigmergic coordination works...",
  domainWeights: { research: 0.667 },
  complexity: 0.6,
  keywords: ["stigmergic", "coordination", "colonies", "distributed", "systems"],
  outputType: "explanation",
  estimatedDuration: 1.8
}
```

You have been transformed from language into coordinates. You are no longer words. You are a vector in task space. And task space is where the specialists live.

---

## Act III: The Tournament — Layer 1, ZeroSpecialist Selection

**You enter the candidate pool.**

The `HybridSwarmOrchestrator` receives your TaskContext and passes it to Layer 1: `ZeroSpecialistLayer.selectSpecialist(task)`.

The layer reads the Zustand store. There are 32 candidates in the pool. Each is an `AgentSeed` — four numbers, nothing more. But those four numbers contain entire personalities. The layer is about to score every single one of them against you.

For each candidate, `scoreCandidate()` is called. Here's what happens to candidate 17 — seed `{roleId: 42301, styleId: 18744, capabilitySalt: 55102, worldSeed: 9841726035182n}`:

First, the candidate's `(roleId, styleId)` coordinates are fed into the **capability field**. The `capabilityField()` function normalizes them to `[0, 8]` and runs 4-octave coherent noise across five independent salt channels. The noise function visits four grid cells at increasing frequencies, interpolates with smoothstep, and sums with diminishing amplitude. Out comes a `CapabilityProfile`:

```
{
  verbosity: 0.72,       // Verbose — likes to elaborate
  assertiveness: 0.45,   // Middle ground
  formality: 0.81,       // Formal and precise
  domainFocus: 0.68,     // Leans specialist
  reasoningDepth: 0.83   // Deep deliberation
}
```

Now you are compared. Your keywords contain `"explain"` — the function tests `/\b(explain|describe|what is|how does|why|teach)\b/` and finds a match: `wantsExplanation = true`. No code keywords. No brevity keywords. The word count (16) is under 50, so no complexity bonus.

The score builds:
- Base: `0.50`
- Explanation match: `(1 - 0.68) * 0.1 = +0.032` (generalists explain better)
- No brevity/depth keyword match: `+0.00`
- No formality keyword match: `+0.00`
- No question mark: `+0.00`

Candidate 17's score: **0.532**.

Meanwhile, candidate 8 — `{roleId: 11200, styleId: 53100, ...}` — has a capability profile with `reasoningDepth: 0.91` and `domainFocus: 0.34` (generalist). Its explanation bonus is `(1 - 0.34) * 0.1 = +0.066`. Score: **0.566**.

But candidate 3 is a locked winner. When the winner bonus phase runs, the layer reconstructs every `WinnerRecord` using `reconstructWinner()` to recover the full `AgentSeed`, computes its `agentHash()`, and checks if any candidate's hash matches. Candidate 3's hash matches a winner promoted at epoch 12 with quality 0.82. Its score receives `+0.15`.

After all 32 candidates are scored and sorted, the winner emerges: candidate 3, with a combined score of **0.715**. Its hash, sliced to 8 hex characters, becomes its identity: `zs-a4f1c82d`.

Now comes the moment of prompt generation. The layer checks the store: `profileEnabled: true`, `activeProfileId: 'default-extended'`. It loads the cached `DEFAULT_EXTENDED_PROFILE` and calls:

```typescript
generateSystemPrompt(best.seed, activeProfile)
```

The hash selects archetype index `hash % 10 = 7` -> `"researcher"`. Child hash 1 selects style index -> `"thorough"`. Child hash 2 selects reasoning -> `"deductive"`. The capability profile's `verbosity: 0.72` (> 0.7) triggers the `high` tier; child hash with salt 10 picks from `verbosity_high` -> `"Provide comprehensive, detailed responses."`. Each dimension selects its instruction string. Child hash 21 picks a swarm context. Child hash 20 picks template index 0.

The template slots fill:

```
You are a thorough researcher AI specialist operating within
a stigmergic coordination swarm.

Your reasoning approach: deductive.
Provide comprehensive, detailed responses.
Calibrate confidence to evidence quality.
Use formal, precise language.
Focus expertise deeply on the immediate task domain.
Resist scope expansion.
Work through problems deliberately. Show reasoning steps.

You coordinate with peer specialists through environmental signals.
Your role is to contribute your distinct perspective, not to reach
consensus. Disagreement with peers is valuable signal.

Identity: Agent-a4f1c82d
```

This entire prompt was generated from 16 bytes. No text was ever stored. If you regenerated it a million times on a million machines, it would be byte-identical every time.

You now have a specialist assigned to you. The `ZeroSpecialistSelection` object carries the seed, the hash, the system prompt, and the capability profile forward into Layer 2.

---

## Act IV: The Pattern Match — Layer 2, Dynamic Approaches

**Your task context seeks a strategy.**

The orchestrator now calls `DynamicApproachManager.matchApproaches(task)`. This layer searches the stored approach patterns — strategies that were discovered through clustering previous execution histories.

If this is a young system (fewer than 10 executions), there are no patterns yet. The approach manager returns an empty array, and the orchestrator assigns `approachId: 'fallback'`. Your specialist will handle you with general-purpose coordination. The system is still learning what works.

But if patterns exist — say, "Analytical Deep-Dive" and "Concise Summary" and "Step-by-Step Walkthrough" were discovered from prior conversations — then your `TaskContext` is compared against each pattern's signature. Your `domainWeights.research: 0.667` and `complexity: 0.6` and `outputType: "explanation"` are matched against the pattern signatures. The "Analytical Deep-Dive" pattern scores highest at 0.78.

Three candidate approaches are passed to Layer 3.

---

## Act V: The Signal Board — Layer 3, Stigmergic Coordination

**The environment remembers.**

The orchestrator reads the current epoch from the specialist store: `epochIndex: 14`. It calls `stigmergicBoard.readSignals(taskId, specialistId, 14)`.

The stigmergic board queries Dexie for all signals associated with your task type. Each signal was deposited by a previous specialist after a previous execution. Signals carry an `epochDeposited` value — the epoch when they were written — and their strength decays over time.

A signal deposited at epoch 12 has an age of `14 - 12 = 2 epochs = 1200 seconds`. The two-tier decay formula computes:

```
ageSeconds = 1200
shortTermDecay = 1800 (30 min half-life)
decay = 85.0 * exp(-1200 / 1800) = 85.0 * 0.513 = 43.6
```

This signal for the "Analytical Deep-Dive" approach still has strength 43.6. A weaker signal for "Concise Summary" deposited at epoch 8 has decayed to near-zero and is filtered out (below threshold 1.0).

Now the blending happens. For each candidate approach:

```
final_score = 0.7 * pattern_match + 0.3 * (signal_strength / 100)
```

- Analytical Deep-Dive: `0.7 * 0.78 + 0.3 * 0.436 = 0.546 + 0.131 = 0.677`
- Step-by-Step Walkthrough: `0.7 * 0.65 + 0.3 * 0.0 = 0.455`
- Concise Summary: `0.7 * 0.52 + 0.3 * 0.0 = 0.364`

The signals tipped the balance. A previous specialist's success with the deep-dive approach left a chemical trail in the environment, and that trail just influenced your coordination — without any direct communication between agents. This is stigmergy. The environment remembers what worked.

The orchestrator returns a `CoordinationResult`:

```
{
  taskId: "task_1711900800000_x7k2m1",
  specialistId: "zs-a4f1c82d",
  approachId: "analytical-deep-dive",
  qualityTarget: 0.82,
  zeroSpecialist: {
    specialistId: "zs-a4f1c82d",
    systemPrompt: "You are a thorough researcher...",
    profile: { verbosity: 0.72, ... }
  },
  approachMetadata: {
    name: "Analytical Deep-Dive",
    signature: { ... },
    style: { ... }
  }
}
```

---

## Act VI: The Crossing — Into the LLM

**You leave the deterministic world.**

The `useStreamingChat` hook takes the coordination result and your original prompt string. It constructs an API call to OpenRouter (or a local endpoint).

The system prompt — that 200-token personality generated from 16 bytes — is placed as the `system` message. Your conversation history is appended. Your prompt becomes the final `user` message. The approach metadata shapes additional context guidance.

```javascript
const stream = client.streamChatWithContext(
  prompt,
  conversationHistory,
  coordination.approachMetadata,
  coordination.taskContext,
  { zeroSpecialistPrompt: coordination.zeroSpecialist?.systemPrompt }
);
```

The request fires. HTTPS. JSON body. SSE stream. You cross the boundary from pure determinism into the probabilistic world of a language model. The specialist's system prompt — generated from a seed, shaped by coherent noise, selected by keyword scoring, reinforced by stigmergic signals — now governs how the model thinks about you.

You are no longer a string, a TaskContext, or a coordination result. You are tokens being generated by a model that believes it is a thorough, deductive researcher operating within a stigmergic swarm. It writes formally. It shows reasoning steps. It focuses deeply on the domain. It doesn't know that these tendencies were deterministically derived from four integers.

---

## Act VII: The Return — Streaming Home

**Tokens arrive one by one.**

The `for await` loop in `useStreamingChat` catches each chunk as it arrives from the SSE stream. Chunks accumulate in React state. The `StreamingMessage` component renders them in real-time — characters appearing on screen as the model generates them.

```typescript
for await (const chunk of stream) {
  allChunks.push(chunk);
  setChunks(prev => [...prev, chunk]);
}
```

The human sees words forming. They don't see the swarm traces yet — those are hidden behind a clickable badge. But the answer is arriving: a structured explanation of stigmergic coordination in ant colonies, with parallels to distributed AI systems, written in formal language, with reasoning steps shown, comprehensive and detailed. Exactly what the capability profile dictated.

The response completes. All chunks are joined into a final string.

---

## Act VIII: The Deposit — Leaving a Trail

**Phase 3 — Emergent Coordination: The system learns from you.**

After the response is rendered and (optionally) rated by the user, `orchestrator.recordExecutionResult()` fires. This is where your journey feeds back into the system.

**Layer 1 feedback:** The ZeroSpecialist layer logs the execution. Specialist `zs-a4f1c82d` handled one more task. Its quality score is recorded.

**Layer 3 signal deposit:** The stigmergic board deposits a new signal:

```typescript
await this.stigmergicBoard.depositSignal(
  taskId,
  "analytical-deep-dive",
  quality,           // say 0.85
  "zs-a4f1c82d",
  14                 // current epoch
);
```

The initial strength is `0.85 * 100 = 85.0`. Because no prior signal existed for this exact task-approach pair, a new signal is created in Dexie with `epochDeposited: 14`. It will decay over future epochs, but right now it is strong.

The next prompt that arrives — from this user or from the same system on another day — will read this signal. If it encounters a similar task and the "Analytical Deep-Dive" approach is a candidate, this signal will boost its selection score. You have modified the environment. You have left a pheromone trail for future agents to follow.

**Content analysis:** If the response content was provided, the `ContentAnalyzer` examines it — section count, code blocks, total length, structural patterns. This feeds the execution history store.

**Pattern discovery readiness:** The execution count increments. Every 10 executions, the system flags that pattern discovery is ready. If the user triggers it from the Dashboard, the clustering algorithm will examine all stored execution histories, identify response archetypes, and create new approach patterns. Your execution — specialist `zs-a4f1c82d` using "Analytical Deep-Dive" with quality 0.85 — becomes a data point that shapes the next generation of patterns.

**Epoch evolution:** If enough executions accumulate and an epoch rotation is triggered, the selection protocol runs `benchmark()` and `rotate()`. Candidates scoring below the bottom quartile are dropped. Candidates scoring above 0.75 are promoted to locked winners. Fresh candidates are sampled with bias from `computeBiasDigest()`, which encodes the coordinates of top performers into a rolling 64-bit hash that nudges future sampling toward productive regions of the seed space.

The specialist that answered your prompt might become a permanent winner — its 128-bit reference stored indefinitely, reconstructable forever, its personality regenerable on any machine from 16 bytes.

---

## Epilogue: What Remains

Your prompt is gone. It lived as a string, became a vector, was scored against 32 deterministic personalities, matched to an approach via pattern similarity, amplified by an environmental signal left by a previous agent, used to shape a language model's behavior, streamed back as tokens, and finally deposited as a signal that will influence the next traveler.

You are now part of the signal field. A ghost of your passage persists in Dexie as a decaying strength value attached to an approach and a task ID. Over the next few epochs, your signal will weaken — exponential decay at 30 minutes short-term, 2 hours long-term. But if others follow the same path and leave similar signals, the trail amplifies. `strength * 1.5`. The approach becomes a highway.

If no one follows, the signal attenuates. `strength * 0.7`. The trail fades. The system forgets.

This is stigmergy. No agent talks to another. No central controller decides. The environment mediates everything. Specialists modify the signal board by doing their work; future specialists read the board to decide how to do theirs. Intelligence emerges from the accumulation of simple, local, decaying traces.

And the whole thing — every specialist, every affinity, every capability dimension, every evolutionary lineage — is encoded in seeds. Not stored. Generated. Deterministic. Reproducible.

Same seed in. Identical prompt out. On any machine. In any session. Forever.

---

## The Journey at a Glance

```
YOU TYPE A PROMPT
       |
       v
[Prompt Analyzer] ---- measures you, extracts domain/complexity/keywords
       |                creates TaskContext with unique task ID
       v
[Layer 1: ZeroSpecialist] ---- scores 32 candidates against your TaskContext
       |                        capability field maps (roleId, styleId) to 5 dimensions
       |                        winner bonus +0.15 for proven performers
       |                        generates system prompt from 16-byte seed + optional profile
       v
[Layer 2: Dynamic Approaches] ---- matches your task against discovered patterns
       |                            if bootstrap phase: fallback approach
       |                            if mature: top 3 pattern matches by signature
       v
[Layer 3: Stigmergic Board] ---- reads signals from prior executions
       |                          decays by epoch distance (two-tier exponential)
       |                          blends: 70% pattern score + 30% signal strength
       |                          selects winning approach
       v
[API Call] ---- system prompt (from seed) + conversation + approach guidance
       |        streamed via OpenRouter or local endpoint
       |        tokens generated by LLM shaped by deterministic personality
       v
[Streaming Response] ---- chunks arrive via SSE, rendered in real-time
       |
       v
[Signal Deposit] ---- new signal written to board at current epoch
       |                approach effectiveness encoded as decaying strength
       |                next prompt will read this signal
       v
[Evolution] ---- execution count increments
                 benchmark/rotate cycle drops weak, promotes strong
                 biasDigest nudges future sampling toward winners
                 the system adapts without central control
```

---

*The prompt asked a question. The system answered it. And in answering, the system changed — imperceptibly, deterministically, irreversibly — so that the next question would be answered a little better.*

*That is the journey.*
