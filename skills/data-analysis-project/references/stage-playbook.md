# Stage playbook — tactics per stage

Detailed guidance for each workflow stage. Read the section for the stage you're in.

## 1. 문제 제기 — frame & qualify (loop with the user)

The goal is to avoid analyzing a problem that is vague, imaginary, or un-actionable.
Intuition is a hypothesis, not evidence — and "interesting" is not the same as "useful".
Run a short **qualification loop** with the user before touching data.

### The three tests
Write the problem as one statement, then check it against all three. Track pass/fail and
show the user each round.

1. **구체성 (Specific)** — does it name the subject, a *measurable* phenomenon, and a
   time/scope bound? Vague: "매출이 안 좋다." Specific: "2024–2025 재방문 고객 비중이
   분기마다 하락." If vague, ask which metric, which segment, which period.
2. **실재성 (Real)** — is there evidence? Gather an article/statistic or do a **cheap data
   sniff** (one query that would reveal the problem if real). Cite sources (title, outlet,
   date, URL). If no signal appears, say so and reframe — don't proceed on faith.
3. **실행가능성 → 비즈니스 인사이트 (Actionable)** — the **"so what?" test**: *if we answered
   this perfectly, what decision or action changes?* If the answer is "nothing concrete",
   it won't produce a business insight. Reshape until the answer maps to a real lever:
   budget allocation, station/store placement, pricing, retention program, staffing, etc.

### The loop
For each failing test, ask the user the specific thing you need and re-assess. Keep
looping until all three pass **and the user confirms**.

```
Round 1 — 문제안: "따릉이 대여소가 부족하다"
  구체성 ✗ (어디가? 무엇으로 측정?)  실재성 ?  실행가능성 ✗ ("so what" 불명확)
  → 사용자에게 질문: 어떤 의사결정을 위해서인가? (증설 예산 배분? 특정 지역?)
Round 2 — 문제안: "수요 대비 대여소가 부족한 지역을 찾아 증설 우선순위를 정한다"
  구체성 ✓  실재성 → 기사·데이터로 확인  실행가능성 ✓ (증설 예산 배분 결정)
  → gate 통과
```

### 상황 점검 — assess the situation (lightweight)
While qualifying, do a quick feasibility scan so a blocker surfaces now, not after modeling.
Keep it to a few lines — a checklist, not a phase:
- **자원 / 데이터 가용성** — beyond the 실재성 sniff, can you obtain the *full* dataset (needed
  span, grain, columns) at acceptable cost/access — not just the one sniff query? If not, reshape
  the problem toward data you can actually get.
- **법 · 프라이버시 · 라이선스** — personal/sensitive data? a usage license that forbids the
  intended use or redistribution? Flag it before relying on the source. (Nailed down in Stage 3.)
- **리스크 + 폴백** — name the 1–2 biggest risks (data too coarse, proxy variable, short span)
  and a fallback if they bite.
- **비용 대비 효과** — assuming the actionability ("so what?") test passed (a lever exists), is
  *pulling* that lever worth the analysis effort? A high-effort answer to a low-stakes decision is
  a reason to rescope.
- **(선택) 용어** — note any ambiguous business/technical terms so author and user mean the same.

A **reframe is a legitimate (often the best) outcome**: if evidence shows the real problem
is different from the user's assumption, surface it immediately and re-run the tests on
the new framing. Cheaper now than after modeling.

## 2. 분석 목표 설정 — set the goal (most important)

Weak goals are the single biggest cause of weak analyses. Spend real effort here.

