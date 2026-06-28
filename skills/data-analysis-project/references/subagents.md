# Subagent delegation patterns

Optional, for when the environment can spawn subagents. Three roles pay off the most: a
**data-source scout** (Stage 3), a **data-quality reviewer** (Stage 6), and a
**critic/evaluator** (Stages 5 & 8). They work by keeping noisy or biased work *out* of the
main thread — the scout keeps search noise out of context; the data-quality reviewer turns a
strict-checker report into a per-column treatment plan; the critic brings in a perspective
the author can't have.

If subagents aren't available, do the same work inline — just be deliberate about it
(summarize sources to a shortlist yourself; self-critique against the checklists below).

---

## 1. Data-source scout (Stage 3)

**Why:** discovering and access-probing open data floods the context with search results,
portal HTML, and trial API responses. Run it in a subagent so the main thread receives
only a clean shortlist and spends its context on the analysis.

**Spawn one scout** (or a few in parallel, each assigned a different angle: official
portals / web & community / a specific provider). Each returns a shortlist; the main
thread merges, picks, and deep-probes the winner itself.

**Scout prompt template:**
```
Find candidate datasets for this analysis goal: <goal + entities + needed grain/period>.
Do the messy exploration yourself — search portals/web, open dataset pages, and where an
API/CSV is exposed, fetch a TINY sample to confirm it's real and readable.
Return ONLY a shortlist (no raw search dumps, no full pages). For each candidate:
  - name / provider / source URL
  - access method (OpenAPI / CSV / portal download / scrape) + auth needs (key?)
  - key fields + grain (e.g. per-station monthly) + time span + update cadence
  - encoding/format quirks you actually observed (e.g. cp949, mixed quoting)
  - license
  - access difficulty (easy / needs key / messy schema / paywalled)
  - fit-to-goal: 1 sentence on how well it answers the goal, and gaps
Rank best-fit first. Flag any that look stale or too coarse for the goal.
```

**Return schema (what the main thread consumes):**
```
[{ name, source_url, access, auth, key_fields, grain, span, cadence,
   quirks, license, difficulty, fit, gaps }]
```

The main thread then verifies the chosen source's schema/keys itself (Stage 3 gate) before
building on it — don't trust the scout's sample as the final word on the join key.

---

## 2. Critic / evaluator (Stages 5 & 8)

**Why:** the person who designed the plan or wrote the conclusion is the worst judge of its
holes. A subagent prompted to *refute* gives an independent, adversarial read. Default it
to skeptical — its job is to find what's wrong, not to reassure.

### Stage 5 — evaluate the method plan
**Critic prompt template:**
```
You are a skeptical senior analyst. Here is a proposed analysis plan: <plan table:
sub-question → method → data → verification>. Goal: <Stage-2 goal>. Data facts: <grain,
size, span, gaps, known quirks>.
Try to break it. For each method, judge:
  - Does it actually fit THIS goal and THIS data's characteristics? (e.g. seasonal model
    on <2 cycles, classifier on extreme imbalance, correlation read as causation)
  - Is there a trivial baseline it must beat? Is one planned?
  - Is the verification real (holdout/backtest, right metric) or hand-wavy?
  - What's missing, confounded, or likely to mislead?
Return: per-method verdict (sound / weak / wrong) + the specific weakness + a concrete fix.
End with the single biggest risk to the whole plan. Do not praise.
```

### Stage 8 — evaluate conclusions & insights
**Critic prompt template:**
```
You are a skeptical reviewer. Here are the analysis results + draft conclusions:
<insights, each with its supporting numbers/figures>. Original goal: <Stage-2 goal +
finalized 정량 목표: metric + threshold>.
First check the goal verdict: does the conclusion state MET / PARTIAL / NOT-MET vs the
finalized threshold, and is it honest (the achieved number actually clears it — no false
success, no metric quietly switched/loosened)?
For EACH insight, judge:
  - Do the cited numbers actually support this claim, at this strength? (check the math/
    direction, not the vibe)
  - Alternative explanations / confounds / selection effects?
  - Is it overclaimed (causal language on correlational evidence, extrapolation)?
  - Does the set of insights actually answer the goal? Which part is unanswered?
  - What's unexamined that could flip the conclusion? (missing segment, period, variable)
Return: per-insight verdict (supported / overclaimed / unsupported) + why + how to fix or
qualify. List missing angles. Default to skeptical when evidence is ambiguous.
```

**Use the verdicts:** revise the plan/conclusions, re-qualify overclaimed insights, and add
the critic's "missing angles" to the limitations section (Stage 8) and report (Stage 9). If the
goal verdict is **NOT MET**, honor it — loop back or publish an honest negative; never soften it
into a false success.
If the critic and author disagree, the honest move is usually to soften the claim or add
the caveat — not to defend it.

### Stronger variant — perspective-diverse panel
For high-stakes conclusions, spawn 2–3 critics with *different lenses* (statistical
validity / domain-business plausibility / data-quality) rather than identical skeptics —
diversity catches failure modes a single reviewer misses. Keep an insight only if it
survives the majority.

---

## 3. Data-quality reviewer (Stage 6)

**Why:** the strict gate (`scripts/data_quality.py --strict`) *detects* problems
deterministically, but the right *treatment* for a missing value or an outlier depends on
the data's shape, the goal, and the chosen method — that's judgment, not a fixed rule.
Delegate the judgment so the main thread gets a per-column plan it can apply and log. (A
script alone can't adapt to arbitrary data shapes — this reviewer is what makes the strict
checker safe to apply across varied datasets.)

**Reviewer prompt template:**
```
You are a careful data-preparation reviewer. Here is the data-quality report:
<paste data_quality.py output or --json>. Goal: <Stage-2 goal>. Chosen method: <Stage-5
method>. Data shape facts: <grain, size, series vs cross-section, skew, seasonality>.
For EACH flagged column/issue (missing, outlier, duplicate, constant, high-cardinality),
recommend ONE treatment + a one-line rationale tied to the data shape AND the goal/method:
  - missing → impute (mean/median/mode/ffill/interpolate/group-wise/model) | drop-rows |
    drop-col | leave-and-document — weigh the missingness mechanism (MCAR/MAR/MNAR)
  - outlier → cap/winsorize | drop | keep-as-signal (it may BE the phenomenon) | transform
  - duplicate → dedup on key | investigate the source
Flag any LEAKAGE or metric-distorting fix (target-derived feature, future info, stats
computed across a train/test split, winsorizing away the effect being measured).
Prefer the least-destructive option when unsure; everything must be logged.
Return: a table {column, issue, recommendation, method, rationale, leakage_risk} + an
overall residual-risk note + which HARD gate failures must be resolved before Stage 6 passes.
```

**Return schema:**
```
[{ column, issue, recommendation, method, rationale, leakage_risk }] + residual_risk + must_fix[]
```

**Use it:** apply the recommended treatments, **log every drop/impute**, then re-run
`python scripts/data_quality.py <prepared_table> --strict` and only pass the Stage 6 gate
when it exits 0. The script flags, the reviewer decides, the main thread applies.
