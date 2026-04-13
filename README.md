# Qomni Engine v8: From Crystal Distillation to Adaptive Cognitive OS

**Percy Rojas Masgo** · CEO, Condesi Perú · **Qomni AI** · Preprint April 2026

---

> **Qomni is not a crystal system.**
> Crystals are one of seven tools Qomni chooses from.
> Qomni is a cognitive orchestrator — it picks the best strategy for each query,
> from zero-compute reflexes to full neural inference, and learns which works best.

---

## What is Qomni?

**Qomni** is an Adaptive Cognitive OS for AI inference. It does not depend on
any single strategy. For each incoming query, Qomni evaluates multiple paths
and selects the minimum-cost option that achieves sufficient quality:

```
Query
  │
  ├─ [Reflex]      Zero-compute pattern → <5ms     (no AI)
  ├─ [CRYS-L JIT]  Physics program      → 50-200ms (deterministic)
  ├─ [Oracle]      Formula retrieval    → 100-400ms (indexed)
  ├─ [Crystal]     Knowledge retrieval  → 100-400ms (compressed NN)
  ├─ [Pipeline]    Oracle + Crystal     → 100-500ms (parallel, no LLM)
  ├─ [CTX Inject]  Crystal → LLM        → 500ms+   (augmented LLM)
  └─ [Free LLM]    Full neural          → 8-45s    (last resort)
```

**None of these layers are mandatory.** Each query independently determines
which layers activate, based on a continuous confidence score. The system learns
which strategies work best for each domain and adjusts thresholds automatically.

---

## The Seven Strategies

| # | Strategy | What it uses | LLM | Latency | vs LLM |
|---|----------|-------------|-----|---------|--------|
| 1 | **Reflex** | Cached patterns + HDC Memory | Never | <5ms | **9,000×** faster |
| 2 | **CRYS-L JIT** | Engineering programs (WASM) | Never | 50-200ms | **225×** faster |
| 3 | **Oracle Direct** | Formula retrieval from Crystal KB | Never | 100-400ms | **112×** faster |
| 4 | **Crystal Direct** | Knowledge retrieval from Crystal KB | Never | 100-400ms | **112×** faster |
| 5 | **Pipeline** | Oracle + Crystal parallel | Never | 100-500ms | **90×** faster |
| 6 | **Crystal-Ctx** | Crystal knowledge injected into LLM | Yes | 500ms+ | 2-10× faster |
| 7 | **Free LLM** | Full neural generation | Yes | 8-45s | baseline |

**Live benchmark: 100% of domain-specific test queries resolved by strategies 1-5 (no LLM).**

---

## Speed: Qomni + CRYS-L Combination

The key to Qomni's speed is the **cascade exit**: each layer returns a result immediately
if it has sufficient confidence. CRYS-L JIT is the fastest *intelligent* layer — it compiles
a calculation plan to WASM once, then executes it at near-native speed in microseconds.

### Why CRYS-L JIT is Uniquely Fast

```
Traditional flow (LLM):
  User: "cuantos hp necesita una bomba de 750 gpm a 100 psi"
  LLM: tokenize → 750M params → decode → 45,000ms → "approximately 18 HP"

Qomni + CRYS-L JIT:
  User: "cuantos hp necesita una bomba de 750 gpm a 100 psi"
  1. Parse: match "hp", "gpm", "psi", "bomba" → plan_pump_sizing (2ms)
  2. Execute WASM: HP = (750×0.063 × 100×0.703) / (0.7×76) (0.1ms)
  3. Format result: "18.04 HP — NFPA 20:2022 §4.26" (0.1ms)
  Total: ~148ms (includes HTTP, JSON parsing, network)
  WASM execution: ~0.2ms
```

### Qomni Planner + CRYS-L: Parallel Execution

When a query has both engineering content AND domain knowledge intent,
Qomni runs **CRYS-L + Crystal retrieval concurrently**:

```
tokio::join!(
    crysl_execute(plan, params),      // deterministic computation
    crystal_retrieve(domain, query),   // knowledge base lookup
)
→ merge results → return combined answer (no LLM)
```

This is the **PIPELINE-DIRECT** route: oracle calculation + crystal context,
assembled and returned in ~300-500ms — 90× faster than LLM baseline.