- **Invoke `superpowers:brainstorming`** and loop with the user. Do not settle for the
  first phrasing. Push until the goal has all of:
  1. a **specific question** (not "understand X" — "does adding stations raise ridership?"),
  2. the **decision** the answer informs (so scope stays tied to something actionable),
  3. a **two-part success criterion**:
     - **비즈니스 합격선** — the decision-level outcome that makes it useful.
     - **정량 목표** — a number **derived from the decision, not the model** (the model is
       undecided until Stage 5). The problem *type* fixes the metric *family* — pick what
       fits; **don't default to one metric**:

       | 문제 유형 | 정량 지표 예시 (여러 개 중 택) |
       |---|---|
       | 예측 / 회귀 | MAPE · sMAPE · RMSE · MAE · R² |
       | 분류 | operating point의 precision / recall · F1 · AUC · PR-AUC · lift@k |
       | 랭킹 / 타겟팅 | precision@k · recall@k · lift · NDCG |
       | 공간 / 커버리지 | N m 내 수요 충족률 · 미충족 지점 수 |
       | 관계 / 효과 | 상관 r · 효과크기(effect size) · 회귀계수+CI · 설명력(% variance) |

       **Prefer a baseline-relative bar** ("naive / seasonal-naive / mean / majority / 현행
       규칙 대비 ≥X 개선") since the model is unknown — it is always settable and matches the
       baseline-first stance. Use an **absolute bar** ("오차 ±10% 이내", "AUC ≥ 0.75") only when
       the decision dictates one. If neither is settable yet: "Stage 5에서 확정, 최소 ≥ baseline".
- Convert each method idea into a check: "if the answer is A we do P, if B we do Q."
- Write the goal as one sentence and **get explicit user confirmation**.
- **Gate:** one-sentence goal + the decision + both criteria (business 합격선 & a model-agnostic
  정량 목표) + user sign-off. Touching data before this is premature.

Keep a short list of sub-questions; each will map to a method in Stage 5.

## 4. 데이터 이해 — profile the sourced data

You now have a real sample (Stage 3). Profile it **before** planning method, so Stage 5
fits the data you *have*, not the data you assumed.

- **Run the profiler.** `python scripts/data_quality.py <file>` prints per-column dtype,
  null-rate, cardinality, distribution stats, and flags outliers (IQR + robust-z),
  duplicates, and constant/all-null columns. Read it — don't eyeball a `head()` and move on.
- **Measure the method-driving facts.** Size (rows × cols), grain (one row = ?),
  seasonality/trend (plot the series), gaps (missing periods), class balance (target
  distribution). These are the inputs to Stage 5 — record the actual numbers.
- **Stay goal-targeted.** Profile what the goal/method needs; resist a generic "analyze
  everything" EDA dump.
- **Gate:** a short written profile of the *real* data — every key field's null-rate +
  distribution seen, outliers/dupes/constant columns flagged, size/grain/seasonality/
  imbalance measured (not assumed). Carry the flags into Stage 6.

## 5. 분석 방법 — plan the method

Choose methods that fit the goal **and** the data's real characteristics (now measured in Stage 4).

- Match method to data: short series → avoid models needing many seasonal cycles;
  heavy class imbalance → pick suitable metrics; spatial questions → mapping/coverage.
- **Baseline first (predictive work).** For predictive projects, always include a trivial
  baseline (naive/seasonal-naive/mean, or majority class) that any complex model must beat. Often
  the baseline wins — that's a finding, not an embarrassment. (Descriptive/unsupervised endpoints
  have no such baseline — use method-appropriate validity instead: silhouette/k-selection, CI/effect size.)
- Plan **comparison and verification**: holdout/backtest scheme, the metric (chosen from the
  Stage-2 family — MAPE/RMSE/MAE for forecasts, AUC/F1/lift@k for classification, r/effect size
  for relationships, % coverage for spatial), and a sanity check per step.
