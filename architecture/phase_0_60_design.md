# Phase 0.60: Per-Domain Learning + Decay + Calibration Engine

## Motivation

Phase 0.59 maintains **global** success rate counters across all domains.
This creates cross-domain contamination:
- High crystal success in `legal_peru` (score=8) lowers the threshold for `hidraulica`
- A temporary spike in oracle failures in one domain penalizes others
- No decay: a good run 1000 queries ago has the same weight as the last 5 queries

Phase 0.60 solves all three problems.

---

## 1. Per-Domain Stats (AtomicU64 Arrays)

```rust
const N_DOMAINS: usize = 6;
// Domain index: 0=legal_peru, 1=contabilidad, 2=whatsapp_ventas,
//               3=hidraulica, 4=tecnico_it, 5=nfpa_electrico

pub struct PlannerStatsV2 {
    // Per-domain counters (raw)
    pub oracle_success:  [AtomicU64; N_DOMAINS],
    pub oracle_fail:     [AtomicU64; N_DOMAINS],
    pub crystal_success: [AtomicU64; N_DOMAINS],
    pub crystal_fail:    [AtomicU64; N_DOMAINS],

    // Exponential Moving Average (stored as ×1000 fixed-point)
    // EMA_new = α × outcome + (1-α) × EMA_old  (α = 0.10)
    pub oracle_ema:      [AtomicU64; N_DOMAINS],   // 0..1000
    pub crystal_ema:     [AtomicU64; N_DOMAINS],   // 0..1000

    // Multi-path learning: which source was preferred when both available
    pub oracle_preferred:  [AtomicU64; N_DOMAINS],
    pub crystal_preferred: [AtomicU64; N_DOMAINS],

    // Global totals
    pub total:           AtomicU64,
    pub calibrations:    AtomicU64,
}

impl PlannerStatsV2 {
    /// Update EMA with a new outcome (1.0=success, 0.0=fail)
    /// Uses compare_exchange loop for lock-free update
    pub fn update_ema(ema: &AtomicU64, outcome: f64) {
        const ALPHA: f64 = 0.10;
        loop {
            let old_raw = ema.load(Ordering::Relaxed);
            let old_val = old_raw as f64 / 1000.0;
            let new_val = ALPHA * outcome + (1.0 - ALPHA) * old_val;
            let new_raw = (new_val * 1000.0) as u64;
            if ema.compare_exchange(old_raw, new_raw,
                Ordering::Relaxed, Ordering::Relaxed).is_ok() { break; }
        }
    }

    /// Adaptive direct threshold for a specific domain
    pub fn direct_thresh(&self, domain_idx: usize) -> f64 {
        let ema = self.oracle_ema[domain_idx].load(Relaxed) as f64 / 1000.0;
        let raw_ok = self.oracle_success[domain_idx].load(Relaxed);
        let raw_fail = self.oracle_fail[domain_idx].load(Relaxed);
        // Only adapt if we have ≥5 samples (Wilson confidence)
        if raw_ok + raw_fail < 5 { return 0.50; }
        (0.50 - (ema - 0.5) * 0.20).clamp(0.40, 0.60)
    }

    /// Adaptive ctx threshold for a specific domain
    pub fn ctx_thresh(&self, domain_idx: usize) -> f64 {
        let ema = self.crystal_ema[domain_idx].load(Relaxed) as f64 / 1000.0;
        let raw_ok = self.crystal_success[domain_idx].load(Relaxed);
        let raw_fail = self.crystal_fail[domain_idx].load(Relaxed);
        if raw_ok + raw_fail < 5 { return 0.28; }
        (0.28 + (0.5 - ema) * 0.20).clamp(0.20, 0.38)
    }
}
```

---

## 2. Exponential Decay Mechanics

The EMA formula `EMA_new = α × x + (1-α) × EMA_old` with α=0.10 means:
- The last observation has weight 10%
- 10 queries ago: weight ~6.5%
- 50 queries ago: weight ~0.5%
- **Half-life**: ~6.6 queries

This means the system adapts to changing query patterns within ~20-30 queries.

### Fixed-Point Lock-Free Update

EMA is stored as `u64 × 1000` (0..1000 range represents 0.0..1.0).
The update uses `compare_exchange` for correctness without a mutex:

```
Thread 1: reads 700 → computes 0.10×1.0 + 0.90×0.700 = 0.730 → CAS(700→730)
Thread 2: reads 700 → computes 0.10×0.0 + 0.90×0.700 = 0.630 → CAS fails (now 730) → retry
```

---

## 3. Calibration Engine (Background Task)

