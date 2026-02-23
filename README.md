## Proof of concept: Central Limit Theorem

The first end-to-end output of the pipeline is a complete formalization of the **Lindeberg-Levy Central Limit Theorem** in Lean 4.27.0 / Mathlib v4.27.0.

No `sorry`, `admit`, or `native_decide`. ~830 lines across 4 files. The human provided the proof architecture (characteristic function route, lemma decomposition, which Mathlib API to target). The agent handled all tactic-level proof search, `lake build` iteration, error resolution, and Mathlib API navigation. No human intervention occurred between `./math.sh full` invocations.

Degenne's independent CLT formalization for Mathlib covers the same theorem via a different proof route. The two efforts converging on the same result from different directions — one human-driven for Mathlib contribution, one agent-driven as a capability demonstration — is useful signal that the agent's output is mathematically sound rather than an artifact of overfitting to a single proof strategy.

### Main result

```lean
theorem central_limit_theorem
    {X : ℕ → Ω → ℝ}
    (hindep : iIndepFun X P)
    (hident : ∀ i, IdentDistrib (X i) (X 0) P P)
    (hmeas : ∀ i, AEStronglyMeasurable (X i) P)
    (hL2 : MemLp (X 0) 2 P)
    (hvar : 0 < variance (X 0) P) :
    Tendsto (β := ProbabilityMeasure ℝ)
      (fun n ↦ ⟨P.map (fun ω ↦
        (∑ i ∈ Finset.range n, X i ω - ↑n * ∫ x, X 0 x ∂P) /
        Real.sqrt (↑n * variance (X 0) P)), ⋯⟩)
      atTop (𝓝 ⟨gaussianReal 0 1, inferInstance⟩)
```

For i.i.d. real-valued random variables with finite positive variance, the law of the standardized sum converges weakly to the standard Gaussian.

### Proof structure

| File | Lines | Purpose |
|------|-------|---------|
| `CLT/CharFun/ExpBound.lean` | 125 | Pointwise bounds on `exp(iy) - polynomial` for all real y |
| `CLT/CharFun/Taylor.lean` | 219 | Second-order Taylor expansion of charFun via DCT |
| `CLT/LevyContinuity.lean` | 215 | Levy continuity theorem (both directions) |
| `CLT/CentralLimitTheorem.lean` | 273 | CharFun factorization, power limit, and the CLT |

**Proof chain:** Exponential bounds provide dominators for DCT. Taylor expansion gives `charFun μ t = 1 + i·t·m₁ - t²·m₂/2 + o(t²)`. Levy continuity converts pointwise charFun convergence to weak convergence (forward: tightness via `measureReal_abs_gt_le_integral_charFun`, Prokhorov compactness, `Measure.ext_of_charFun` uniqueness; converse: Portmanteau). The CLT factorizes the charFun of i.i.d. sums as a power, applies Taylor to get `n·(ψ(t/√(nσ²)) - 1) → -t²/2`, takes the n-th power limit via `Complex.tendsto_one_add_pow_exp_of_tendsto`, and lifts through Levy continuity.

### Construction log

The full record of what was proved, what failed, what was retried, and what dead-end approaches were recorded is in [`CONSTRUCTION_LOG.md`](CONSTRUCTION_LOG.md). The negative knowledge accumulated during proving is in [`DOMAIN_CONTEXT.md`](DOMAIN_CONTEXT.md) under `## DOES NOT APPLY`.

## Building the POC

```bash
lake update    # first time only
lake build
```

Requires Lean 4.27.0 (see `lean-toolchain`). Mathlib v4.27.0 is pinned in `lakefile.toml`.

## Theorem inventory

| Declaration | File |
|-------------|------|
| `norm_cexp_mul_I_sub_one_sub_le` | `ExpBound.lean` |
| `norm_cexp_mul_I_sub_one_sub_le_sq` | `ExpBound.lean` |
| `norm_cexp_mul_I_taylor2_le` | `ExpBound.lean` |
| `norm_cexp_mul_I_taylor2_le_cube` | `ExpBound.lean` |
| `charFun_taylor_remainder_isLittleO` | `Taylor.lean` |
| `charFun_taylor_centered` | `Taylor.lean` |
| `charFun_taylor_centered_unit_variance` | `Taylor.lean` |
| `levy_continuity` | `LevyContinuity.lean` |
| `tendsto_charFun_of_tendsto_probabilityMeasure` | `LevyContinuity.lean` |
| `charFun_iid_sum_eq_pow` | `CentralLimitTheorem.lean` |
| `central_limit_theorem_charFun` | `CentralLimitTheorem.lean` |
| `central_limit_theorem` | `CentralLimitTheorem.lean` |

# claude-mathematics-kit

An orchestration framework for autonomous formal mathematics. An LLM agent operates under phased constraints — file locks, tool-use hooks, error summarization — to produce Lean 4 / Mathlib proofs from natural-language specifications without human intervention beyond the initial `./math.sh full <spec>` invocation.