- **Finalize the Stage-2 정량 목표** into an exact, checkable bar now that the method is known
  (e.g. "backtest MAPE ≤ 15%", "precision@10% ≥ 2× random", "r ≥ 0.5 on holdout", "수요 충족률
  ≥ 90%"). This is the bar Stage 8 judges against — keep it consistent with the Stage-2 target.
- Note data limitations now (gaps, lags, proxy variables) so they flow into Stage 8.
- **Gate:** a written plan — a table mapping each goal sub-question → method → verification, with
  the **named metric + finalized threshold**. Use the planning template below.

### Method-plan template
```
| Sub-question | Method | Data used | Verification |
|---|---|---|---|
| ... | ... | ... | backtest MAPE / corr / map / ... |
```

## 6. 데이터 준비 — prepare & validate the modeling table

Turn the raw, profiled data into the **analysis-ready table** the notebook will model.
CRISP-DM puts 70–80% of effort here — treat it as a real stage with a hard gate, not a
stray "clean/join" cell.

- **Treat what the profile flagged.** Missing values and outliers have no fixed fix — the
  right move depends on the data shape, the goal, and the chosen method. Delegate to the
  `data-quality` subagent: it reads the `data_quality.py` report and recommends per-column
  treatment — impute (mean/median/mode/ffill/interpolate/group-wise/model) vs drop-rows vs
  drop-col vs cap/winsorize vs **keep-as-signal** — with rationale, and flags leakage.
  - Don't mean-impute a skewed or seasonal series; don't drop rows when missingness isn't
    random (it biases); outliers may **be** the signal (demand spike, fraud) — don't blindly clip.
- **Engineer + select with intent.** Build the features the Stage-5 method needs; select
  rows/columns deliberately. **Log every drop and impute** — silent fixes are how a result
  becomes irreproducible.
- **Validate the join.** Check row-counts before/after a merge and verify cardinality (no
  silent fan-out); assert post-merge null-rates didn't explode.
- **Guard leakage.** No target-derived features, no future information, no stats computed
  across a train/test boundary.
- **Gate:** `python scripts/data_quality.py <prepared_table> --strict` exits 0 (null-rate
  within bound, no full-row/key duplicates, no constant/all-null columns); join cardinality
  verified; leakage checked; every transformation logged. Re-run until it passes.

## 8. 결론 · 비즈니스 인사이트 — conclude & critique

Three parts, all required.

**Verdict against the goal (do this first):** restate the finalized 정량 목표 (Stage-5 metric +
threshold) and the 비즈니스 합격선, put the **achieved number** beside each, and declare
**MET / PARTIAL / NOT MET** — a numeric comparison, not a vibe. Use whatever metric your goal
actually chose (don't force everything into MAPE):

```
| 기준 | 임계 | 달성 | 판정 |
|---|---|---|---|
| backtest MAPE          | ≤ 15%        | 22%   | NOT MET |
| baseline 대비 개선     | ≥ 30%        | 41%   | MET     |
| precision@10%          | ≥ 2× random  | 1.7×  | PARTIAL |
| 상관 r (holdout)       | ≥ 0.5        | 0.58  | MET     |
| N m 내 수요 충족률     | ≥ 90%        | 76%   | NOT MET |
```

**Never call an unmet threshold a success** (no false success). A surface metric looking fine ≠
the business 합격선 met — check both, and report the one that fails.

**Insights:** conclusions tied to **specific numbers**, each translated to a business action
("util > X stations risk stockouts → rebalance"). No generic advice ungrounded in a result.

**Self-critique (do not skip):** framed right? If the real lever is elsewhere, state the
**reframe** with a number. What data was missing that would change the conclusion (OD, weather,
competitors, retention, price)? What's the next study?

**go/no-go on a shortfall:** if the verdict is PARTIAL / NOT MET, **ask the user** which way to
go — (a) **loop back** to refine method/data (Stage 5/6) or reframe the problem (Stage 1/2), or
(b) **proceed** and publish an honest "목표 미달 — 사유 + 다음 수" report. Don't silently ship as
if the goal were met. A reframe or an honest negative is a strong result.

**Gate:** an explicit MET/PARTIAL/NOT-MET verdict (achieved vs threshold, numeric) on both
criteria; every insight cites a result; limitations name what would flip the conclusion; and on
a shortfall, the user has chosen loop-back vs honest-publish.

This honest reckoning is frequently the most valuable part of the report. Resist the urge to
declare success when the data is ambiguous or the threshold went unmet.