### Speed Comparison (measured, 2026-04-13)

```
Query: "calcular hp bomba contra incendio nfpa20 750 gpm 100 psi"

Strategy          | Time    | What happened
──────────────────|─────────|──────────────────────────────────────
Reflex (miss)     | <2ms    | no cached pattern → pass to next layer
CRYS-L JIT (hit)  | 148ms   | plan_pump_sizing compiled+executed
                  |         | → "18.04 HP, shutoff 21.6 HP (NFPA 20)"
Full LLM          | ~12,000ms | neural generation, approximate answer

Speedup: 81× faster than LLM, exact answer from published standard
```

```
Query: "como constituir una empresa en sunat peru"

Strategy          | Time    | What happened
──────────────────|─────────|──────────────────────────────────────
Reflex (miss)     | <2ms    | no reflex pattern
CRYS-L (miss)     | <2ms    | not an engineering calculation
Planner:          |         |
  domain_score=3  | 5ms     | legal_peru detected
  oracle retrieve | 180ms   | legal formula retrieval
  crystal retrieve| 180ms   | parallel legal KB lookup
  → oracle-direct | 378ms   | crystal score=8 → direct response
Full LLM          | ~18,000ms| neural generation

Speedup: 47× faster than LLM
```

---

## Version History

| Version | Key Milestone |
|---------|--------------|
| v1 | Crystal Distillation — Physics-as-Oracle → `.crystal` binary |
| v2 | CRYS-L Compiler — open DSL for engineering calculations |
| v3 | Modular Rust Architecture — 12 modules, 81% code reduction |
| v4 | Cognitive Features — ReflexEngine, LiquidMemory, SleepPipeline |
| v5 | Neural Gating, HDC Memory, Solid State Inference |
| v6 | Lock-Free Architecture — DashMap, AtomicU64, 10/10 security |
| v7 | TUI Dashboard, Mesh Auth, 45 unit tests |
| **v8** | **Adaptive Planner — Learning Loop, Phase 0.59 live** |

---

## Qomni Planner — Evolution

```
Phase 0.55 → Binary: oracle≥3 AND domain≥2 → PIPELINE else FREE
Phase 0.56 → Multi-Factor: domain + oracle + numeric scores
Phase 0.57 → Direct: crystal score≥3 → bypass LLM entirely
Phase 0.58 → Continuous confidence: soft scoring + self-healing fallback
Phase 0.59 → Learning Loop: AtomicU64 EMA, adaptive thresholds, persistence
Phase 0.60 → Per-Domain: EMA per domain + decay + calibration engine (designed)
```

### Confidence Formula

```
pre_conf = oracle_c + domain_c + numeric_c
conf     = pre_conf + crystal_bonus(actual retrieval score)

oracle_c  : 0→0.00, 1→0.10, 2→0.22, 3→0.35, 4+→0.42
domain_c  : 0→0.00, 1→0.13, 2→0.28, 3→0.36, 4+→0.42
numeric_c : (score/4).min(1.0) × 0.16
crystal_b : score 0→0.00, 1→0.04, 2→0.07, 3+→0.10

conf ≥ 0.70 → PIPELINE-DIRECT  (no LLM)
conf ≥ 0.50 → ORACLE/CRYSTAL-DIRECT (no LLM)
conf ≥ 0.28 → CRYSTAL-CTX (inject → LLM)
conf < 0.28 → FREE LLM
```

### Learning Loop (Phase 0.59 → 0.60)

```
Every query:
  decide(conf) → execute(route) → evaluate(validate_result) → record(AtomicU64) → adapt(thresholds)

Phase 0.59: global success rates → global threshold adaptation
Phase 0.60: per-domain EMA decay → per-domain thresholds → Wilson CI calibration
```

---

## CRYS-L v2 — Open Language (MIT License)

CRYS-L is the open language for expressing engineering calculations as
deterministic programs. The compiler and WASM runtime are **MIT-licensed**
and available for community extension.