Runs every 50 queries via a Tokio background task:

```rust
tokio::spawn(async move {
    loop {
        tokio::time::sleep(Duration::from_secs(30)).await;
        let ps = planner_stats_v2();
        if ps.total.load(Relaxed) % 50 != 0 { continue; }

        for d in 0..N_DOMAINS {
            let ok  = ps.oracle_success[d].load(Relaxed) as f64;
            let fail = ps.oracle_fail[d].load(Relaxed) as f64;
            let n = ok + fail;
            if n < 10.0 { continue; }

            // Wilson score confidence interval
            let p_hat = ok / n;
            let z = 1.96;  // 95% CI
            let ci_width = 2.0 * z * (p_hat * (1.0 - p_hat) / n).sqrt();

            if ci_width < 0.15 {
                // Sufficient statistical confidence — log adjustment
                tracing::info!(
                    "CALIBRATION[v0.60]: domain={} oracle_ema={:.3} ci_width={:.3} \
                     → thresh={:.3}",
                    DOMAIN_NAMES[d], p_hat, ci_width,
                    ps.direct_thresh(d)
                );
                ps.calibrations.fetch_add(1, Relaxed);
            }
        }
        ps.persist_snapshot();
    }
});
```

---

## 4. Multi-Path Learning

When both oracle and crystal return valid results (PIPELINE route), track which
was more informative (longer, higher scoring):

```rust
// In PIPELINE-DIRECT branch:
match (&oracle_text, &crystal_text) {
    (Some(oc), Some(cc)) => {
        // Learn which was preferred (length as proxy for information content)
        if oc.len() >= cc.len() {
            ps.oracle_preferred[domain_idx].fetch_add(1, Relaxed);
        } else {
            ps.crystal_preferred[domain_idx].fetch_add(1, Relaxed);
        }
        // Construct response prioritizing the preferred source
        let oracle_pref_rate = ...;
        let resp = if oracle_pref_rate > 0.6 {
            format!("**Cálculo:**\n{}\n\n**Contexto:**\n{}", oc, cc)
        } else {
            format!("**Contexto:**\n{}\n\n**Cálculo:**\n{}", cc, oc)
        };
        break 'p060 Some(resp);
    }
}
```

---

## 5. Real-Time Dashboard

### TUI Tab: "Planner" (ratatui)

```
┌─ Qomni Planner v0.60 ────────────────────────────────────┐
│ Route Distribution (last 100)   Threshold Evolution       │
│ PIPELINE  ████████░░  42%       legal_peru  0.47 → 0.44  │
│ ORACLE    ████░░░░░░  18%       contabilidad 0.50 → 0.50  │
│ CRYSTAL   ██░░░░░░░░  9%        hidraulica  0.50 → 0.46  │
│ CTX       ███░░░░░░░  15%       nfpa_elec.  0.50 → 0.48  │
│ FREE      ████░░░░░░  16%                                 │
│                                                           │
│ Domain Confidence Heat Map                                │
│ legal_peru  [oracle=0.78] [crystal=0.82] ████████ 0.80   │
│ contabilid  [oracle=0.50] [crystal=0.50] ████░░░░ 0.50   │
│ hidraulica  [oracle=0.85] [crystal=0.60] █████████ 0.73  │
│ nfpa_elec   [oracle=0.65] [crystal=0.70] ██████░░ 0.68   │
│ tecnico_it  [oracle=0.50] [crystal=0.50] ████░░░░ 0.50   │
│ whatsapp    [oracle=0.50] [crystal=0.72] ██████░░ 0.61   │
│                                                           │
│ Total: 247 queries │ Calibrations: 4 │ Uptime: 2h 14m   │
└──────────────────────────────────────────────────────────┘
```

### Web Endpoint: GET /qomni/planner/dashboard

Returns HTML with:
- Chart.js bar chart: route distribution
- Line chart: threshold evolution per domain (last 24h)
- Table: per-domain EMA, sample count, Wilson CI
- Auto-refresh every 10 seconds

---

## Implementation Plan

| Step | Change | Effort |
|------|--------|--------|
| 1 | PlannerStatsV2 struct (array-based) | 1h |
| 2 | EMA update in all 9 route branches | 30min |
| 3 | domain_idx lookup from best_domain string | 15min |
| 4 | adaptive_thresh_for_domain() functions | 30min |
| 5 | Calibration background task | 1h |
| 6 | Multi-path preference tracking | 30min |
| 7 | TUI tab "Planner" | 1.5h |
| 8 | /qomni/planner/dashboard HTML endpoint | 1h |
| **Total** | | **~6h** |
