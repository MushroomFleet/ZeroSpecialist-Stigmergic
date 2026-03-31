# ZeroSpecialist Stigmergic

A Tauri desktop application implementing **three-layer swarm intelligence** with deterministic seed-hash agents, dynamic approaches, and stigmergic coordination. ZeroSpecialist replaces mutable agent configurations with a pure coordinate-hash architecture — every specialist is a deterministic function of a 4-integer seed tuple, enabling an addressable space of **4.2 billion unique agents** while storing each identity as a 128-bit seed reference instead of a 4,000-token system prompt.

- LIVE WEB: [https://scuffedepoch.com/zerospecialist-swarm/](https://scuffedepoch.com/zerospecialist-swarm/)
- LIVE EDU: [https://scuffedepoch.com/zerospecialist-web/](https://scuffedepoch.com/zerospecialist-web/)
- PIPELINE-AS-NARRATIVE: [ZeroSpecialist-Stigmergic-Journey.md](https://github.com/MushroomFleet/ZeroSpecialist-Stigmergic/blob/main/ZeroSpecialist-Stigmergic-Journey.md)

## The ZeroSpecialist Advantage

Traditional AI orchestration systems store full system prompts for each specialist — typically 2,000–4,000 tokens per agent. ZeroSpecialist eliminates this entirely:

| Metric | Hardcoded Prompts | ZeroSpecialist |
|---|---|---|
| Storage per agent | ~4,000 tokens | 128 bits (4 integers) |
| Addressable agents | Dozens (manually authored) | **65,536² = 4,294,967,296** |
| Agent reconstruction | Load from database | O(1) pure function |
| Reproducibility | Mutable, drift-prone | Byte-identical forever |
| Token overhead per request | Full prompt in context | Generated on demand |

By generating system prompts deterministically at inference time rather than storing and loading them, ZeroSpecialist reduces per-request token overhead and eliminates prompt storage entirely. The result: **near-instant responses** even with web search grounding enabled. Tested with Claude Sonnet 4.6 via OpenRouter, responses arrive in under a second with `:online` search variants — the swarm coordination adds negligible latency because agent selection is pure arithmetic, not database lookup.

**Web search grounding is recommended** for production use — it keeps specialist responses factually anchored without meaningful speed penalty.

## How It Works

### Three-Layer Coordination

The system coordinates without centralized control through three independent layers:

**Layer 1: ZeroSpecialist Deterministic Identity**
Every specialist is a pure function of a seed tuple `(roleId, styleId, capabilitySalt, worldSeed)`. The `worldSeed` is generated once per installation — the only randomness ever allowed. Each seed maps to a 5-dimensional behavioral profile (verbosity, assertiveness, formality, domain focus, reasoning depth) via coherent noise fields. Neighboring seeds produce measurably similar agents; distant seeds produce radically different ones. The swarm samples 32 candidates per epoch, benchmarks them against tasks, and locks winners by seed reference.

**Layer 2: Dynamic Approach Patterns**
While ZeroSpecialist selects *who* handles a task, this layer determines *how*. Approaches are response strategies discovered through clustering analysis of past executions. They capture style, structure, and strategy — and evolve through multi-generational refinement. After 10+ diverse conversations, pattern discovery creates approach objects with signature characteristics that are matched to future tasks.

**Layer 3: Stigmergic Coordination**
Named after how ants coordinate through pheromone trails. When a specialist-approach pair performs well, it deposits a signal that influences future selections. Signals decay on two tiers: short-term (30 minutes) for tactical responsiveness, long-term (2 hours) for background patterns. This creates emergent coordination without direct agent communication.

### System Lifecycle

1. **Phase 0 — ZeroSpecialist Initialization:** On first launch, a worldSeed is generated and 32 candidate specialists are sampled from the infinite procedural space. All agent identities are deterministic from their seed coordinates.
2. **Phase 1 — Bootstrap:** The system uses fallback approaches while accumulating execution history. A progress indicator shows execution count toward the pattern discovery threshold (10+).
3. **Phase 2 — Pattern Discovery:** From the Dashboard, trigger clustering analysis to create approach patterns from conversation history.
4. **Phase 3 — Emergent Coordination:** All three layers active. ZeroSpecialist selects agents by seed scoring, approaches match by pattern similarity, stigmergic signals reinforce successful combinations.

## Quick Start

### Option 1: Desktop Application (Recommended) Openrouter API

Download the latest MSI installer from [Releases](https://github.com/MushroomFleet/ZeroSpecialist-Stigmergic/releases).

1. Install and launch **ZeroSpecialist Stigmergic**
2. Navigate to **Settings** and add your **OpenRouter.ai API key**
3. Select a model — **Claude Sonnet 4.6** recommended, with `:online` variant for web search grounding
4. Start chatting — the swarm coordinates automatically

### Option 2: Desktop Application (UNTESTED) Ollama

Run completely offline with local LLMs — no API keys or internet required:

1. Install [Ollama](https://ollama.ai) and pull a model (e.g. `ollama pull gemma3-1b-Q8_0`)
2. In Settings, enable **Local Mode** and set the Ollama endpoint
3. Select your model from the dropdown

Local mode supports single-specialist and dual-execution modes, optimized for consumer hardware (4GB+ VRAM).

## Dashboard & Visualization

- **Signal Network Graph** — real-time stigmergic signal visualization with filtering by strength, type, and time range
- **Performance Analytics** — multi-tab system covering specialist quality, approach trends, and signal distribution
- **Specialist Carousel** — deep-dive into each agent's seed identity, capability profile, execution history, and deterministic system prompt
- **ZeroSpecialist Identity Tab** — view seed coordinates, 5-dimension behavioral profile, performance envelope, locked winner status, and agent affinities
- **Pattern Discovery Controls** — configurable clustering thresholds for approach generation

## Features

- **Deterministic Agents** — 4.2 billion addressable specialists from a 65,536² seed space, each reconstructable in O(1)
- **Three-Layer Coordination** — ZeroSpecialist selection, dynamic approaches, stigmergic signals
- **Parallel Execution** — 2–5 specialists execute simultaneously with quality voting
- **Near-Instant Responses** — minimal token overhead enables sub-second latency with capable models
- **Web Search Grounding** — recommended for factual anchoring via OpenRouter `:online` variants
- **Transparent Decisions** — every coordination choice visible via swarm traces in chat
- **Tauri Desktop App** — native Windows application with MSI/NSIS installers
- **Ollama Support** — complete offline operation with local LLMs
- **Pattern Discovery** — emergent approach patterns from execution history clustering
- **Epoch Rotation** — candidate pool refreshes with pattern-biased resampling
- **Winner Locking** — high-performing seeds promoted and preserved across epochs

## Architecture

### Core Components
- **ZeroSpecialist Layer** — deterministic seed-hash agent generation and scoring (`src/zero-specialist/`)
- **Hybrid Orchestrator** — three-layer coordination pipeline (`src/core/hybrid-orchestrator.ts`)
- **Dynamic Approach Manager** — pattern matching and evolution (`src/core/dynamic-approaches.ts`)
- **Stigmergic Board** — two-tier signal decay and coordination (`src/core/stigmergic-board.ts`)
- **Parallel Executor** — concurrent specialist execution with quality voting
- **Content Analyzer** — task-aligned response quality assessment

### Technology Stack
- **Desktop:** Tauri 2 (Rust backend, WebView frontend)
- **Frontend:** React 18, TypeScript, Vite
- **UI:** Tailwind CSS, shadcn/ui, Recharts
- **State:** Zustand (in-memory), Dexie/IndexedDB (persistent)
- **Data:** React Query for async data management
- **LLM Access:** OpenRouter.ai, Ollama (local)

## Related Projects

- [Cognition-9](https://github.com/MushroomFleet/Cognition-9) — Sub Agent Topology Study, experimental origins
- [Hybrid Swarm Agent](https://github.com/MushroomFleet/Hybrid-Swarm-Agent) — Claude Code agent implementation
- [Zerobytes Family Skills](https://github.com/MushroomFleet/ZeroBytes-Family-Skills) - Determinism Skill collection

## Citation

```bibtex
@software{zerospecialist_stigmergic,
  title = {ZeroSpecialist Stigmergic: Deterministic AI Swarm Orchestration},
  author = {Drift Johnson},
  year = {2025},
  url = {https://github.com/MushroomFleet/ZeroSpecialist-Stigmergic},
  version = {0.6.4}
}
```

## License

Copyright 2025 Drift Johnson. All rights reserved.

---

Powered by [ScuffedEpoch](https://scuffedepoch.com) and [OragenAI](https://oragenai.com)
