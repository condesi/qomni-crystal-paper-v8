# Qomni Engine v8 — 5-Layer Cognitive Cascade

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    QOMNI ENGINE v8 (207 features)               │
│                                                                 │
│  Query ──────────────────────────────────────────────────────▶  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ LAYER 0: Reflex Engine (v4.6)                           │   │
│  │ • Zero-compute pattern matching                         │   │
│  │ • HDC Memory lookup (v5.7, 10,000-dim bit vectors)      │   │
│  │ • Latency: <5ms  │  LLM: BYPASSED                      │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │ miss                              │
│  ┌─────────────────────────▼──────────────────────────────┐   │
│  │ LAYER 0.5: CRYS-L JIT (v2.1)                           │   │
│  │ • Engineering DSL: plan declarations → WASM execution  │   │
│  │ • Domains: fire protection, electrical, hydraulics...  │   │
│  │ • Latency: 50-200ms  │  LLM: BYPASSED                 │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │ not an engineering calc           │
│  ┌─────────────────────────▼──────────────────────────────┐   │
│  │ LAYER 0.59: Qomni Planner (Cognitive Decision Engine)  │   │
│  │                                                         │   │
│  │  confidence = oracle_c + domain_c + numeric_c           │   │
│  │            + crystal_bonus(actual retrieval)            │   │
│  │                                                         │   │
│  │  ┌─────────────────────────────────────────────────┐   │   │
│  │  │ tokio::join!(oracle_generate, crystal_retrieve) │   │   │
│  │  └─────────────────────────────────────────────────┘   │   │
│  │                                                         │   │
│  │  conf≥0.70 → PIPELINE-DIRECT ─────────────────────▶ ✓ │   │
│  │  conf≥0.50 → ORACLE/CRYSTAL-DIRECT ───────────────▶ ✓ │   │
│  │  conf≥0.28 → CRYSTAL-CTX ─────────────────────────▶ ↓ │   │
│  │  conf<0.28 → FREE ─────────────────────────────────▶ ↓ │   │
│  │                                                         │   │
│  │  Learning Loop: AtomicU64 EMA per route+domain          │   │
│  │  Persistence: /opt/nexus/planner_stats.json             │   │
│  │  Latency: 100-500ms  │  LLM: CONDITIONAL               │   │
│  └──────────────────────────┬──────────────────────────────┘   │
│                             │ conf<0.28 or ctx inject           │
│  ┌─────────────────────────▼──────────────────────────────┐   │
│  │ LAYER 1: LLM Inference                                  │   │
│  │ • Model: qwen2.5-1.5b-instruct-q4_k_m (CPU)            │   │
│  │ • NeuralGating (v5.3): dynamic layer skipping           │   │
│  │ • SolidStateInference (v5.8): response caching          │   │
│  │ • Latency: 8,000-45,000ms                               │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │ POST-INFERENCE: Cognitive Loop (async)                  │   │
│  │ • LiquidMemory (v4.3): raw→concept→pattern→intuition    │   │
│  │ • SleepPipeline (v4.7): idle-time consolidation         │   │
│  │ • SemanticClusterer (v4.8): concept graph evolution     │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

## Latency Budget

| Layer | Condition | Latency | LLM Cost |
|-------|-----------|---------|----------|
| Reflex | Pattern match | <5ms | None |
| CRYS-L JIT | Engineering query | 50–200ms | None |
| Planner Direct | conf ≥ 0.50 | 100–500ms | None |
| Planner + LLM | conf < 0.50 | 8,000–45,000ms | Full |

## Feature Count by Module (v8)

| Module | Features | Status |
|--------|----------|--------|
| v4.1 Energy Metrics | 1 | Active |
| v4.2 Speculative Drafter | 1 | Stub |
| v4.3 Liquid Memory | 1 | Active |
| v4.4 Mesh Admin | 1 | Active |
| v4.5 Self-Coding Kernel | 1 | Active (sandbox) |
| v4.6 Reflex Engine | 1 | Active |
| v4.7 Sleep Pipeline | 1 | Active |
| v4.8 Semantic Clusterer | 1 | Active |
| v5.3 Neural Gating | 1 | Active |
| v5.7 HDC Memory | 1 | Active |
| v5.8 Solid State Inference | 1 | Active |
| Planner (v0.55-0.59) | 5 phases | Active |
| Crystal Engine | 8 domains | Active |
| CRYS-L JIT | 15+ plan types | Active |
| Security (C1-C4, H1-H5, M3) | 10 fixes | Complete |
| **Total** | **207** | |