```crysl
plan_pump_sizing(Q_gpm: f64, P_psi: f64, eff: f64 = 0.7) {
    meta { standard: "NFPA 20:2022", domain: "nfpa_electrico" }

    let HP_req = (Q_gpm * 0.06309 * P_psi * 0.70307) / (eff * 76.04);

    assert Q_gpm > 0.0 msg "flow must be positive";
    assert eff   <= 1.0 msg "efficiency must be <= 1.0";

    output HP_req label "Fire pump HP required" unit "HP";
}
```

**Supported domains (v2):** hidraulica · nfpa\_electrico · civil · mecanica · termica · sanitaria

**How to contribute a new plan:**
1. Write your `.crysl` plan following the grammar in [`architecture/crysl_v2_language_spec.md`](architecture/crysl_v2_language_spec.md)
2. See examples in [`architecture/crysl_v2_examples.crysl`](architecture/crysl_v2_examples.crysl)
3. Submit a Pull Request — all valid plans following the grammar are welcome

---

## Live Benchmarks (2026-04-13, Server5)

### Routing Latency (2 runs, 12 queries total)

| Query | Route | ms |
|-------|-------|-----|
| bomba nfpa20 750 gpm HP | crysl-jit | 148 |
| constituir empresa sunat | oracle-direct | 378 |
| hazen williams 4'' 200m | crysl-jit | 130 |
| transformador 50 kva | crysl-jit | 114 |
| hola buenos dias | reflex | 51 |
| factura igv planilla cts | oracle-direct | 335 |
| **Average (run 2)** | | **193ms** |

**LLM bypass rate: 100% · Speedup over LLM: 41–233×**

### Crystal Retrieval (all 6 domains)

| Domain | Latency | Max Score |
|--------|---------|-----------|
| legal_peru | 40ms | 7 |
| contabilidad | 58ms | 8 |
| hidraulica | 72ms | 11 |
| nfpa_electrico | 27ms | 4 |
| tecnico_it | 38ms | 7 |
| whatsapp_ventas | 39ms | 10 |
| **Average** | **45.7ms** | |

---

## Phase 0.60 Preview

```
Per-domain EMA decay (α=0.10, half-life ≈ 7 queries):
  EMA[domain] = 0.10 × outcome + 0.90 × EMA[domain]

Adaptive threshold per domain:
  θ_direct(d) = clamp(0.50 − (ema_oracle[d] − 0.5)×0.20, 0.40, 0.60)
  θ_ctx(d)    = clamp(0.28 + (0.5 − ema_crystal[d])×0.20, 0.20, 0.38)

Calibration engine (background Tokio task, every 30s):
  Wilson score CI < 0.15 → apply adjustment, log event

Real-time dashboard:
  TUI tab "Planner" + GET /qomni/planner/dashboard (HTML + Chart.js)
```

Full spec: [`architecture/phase_0_60_design.md`](architecture/phase_0_60_design.md)

---

## Repository Structure

```
qomni-crystal-paper-v8/
├── arxiv/
│   └── main.tex                    # Full LaTeX paper (arXiv-ready)
├── architecture/
│   ├── cascade_diagram.md          # 5-layer cascade architecture
│   ├── phase_0_60_design.md        # Phase 0.60 full technical spec
│   ├── crysl_v2_language_spec.md   # CRYS-L v2 grammar + stdlib docs
│   └── crysl_v2_examples.crysl    # Annotated plan examples (open, MIT)
├── results/
│   ├── benchmark_v8.json           # Live latency benchmarks (2 runs)
│   ├── crystal_benchmark.json      # Crystal retrieval by domain
│   └── planner_evolution.json      # Phase 0.55→0.59 data
├── datasets/
│   ├── test_queries.jsonl          # 12 annotated queries + expected routes
│   └── routing_decisions.jsonl     # Observed live routing decisions
└── README.md
```

> **Source code**: Qomni Engine is proprietary.
> CRYS-L language, compiler, and runtime are MIT-licensed.
> All benchmark data, architecture specs, and datasets in this repo are open.

---

## Citation

```bibtex
@article{rojasmasgo2026qomni-v8,
  title   = {Qomni Engine v8: From Crystal Distillation to Adaptive Cognitive OS},
  author  = {Rojas Masgo, Percy and {Qomni AI}},
  year    = {2026},
  month   = {April},
  note    = {Preprint},
  url     = {https://github.com/condesi/qomni-crystal-paper-v8}
}
```