~3,000 lines of orchestration infrastructure (`math.sh`, `scripts/`, `.claude/prompts/`, `.claude/hooks/`). The agent under orchestration is Claude (Anthropic).

## Architecture

The pipeline decomposes formalization into 8 phases, each executed by a fresh sub-agent with phase-specific tool permissions:

```
SURVEY → SPECIFY → CONSTRUCT → FORMALIZE → PROVE → POLISH → AUDIT → LOG
```

Each phase gets its own prompt (`.claude/prompts/math-<phase>.md`), its own file-permission state, and its own hook enforcement rules. The orchestrator (`math.sh`) sequences phases, manages revision loops, and handles inter-phase context transfer via disk — the sub-agents never share an LLM context window.

### Design decisions

**Phase-locked file permissions.** During PROVE, spec files are `chmod 444`. During AUDIT, all `.lean` files are locked. Without this, the agent "fixes" build errors by relaxing theorem statements instead of fixing proofs. The permission system makes the wrong thing impossible rather than merely discouraged.

**Pre-tool-use hooks.** A Claude Code hook (`.claude/hooks/pre-tool-use.sh`) intercepts every Edit, Write, and Bash call. It blocks `axiom`, `unsafe`, `native_decide`, and `admit` in all phases. During PROVE, it parses the Edit target to detect signature modifications and blocks them. During FORMALIZE, it blocks proof tactics — only `sorry` is allowed. Without this, the agent introduces `admit` to close difficult goals, or writes proofs during formalization that later constrain the proof strategy.

**Error summarization.** `lake-summarized.sh` wraps `lake build`. On success, it emits one line (`BUILD OK | clean | 34s`). On failure, it pipes through `lean-error-summarize.sh`, which strips Mathlib type expansions, classifies errors (`TYPE_MISMATCH`, `TACTIC_FAIL`, `UNKNOWN_IDENT`, `TIMEOUT`, `UNIVERSE_INCOMPAT`), extracts goal states, and caps output at 40 lines. Without this, a single type mismatch error can expand to 200+ lines of Mathlib internals and consume the agent's entire context window.

**Sorry batching.** `enumerate-sorrys.sh` locates every `sorry` with its enclosing theorem name and line number. `batch-sorrys.py` groups them into batches of 5, respecting file boundaries. The prove phase can run in chunked mode, spawning one sub-agent per batch. Without batching, a file with 20 sorrys overwhelms the agent; it loses track of which sorry it's working on and produces overlapping edits.

**Negative knowledge accumulation.** `DOMAIN_CONTEXT.md` has a `## DOES NOT APPLY` section where the prove agent records failed approaches: lemma names that don't exist in the current Mathlib version, tactic strategies that produce type mismatches, API renames. The next revision cycle reads this before starting. Without it, agents retry `measureReal_univ` (doesn't exist; correct name is `probReal_univ`) on every attempt.

**Dependency-ordered construction queue.** `CONSTRUCTIONS.md` defines a DAG of proof targets. `resolve-deps.py` topologically sorts them and selects the next unblocked construction. Program mode (`./math.sh program`) auto-advances through the queue, running `full` on each construction in dependency order, marking downstream constructions as blocked on failure.

**Revision loop with bounded retries.** If AUDIT finds sorrys or axioms, it writes `REVISION.md` specifying which phase to restart from. The `full` pipeline re-enters at that phase, up to 3 revisions. The revision file is archived after each attempt. Without bounded retries, the agent loops indefinitely on an impossible goal.

### Enforcement

The hook system (`pre-tool-use.sh`) performs the following concrete checks:

| Phase | Blocked action | Mechanism |
|-------|---------------|-----------|
| All | `axiom`, `unsafe`, `native_decide`, `admit` in file writes | Regex on tool input |
| All | `chmod`, `sudo`, destructive git (`revert`, `checkout`, `restore`) | Regex on Bash input |
| All | `lake clean` (preserves olean cache) | Regex on Bash input |
| SURVEY | Any file write or shell redirect | Tool type check |
| SPECIFY | Write to `.lean` files | Filename regex |
| CONSTRUCT | Write to `.lean` files | Filename regex |
| FORMALIZE | Proof tactics in `.lean` writes (only `sorry` allowed) | Tactic keyword regex |
| FORMALIZE | Raw `lake build` (must use summarized wrapper) | Command regex |
| PROVE | `Write` to `.lean` (must use `Edit`) | Tool type check |
| PROVE | Edit that modifies theorem/lemma/def signatures | JSON parse of `old_string` |
| PROVE | Write to spec files | Filename regex |
| POLISH | Edit that modifies proof bodies | Tactic keyword regex in `old_string` |
| AUDIT | Any write to `.lean` or spec files | Filename regex |

## License

Apache 2.0
