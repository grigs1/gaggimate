# Adaptive Brew — Gap Analysis & Improvement Proposals

Based on `adaptive-brew-diagram.html` (PR #4).

---

## 1. Prediction Pipeline Gaps

### 1.1 Fallback condition `grind≤0` is ambiguous
The SVG tier arrows are labelled `grind≤0` as the condition to fall to the next tier.
This conflates two different failure modes:

- **No data** — the tier was never attempted (no bean registered, too few shots).
- **Invalid result** — the tier ran and returned a nonsensical value.

A grind setting of exactly 0 is a possible (degenerate) valid result on some stepless grinders.
The transition condition should be a named enum: `RESULT_NO_DATA | RESULT_INVALID | RESULT_OK`.

### 1.2 Transfer-learning similarity metric is undefined (Tier 0b)
The diagram says "similarity-weighted grind average from similar known beans" without defining
similarity. No implementation can be written from this spec. Proposed metric:

```
similarity(A, B) =
  w_roast    × roastProximity(A, B)      // 1.0 if same level, 0.5 if ±1 level, 0 otherwise
+ w_process  × processProximity(A, B)    // 1.0 same; 0.5 Natural↔Honey; 0 otherwise
+ w_baseline × gaussianProximity(grindBaseline_A, grindBaseline_B, σ=1.5 steps)
```

Weights `(w_roast=0.4, w_process=0.35, w_baseline=0.25)` can be defaults that WeightOptimizer
later tunes. A minimum similarity threshold (e.g., 0.3) should gate inclusion.

### 1.3 Dose coefficient fallback 0.05 steps/g is a single global constant
When all historical shots share the same dose, `doseCoeff` falls back to `0.05 steps/g`
regardless of grinder type. A conical grinder with 40 steps/revolution has a very different
steps-per-gram sensitivity than a flat burr with 300 micro-steps. Two options:

- **Option A** — Add `grinderStepsPerGram` to the user-settings schema (one-time calibration
  shot: dose +2 g → measure grind delta → derive coefficient).
- **Option B** — Store a per-roast-level fallback table in `BeanGrindPrior` so at least the
  dose sensitivity varies by bean density (light roasts are less dense → larger grind swing
  per gram).

### 1.4 Blend alphas for Tier 0.5 early-model are hardcoded without rationale
`α = 0.4 / 0.6 / 0.8` for 2/3/4 shots are arbitrary constants. There is no mechanism to
learn whether the prior or the early model was more accurate in previous early-shot scenarios.
Proposed fix: track `priorError` (|prior_prediction − actual|) per bean across its first 4
shots and adjust α = 1 − exp(−priorError / σ_prior) so a better-matching prior is weighted
more heavily.

### 1.5 Extraction yield formula is a fixed linear equation
```
predictedYield = 18% + (time−28)×0.3 + (ratio−2)×2    clamp 14–24%
```
This formula:
- Uses hardcoded reference time (28 s) and ratio (2.0) that differ between espresso styles.
- Does not account for roast level (light roasts typically extract 1–3% higher at the same
  time/ratio).
- Does not account for dose-in (higher dose → lower extraction at the same ratio).
- Ignores grind fineness (finer → higher yield for the same time).

Proposed: fit the yield formula coefficients per-bean from shot history once ≥5 shots exist.
Until then keep the global linear fallback but at least add a roast-level offset
(`light +1.5%, med-light +0.5%, med 0, med-dark −0.5%, dark −1.5%`).

### 1.6 Temperature is an input dimension to KNN but never a recommended output
The 8-dim KNN feature vector includes `temp`, meaning the model implicitly learns that
temperature correlates with extraction quality. But the recommendations payload only exposes
`grindSetting` — temperature is never surfaced as advice. For light roasts especially,
a +1–2 °C adjustment can matter more than a half-step grind change.

Proposed addition to `res:adaptive-brew:recommendations`:
```json
"suggestedTemp": 93.5,
"tempReasoning": "Light roast + natural process — higher temp improves acidity clarity"
```
The metadata prior table (`BeanGrindPrior`) already has the processing×roast matrix; add a
parallel `tempOffset` column (in °C from the profile baseline).

### 1.7 Confidence is a point estimate with no uncertainty range
Confidence values (75%, 55%, 35%, …) are single numbers. A model trained on 5 shots that
happen to be tightly clustered gives the same 75% as one trained on 50 shots. There is no
signal to the user about *precision* vs. *accuracy*.

Proposed: add `grindRange: [low, high]` to the payload (the 80th-percentile KNN kernel
spread), shown in the UI as "28 ± 0.5 steps." This also guards against showing
"75% confidence, grind 28–34" which would be a red flag the user can act on.

### 1.8 No pre-infusion duration recommendation
`brewDelay` is shown in the payload with the annotation "(from settings, not AI)." Pre-infusion
duration is one of the most effective levers against channeling — yet the ML system never
recommends adjusting it. The channeling score and shot history already contain the correlation
signal. Proposed: add `suggestedPreInfusionMs` derived from the correlation between
`channelingScore` and `brewDelay` across the bean's shot history.

---

## 2. Channeling Detection Gaps

### 2.1 Score aggregation is additive with a hard cap
`score = min(100, signal1 + signal2 + signal3)` means three weak partial signals can produce
the same score as one dominant signal. Example: 30 pt pressure + 15 pt resistance + 20 pt
CoV = 65 (severe), which equals a single catastrophic pressure event (55 pt alone → mild at 55).

Proposed: weighted geometric mean or a simple Bayesian fusion where each signal contributes
a likelihood ratio, producing a posterior probability of channeling rather than an
uncalibrated integer score. This makes the severity thresholds statistically meaningful.

### 2.2 `flowCV` measurement window is unspecified
The puck-flow CoV signal contributes up to 30 points but the time window for the coefficient
of variation is not defined. A 500 ms window is dominated by pump pulsation noise; a 5 s
window misses a channel that opens and closes quickly. The diagram must specify: window
length, minimum sample count, and whether a rolling or block window is used.

### 2.3 No temporal trend across shots
A single shot with score 35 (mild) is ambiguous. Ten consecutive shots trending from 20 → 55
is a clear signal that something is changing (grind getting too coarse, burrs wearing, puck
prep degrading). The system currently resets per-shot with no inter-shot memory of channeling
trajectory. Proposed: add `channelingTrend` (`IMPROVING | STABLE | WORSENING`) computed
over the last N shots in `learnFromShot()`, surfaced in the post-shot panel and as an
advisory tag in `grindAdvice`.

### 2.4 No mid-shot alert or intervention hook
The `loop()` fires every 250 ms and accumulates a live score (`brewing_` flag exposes
`liveScore`). Yet the only consumer is a post-shot result. There is no mechanism to:
- Surface a real-time channeling warning in the UI during extraction.
- Optionally trigger a `pump:reduce-flow` event when score crosses a configurable threshold.

The event bus already supports this pattern. Adding a `CHANNELING_LIVE_ALERT_THRESHOLD`
setting and an outbound `adaptive-brew:channeling-live-alert` event would give users
(and future auto-response plugins) an actionable mid-shot signal.

---

## 3. Learning Loop Gaps

### 3.1 "Quality shot" is undefined
`WeightOptimizer` runs "every 10 quality shots" and `exportTrainingData()` requires
"≥20 quality shots," but quality is never defined in the diagram. Proposed definition:

| Criterion | Threshold |
|---|---|
| Extraction duration | 18–50 s |
| Yield ratio | 1.4–3.5 |
| Channeling score | < 65 (not severe) |
| Grind was not manually overridden post-recommendation | required |
| Shot has a user rating | optional but preferred |

Shots failing any hard criterion are stored but flagged `qualityExcluded: true` in notes JSON
and omitted from optimizer and training export (but retained for outlier-removal statistics).

### 3.2 User rating is stored but ignored by PostShotGrindAdvisor
`rating` appears in the notes JSON schema and in the KNN distance formula (`dist + decay + rating`)
but is not listed among the 5 signals of `PostShotGrindAdvisor` (time + ratio + taste +
channeling + freshness). A poorly-rated shot where the grind was at the recommended setting is
a direct feedback signal that the recommendation was wrong — stronger than any inferred signal.

Proposed: add `rating` as a 6th signal with weight ~0.25 (similar to freshness). Map:
- rating ≤ 2/5 at recommended grind → push adjustment in the direction of reported taste
  (sour → finer, bitter → coarser).
- rating ≥ 4/5 → reinforce current grind, reduce adjustment magnitude.

### 3.3 Bag-change threshold |shift| > 0.15 is a hardcoded constant
A bean with naturally high inter-shot variance (e.g., single-origin lot variation, altitude
storage humidity) may routinely show |shift| > 0.15 without any bag change. This would
fire a spurious bag-change correction on every new session, degrading confidence
unnecessarily.

Proposed: replace the fixed threshold with `|shift| > k × beanGrindStdDev` where
k ≈ 2.5 and `beanGrindStdDev` is stored per-bean in the profiles JSON.
For a new bean (< 3 shots, no variance estimate), fall back to the global 0.15.

### 3.4 WeightOptimizer grid search is computationally dangerous on ESP32
"Grid search on 8 feature weights" with 5-fold cross-validation on an ESP32-S3 at 240 MHz
is likely to take tens of seconds to minutes depending on grid resolution, blocking the main
loop or causing WDT resets. The diagram doesn't specify grid step size or time bounds.

Options:
- **Option A** — Coarse 3-level grid (low/med/high per dimension = 3^8 = 6561 evals),
  run in a background FreeRTOS task with a 30 s timeout, emit progress events.
- **Option B** — Replace grid search with coordinate descent (one dimension at a time),
  O(8 × grid_levels × folds) ≈ 120 evals, much safer on embedded hardware.
- **Option C** — Run optimization off-device: include current weights in `training-data-ready`
  export; accept optimized weights via a new `adaptive-brew:set-weights` inbound event.

### 3.5 TFLite model import path is missing
`exportTrainingData()` sends a JSON to SD and emits `adaptive-brew:training-data-ready`.
The diagram shows no path to bring a trained `.tflite` model back onto the device.
Without this, Tier 1 (TFLite ML, 60–95% confidence) can never be activated after the first
batch of shots. Required additions:
- **Inbound event**: `adaptive-brew:load-tflite-model { path: "/models/brew_v2.tflite" }`
- **Sub-component**: `TFLiteModelLoader` — validates file, loads into arena, replaces active model.
- **Outbound event**: `adaptive-brew:model-loaded { version, inputShape, accuracy }`

### 3.6 No time-decay for KNN historical shots
The KNN uses all historical shots (post MAD-3σ removal) with equal temporal weight. A shot
from 18 months ago on a different grind calibration (worn burrs, repositioned machine) can
drag predictions away from current reality. The `decay` term in the KNN distance formula is
mentioned but not defined — it is presumably a shot-recency factor.

Proposed: explicitly define decay as `exp(−λ × daysSinceShot)` with `λ = 0.005`
(half-weight at ~140 days, ~10% weight at ~460 days). Store `shotTimestamp` in notes JSON
(it may already exist via shot ID).

### 3.7 Integer rounding is not applied to PostShotGrindAdvisor output  ★ HIGH
The diagram specifies:
> *"Integer-scale auto-detect: round if all DB grinds are whole numbers"*

This rounding is applied to `grindSetting` (the absolute prediction) and consequently to
`grindAdjustment = predicted − last shot` in the pre-shot panel. Both values are already
protected — they will never show a fractional step on a stepped grinder.

However, `PostShotGrindAdvisor` outputs `adjustmentSteps` via a separate code path and the
integer auto-detect is **not applied there**. The advisor accumulates small per-signal deltas
and a trend multiplier and emits the raw float directly to the UI. On a grinder with integer
steps this produces unactionable advice such as "Coarser +0.15" — a step size the user
physically cannot apply.

**Root cause:** the advisor and the prediction pipeline share the same integer-detection flag
(`integerScaleDetected`) but only the prediction pipeline acts on it.

**Fix — apply the same guard in PostShotGrindAdvisor:**
```cpp
float steps = computeRawAdjustment();   // e.g. +0.15

if (db_.isIntegerScale()) {
    if (std::abs(steps) < 0.5f) {
        advice.direction   = DIRECTION_NONE;
        advice.borderline  = true;          // new flag
        advice.stepCount   = 0;
    } else {
        advice.stepCount = std::round(steps);  // ±1, ±2, …
    }
} else {
    advice.stepCount = steps;   // stepless grinder: keep decimal
}
```

**New `borderline` state in the UI:**
When `borderline = true`, instead of showing "Coarser +0.15" show:

> "Direction confirmed: slightly coarser — but the correction is smaller than one step.
>  Pull one more shot at the current setting. If still slow or bitter, go +1 step."

This prevents users from misreading a sub-step advisory as an instruction to move the grinder.

**How the two outputs eventually converge:**
The pre-shot prediction accumulates evidence shot by shot. When the bean model's internal
regression drifts past the 0.5-step boundary (typically after 2–4 more borderline shots), the
pre-shot `grindSetting` rounds up by one integer step and `grindAdjustment` shows "+1".
At that point both outputs agree and the borderline state clears automatically.

---

## 4. Input & Output Gaps

### 4.1 No grinder type or step-size input
The system auto-detects integer vs. decimal grind scale but cannot know absolute step sizes.
A Niche Zero at setting 20 is a completely different grind than a DF64 at setting 20.
`BeanGrindPrior`'s `globalGrindBaseline()` has no reference frame. Without a one-time
grinder calibration, the metadata prior is meaningless for new users.

Proposed: add a `grinderProfile` to settings:
```json
{ "model": "Niche Zero", "minSetting": 1, "maxSetting": 50, "stepsPerTurn": null }
```
This anchors the global baseline and allows cross-device bean profile sharing in future.

### 4.2 No user target for extraction yield
Users have preferences: a Third Wave fan targets 20–22% EY; a traditional Italian espresso
drinker targets 18–19%. The system always predicts yield but never optimises toward a user-
set target. `predictedYield` should be accompanied by `targetYield` (user setting, default
18–20%), and grind/temp advice should explicitly reference whether the prediction is trending
toward or away from target.

### 4.3 Taste prediction is a single label, not multi-dimensional
`predictedTaste` returns one of {balanced, sour, bitter, …}. Coffee taste has at least four
semi-independent axes: **acidity**, **body**, **sweetness**, **bitterness**. A natural process
light roast can be both high-acidity and high-body simultaneously — a single label loses this.

Proposed: return `predictedTaste` as:
```json
{ "label": "bright-acidic", "acidity": 7, "body": 6, "sweetness": 5, "bitterness": 3 }
```
Scores computed by separate KNN queries on each dimension from rated shots. Falls back to
label-only when < 5 rated shots exist.

### 4.4 Channeling score not used in taste prediction
`TastePredictor` uses dose, time, grind deviation, and bean KNN — but not `channelingScore`.
Channeling causes local under-extraction which manifests as sourness in an otherwise
well-timed shot. The taste predictor should receive the channeling result and up-weight
the sour prediction probability proportionally to channeling severity.

---

## 5. Architecture Gaps

### 5.1 30 s WebSocket timeout vs. long-running operations
`ApiService.js` resolves or rejects requests after 30 s. `WeightOptimizer` (5-fold CV +
grid search) and `exportTrainingData()` (JSON serialisation of potentially hundreds of shots)
may exceed this on the ESP32. The diagram has no async progress pattern.

Proposed: for operations that may exceed 10 s, switch to a fire-and-forget request:
- Client sends `req:adaptive-brew:optimize-weights` and immediately shows "optimising…" state.
- Firmware emits `adaptive-brew:optimization-progress { pct: 42 }` on the event bus
  periodically.
- Firmware emits `adaptive-brew:optimization-complete { improved: true, newWeights }` when done.
- `ApiService.js` subscribes to these broadcast events (no rid matching needed).

### 5.2 No versioning or rollback for bean models
When `WeightOptimizer` updates the 8 feature weights, or `learnBehavior()` updates regression
coefficients, there is no way to know if accuracy improved or degraded. A bad batch of shots
(miscalibrated scale, operator error) can silently corrupt the model.

Proposed: store `modelVersion` (integer) and `modelAccuracy` (cross-val MAE on last 5 quality
shots) in bean profiles JSON. If new accuracy > old + ε → accept; otherwise keep previous
weights and increment a `rejectionCount`. After 3 consecutive rejections, surface an advisory.

### 5.3 No error / degradation path for SD card failure
All persistence (notes JSON, bean profiles JSON, training export) goes to SD card. If the
SD card is removed, full, or corrupt mid-session, the diagram shows no fallback. At minimum:
- In-memory ring buffer of the last N shots (N = available heap / shot size).
- Emit `adaptive-brew:storage-error { reason }` so the UI can warn the user.
- Continue serving predictions from in-memory state; mark learned updates as `pendingPersist`.

### 5.4 No UI reconciliation when post-shot advisor and pre-shot recommendation disagree
`PostShotGrindAdvisor` (reactive, last-shot signals) and `getPredictionWithContext()`
(predictive, full history) are independent systems. They can produce outputs that appear to
contradict each other with no explanation offered to the user:

- Post-shot advisor: "Coarser +0.15" (or even after the fix: "borderline — direction coarser")
- Pre-shot recommendation next call: "Grind 28, adjustment 0 (no change)"

A user reading both panels will logically ask: *"Should I move the grinder or not?"* The UI
currently shows both numbers side by side with no reconciliation.

**When this disagreement occurs:**
The advisor fires immediately after a shot based on reactive signals. The prediction model
integrates those signals into the bean regression more slowly — it needs the sub-step drift
to accumulate past the rounding boundary before it updates the integer recommendation.
This means disagreement is normal and expected during the early learning phase and whenever
signals are borderline.

**Fix — add a reconciliation note between the two panels:**

| Situation | Message |
|---|---|
| Advisor says "borderline coarser" AND recommendation says "no change" | "The trend is pointing coarser but hasn't crossed a full step yet. Recommendation will update automatically once enough shots confirm the direction." |
| Advisor says "coarser +1" AND recommendation says "no change" | "Post-shot signals suggest a change but the full model needs more shots to agree. Consider following the advisor if taste is consistently off." |
| Advisor and recommendation agree | No message — no confusion. |

---

## Summary Table

| # | Area | Gap | Severity |
|---|---|---|---|
| 1.1 | Prediction | Fallback condition `grind≤0` is ambiguous | Medium |
| 1.2 | Prediction | Transfer similarity metric undefined | **High** |
| 1.3 | Prediction | Dose coefficient fallback is global constant | Medium |
| 1.4 | Prediction | Early-blend alphas are hardcoded | Low |
| 1.5 | Prediction | Yield formula is non-adaptive linear | Medium |
| 1.6 | Prediction | Temperature never recommended, only consumed | Medium |
| 1.7 | Prediction | Confidence has no uncertainty range | Low |
| 1.8 | Prediction | Pre-infusion not ML-driven | Low |
| 2.1 | Channeling | Additive score loses signal semantics | Medium |
| 2.2 | Channeling | CoV window undefined | **High** |
| 2.3 | Channeling | No inter-shot channeling trend | Medium |
| 2.4 | Channeling | No mid-shot alert or intervention | Medium |
| 3.1 | Learning | "Quality shot" undefined | **High** |
| 3.2 | Learning | User rating ignored by grind advisor | **High** |
| 3.3 | Learning | Bag-change threshold is a magic constant | Medium |
| 3.4 | Learning | Grid search unsafe on embedded hardware | **High** |
| 3.5 | Learning | TFLite import path missing | **High** |
| 3.6 | Learning | No time-decay on KNN history | Medium |
| 3.7 | Learning | Integer rounding not applied to PostShotGrindAdvisor output | **High** |
| 4.1 | Input | No grinder type/calibration input | Medium |
| 4.2 | Input | No user target yield | Low |
| 4.3 | Output | Taste prediction is single-label | Low |
| 4.4 | Output | Channeling not used in taste predictor | Medium |
| 5.1 | Architecture | 30 s WS timeout vs. long operations | **High** |
| 5.2 | Architecture | No model versioning or rollback | Medium |
| 5.3 | Architecture | No SD card failure degradation path | Medium |
| 5.4 | Architecture | No UI reconciliation when advisor and recommendation disagree | Medium |
