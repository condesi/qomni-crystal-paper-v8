# CRYS-L v2 — Language Specification & Grammar

## What is CRYS-L?

**CRYS-L** (Crystal Language) is an open, domain-specific language for expressing
deterministic engineering calculations as declarative *plan programs*.
A CRYS-L program describes **what to compute**, not how — the runtime
(CRYS-L JIT, compiled to WASM) handles numeric dispatch, unit validation,
and result formatting.

CRYS-L is designed to be:
- **Human-readable**: plain-English keywords, no boilerplate
- **Verifiable**: all formulas are from published standards (NFPA, IEC, IS.010)
- **Zero-LLM**: executes deterministically, no neural network involved
- **Community-extensible**: add new `plan_*` declarations via pull request

---

## Grammar (EBNF)

```ebnf
program         ::= plan_decl+

plan_decl       ::= 'plan_' identifier '(' param_list? ')' '{' body '}'

identifier      ::= [a-z][a-z0-9_]*

param_list      ::= param (',' param)*
param           ::= identifier ':' type ('=' default)?

type            ::= 'f64' | 'f32' | 'i64' | 'bool' | 'str'

default         ::= literal

body            ::= (const_decl | let_decl | formula | assert | output | meta)+

const_decl      ::= 'const' identifier '=' expr ';'
let_decl        ::= 'let' identifier '=' expr ';'
formula         ::= 'formula' string_lit ':' expr ';'
assert          ::= 'assert' expr 'msg' string_lit ';'
output          ::= 'output' identifier 'label' string_lit ('unit' string_lit)? ';'
meta            ::= 'meta' '{' meta_field+ '}'

meta_field      ::= identifier ':' string_lit ','

expr            ::= term (op term)*
term            ::= literal | identifier | call | '(' expr ')'
call            ::= identifier '(' expr_list? ')'
op              ::= '+' | '-' | '*' | '/' | '^' | '%'
literal         ::= number | string_lit | bool_lit
number          ::= [0-9]+ ('.' [0-9]+)?
string_lit      ::= '"' [^"]* '"'
bool_lit        ::= 'true' | 'false'
expr_list       ::= expr (',' expr)*
```

---

## Built-In Functions

| Function | Description | Example |
|----------|-------------|---------|
| `sqrt(x)` | Square root | `sqrt(area)` |
| `pow(x, n)` | Power | `pow(r, 0.63)` |
| `min(a, b)` | Minimum | `min(q, q_max)` |
| `max(a, b)` | Maximum | `max(0.0, delta)` |
| `abs(x)` | Absolute value | `abs(delta_h)` |
| `log(x)` | Natural log | `log(ratio)` |
| `log10(x)` | Base-10 log | `log10(pressure)` |
| `round(x, n)` | Round to n decimals | `round(hp, 2)` |

---

## Standard Library Plans (v2)

### Hydraulics

```crysl
plan_hazen_williams(
    Q: f64,          // flow rate (L/s)
    D: f64,          // internal diameter (mm)
    C: f64 = 120.0,  // roughness coefficient (PVC=140, steel=120)
    L: f64           // pipe length (m)
) {
    meta {
        standard: "Hazen-Williams (SI)",
        source: "IS.010 Peru / AWWA M22",
        domain: "hidraulica",
    }

    const A = 3.14159 * pow(D / 1000.0, 2.0) / 4.0;
    let V = Q / (1000.0 * A);
    let R = (D / 1000.0) / 4.0;

    formula "Hazen-Williams velocity": V = 0.8492 * C * pow(R, 0.63) * pow(S, 0.54);
    let S = pow(V / (0.8492 * C * pow(R, 0.63)), 1.0 / 0.54);
    let h_loss = S * L;

    assert V > 0.0 msg "flow velocity must be positive";
    assert D > 0.0 msg "diameter must be positive";

    output V      label "Velocity"      unit "m/s";
    output h_loss label "Head loss"     unit "m";
    output A      label "Pipe area"     unit "m²";
}
```

### Fire Protection (NFPA)

```crysl
plan_pump_sizing(
    Q_gpm: f64,       // required flow (GPM)
    P_psi: f64,       // required pressure (PSI)
    eff:   f64 = 0.7  // pump efficiency (0-1)
) {
    meta {
        standard: "NFPA 20 - Fire Pump Standard",
        source: "NFPA 20:2022, Section 4.26",
        domain: "nfpa_electrico",
    }

    const GPM_TO_LPS = 0.06309;
    const PSI_TO_M   = 0.7031;

    let Q_lps   = Q_gpm * GPM_TO_LPS;
    let H_m     = P_psi * PSI_TO_M;
    let HP_req  = (Q_lps * H_m) / (eff * 76.0);
    let HP_max  = HP_req * 1.20;        // NFPA 20: shutoff ≤ 140% rated

    assert Q_gpm > 0.0 msg "flow must be positive";
    assert P_psi > 0.0 msg "pressure must be positive";
    assert eff > 0.0   msg "efficiency must be > 0";
    assert eff <= 1.0  msg "efficiency must be ≤ 1.0";

    output HP_req  label "Fire pump HP required" unit "HP";
    output HP_max  label "Max shutoff HP (NFPA)" unit "HP";
    output Q_lps   label "Flow rate"             unit "L/s";
    output H_m     label "Total head"            unit "m";
}
```

### Electrical

```crysl
plan_voltage_drop(
    P_kw:  f64,       // load power (kW)
    V:     f64,       // nominal voltage (V)
    L:     f64,       // cable length ONE-WAY (m)
    fp:    f64 = 0.9, // power factor
    rho:   f64 = 0.0175 // copper resistivity (Ω·mm²/m)
) {
    meta {
        standard: "IEC 60364-5-52 / NEC 210.19",
        source: "IEC 60364:2019",
        domain: "nfpa_electrico",
    }

    let I    = (P_kw * 1000.0) / (V * fp);
    let I_design = I * 1.25;   // NEC 125% rule

    // Minimum section for 3% drop (IEC recommendation)
    let drop_limit = V * 0.03;
    let S_min = (2.0 * rho * L * I) / drop_limit;
    let delta_V = (2.0 * rho * L * I) / S_min;
    let drop_pct = (delta_V / V) * 100.0;

    assert P_kw > 0.0 msg "power must be positive";
    assert V > 0.0    msg "voltage must be positive";
    assert L > 0.0    msg "length must be positive";

    output I        label "Nominal current"     unit "A";
    output I_design label "Design current"      unit "A";
    output S_min    label "Min cable section"   unit "mm²";
    output delta_V  label "Voltage drop"        unit "V";
    output drop_pct label "Drop percentage"     unit "%";
}
```

---

## CRYS-L v2 Additions (over v1)

| Feature | v1 | v2 |
|---------|----|----|
| `meta {}` block | No | Yes — standard, source, domain |
| `assert` statements | No | Yes — runtime validation |
| `formula` declarations | No | Yes — named formula strings |
| Default parameters | No | Yes |
| String type | No | Yes |
| Output units | No | Yes |
| Multi-domain stdlib | 2 | 6+ |
| WASM compilation target | Yes | Yes |
| Browser runtime (CRYS-L JIT) | Yes | Yes (v2.1) |
| Community PR path | No | **Yes** |

---

## Community Extension Guide

To add a new plan to the CRYS-L standard library:

1. Create a `.crysl` file in `crys-l/stdlib/`
2. Follow naming: `plan_{domain}_{calculation}()`
3. Include a `meta {}` block with `standard`, `source`, `domain`
4. Add at least one `assert` per input
5. Add test vectors in `crys-l/tests/{plan_name}.json`
6. Submit PR to `condesi/qomni-crystal-paper-v8`

### Test Vector Format

```json
{
  "plan": "plan_hazen_williams",
  "cases": [
    {
      "inputs": {"Q": 2.5, "D": 150.0, "C": 120.0, "L": 200.0},
      "expected": {"V": 0.141, "h_loss": 0.089},
      "tolerance": 0.005,
      "source": "IS.010 Peru Example 3.2"
    }
  ]
}
```

---

## Supported Domains (v2)

| Domain | Plans Available | Standard |
|--------|----------------|----------|
| `hidraulica` | hazen_williams, bernoulli, darcy_weisbach, pipe_sizing | IS.010/IS.020 Peru |
| `nfpa_electrico` | pump_sizing, voltage_drop, cable_sizing, transformer | NFPA 13/20/72, IEC 60364 |
| `civil` | beam_deflection, column_load, slab_thickness | ACI 318, NTE Peru |
| `mecanica` | shaft_power, torque, gear_ratio | ISO 6336 |
| `termica` | heat_transfer, hvac_load, insulation | ASHRAE 90.1 |
| `sanitaria` | water_demand, tank_sizing, sewage_flow | IS.010 Peru |

---

## Runtime: CRYS-L JIT (Open)

The CRYS-L JIT compiler and WASM runtime are **open-source** under MIT license.

- Compiles `.crysl` plans to WASM modules
- Executes in browser (no server) via `qomni.clanmarketer.com/crysl/`
- Also available as server-side Rust evaluator
- Source: `crys-l/` directory in this repo

The Qomni Engine integration (how CRYS-L plans are invoked from the planner)
is proprietary — but the language, compiler, and runtime are fully open.
