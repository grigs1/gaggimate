# Adaptive Brew — UX Improvements: Reduce Manual Data Entry

Goal: the user should do the minimum possible input. The system should infer,
remember, carry-forward, and auto-fill everything it already knows.

---

## 1. Bean & Freshness Inputs

### 1.1 Store roast date, compute age automatically  ★ HIGH PRIORITY
Replace the "Roast Age (days)" manual field with a **Roast Date** date-picker
(entered once at registration). Age is then computed as `today − roastDate`
on every GetRecommendations call — the user never touches it again.

Current: user must mentally compute and re-enter days every session.  
Proposed: `roastAgeDays = currentDate − profile.roastDate` — always fresh.

### 1.2 Auto-fill bean fields from registered profile
Once a Bean ID is selected from a dropdown of known beans, all read-only
fields (Roast Level, Processing, Roast Date) are populated automatically
from the bean profiles JSON. The user only confirms Dose In.

Current: user must re-enter roast level and processing every session.  
Proposed: selecting a known Bean ID locks and pre-fills those fields.

### 1.3 Remember last session's bean on app open
On page load, pre-select the most recently used Bean ID so returning users
start with their last bean already active.

### 1.4 Freshness window notifications
Show a badge or toast when roast age crosses a threshold:
- < 7 days → "Too Fresh — degassing, predictions may vary"
- 7–21 days → "Optimal window"
- 21–42 days → "Aging — expect finer grind creep"
- > 42 days → "Stale — consider a new bag"

User doesn't need to track this mentally; the system surfaces it.

### 1.5 Bag-change prompt instead of silent re-registration
When the bag-change detection fires (|shift| > threshold), show a prompt:
"New bag detected — please confirm roast date for [Bean ID]."
One field, one tap. No need to re-register the full bean.

---

## 2. Dose — BLE Scale Integration

### 2.1 Auto-fill Dose In from BLE scale  ★ HIGH PRIORITY
The `BLEScalePlugin` already exists in the firmware. When a scale is connected
and beans are placed on the portafilter, capture the stable weight and
auto-fill **Dose In** in the recommendation card — no typing required.

### 2.2 Auto-fill Dose Out / Yield from BLE scale
During extraction, capture the cup weight at shot end and auto-fill
**actual yield ratio** in the shot history form. The user never enters it.

### 2.3 Carry-forward last Dose In as default
If no scale is connected, pre-fill Dose In from the previous session's value
for the same bean. Most users use a consistent dose per bean.

---

## 3. Shot History — Pre-population  ★ HIGH PRIORITY

When the shot ends, the system already holds all of this data. Pre-fill the
shot history form automatically so the user only needs to provide a score.

| Field | Source | Manual? |
|---|---|---|
| Bean ID | Current session input | No — carried from input card |
| Roast Level | Bean profile | No — from registered profile |
| Processing | Bean profile | No — from registered profile |
| Roast Age (days) | Computed from roast date | No — automatic |
| Dose In | Input card / BLE scale | No — carried from input |
| Grind Setting | Prediction output | No — from recommendation |
| Extraction Time | Machine (brew timer) | No — from controller |
| Dose Out / Ratio | BLE scale or flow meter | No — from sensor |
| Channeling Score | ChannelingDetector | No — from real-time analysis |
| Channeling Severity | Derived from score | No — automatic |
| Pre-infusion Delay | Profile setting | No — from machine |
| Profile ID | Machine | No — read-only |
| Predicted Taste | TastePredictor | No — from prediction |
| **Score / Rating** | **User** | **Yes — the only required input** |
| Notes (free text) | User | Optional |

The shot history form opens post-shot as a review screen, not a data-entry
screen. Everything is pre-filled; the user taps a star rating and saves.

---

## 4. Recommendations — Reduce Interaction Friction

### 4.1 Auto-trigger GetRecommendations when inputs are complete
If Bean ID is selected (known bean, all fields auto-filled) and Dose In is
populated, call GetRecommendations automatically on load — no button tap.
Show a loading state and update results. The user arrives at the card and
sees the recommendation already waiting.

### 4.2 Quick-repeat mode for returning beans
When the same Bean ID is used as the previous session, show a **"Same as
yesterday"** summary bar with just the updated grind recommendation and
freshness badge. Full input form is collapsed behind a "Change inputs" toggle.

### 4.3 Grind advice confirmation button
After the post-shot grind advice is shown ("go finer 1 step"), add a
**"✓ Applied"** button. Tapping it logs `grinderApplied: true` to shot notes,
which lets the system know the next shot started from the advised position.
This closes the loop between advice and the next prediction's `grindAdjustment`
reference point.

### 4.4 Inline adjustment controls
Instead of going to the grinder, adjusting, coming back, show a **+/−** stepper
next to the grind advice. The user taps it to confirm how many steps they
actually moved (may differ from advice), and this is stored as the actual
next-shot starting grind.

---

## 5. Post-Shot Rating — Minimal Effort

### 5.1 Immediate rating overlay on shot end  ★ HIGH PRIORITY
As soon as the shot timer stops, show a compact overlay:
```
  How was that shot?
  ★ ★ ★ ★ ☆
  [Too sour]  [Balanced]  [Too bitter]
  [Save & Learn]   [Skip]
```
Two taps (stars + taste label) and the system has everything it needs to
learn. The full shot history form is available for detail but not required.

### 5.2 Taste quick-select instead of free text
Replace open-ended taste notes with a multi-select tag palette:
`Sour · Bright · Balanced · Sweet · Chocolatey · Bitter · Astringent · Flat`
Stored as structured tags, not free text — usable by TastePredictor.

### 5.3 Prompted "did you follow the recommendation?"
If the user's actual grind differs from the recommended grind, show a one-tap
prompt: "Did you use the recommended grind of 27?" → [Yes] / [No, I used ___].
This guards against the model learning from a shot taken at an unintended setting.

---

## 6. Session Persistence & Memory

### 6.1 Restore full session state on app reload
All input fields, the last recommendation, and any unsaved post-shot data
should survive a page reload or connection drop. Store in browser localStorage
with a session timestamp. Stale sessions (> 4 hours) are discarded.

### 6.2 Multi-bean quick-switch
Show the last 3 used beans as one-tap chips at the top of the input card.
Tapping a chip loads that bean's full profile and refreshes recommendations
without any form interaction.

### 6.3 Grinder position memory
Store the last confirmed grind setting (either from "Applied" tap or from
saved shot notes) as `currentGrinderPosition` in settings. The pre-shot panel
shows: "Grinder currently at 27 — recommendation is 27 (no change needed)."
Removes ambiguity about whether to re-adjust before the next shot.

---

## 7. Notifications & Proactive Hints

### 7.1 Unsaved shot reminder
If a brew completed but no shot rating was saved within 10 minutes, show a
notification: "Don't forget to rate your last shot — it helps the model learn."

### 7.2 Model confidence milestone alerts
When the bean model crosses a confidence tier, notify the user:
- "You've pulled 5 shots of [bean] — bean model is now active (75% confidence)"
- "10 quality shots reached — running weight optimisation in background"
- "20 shots exported — ready for TFLite model training"

### 7.3 Grind drift advisory
If the recommended grind for a bean has trended consistently in one direction
over the last 10 shots (e.g., always getting finer), surface: "Grind has
shifted −3 steps over 2 weeks — burr check or new bag?"

---

## 8. Metric Explanations — Plain Language & Benchmarks

Every number shown in the UI should carry enough context that a user who has
never read a barista manual can understand what it means and what to do.
Currently the UI shows raw values with severity labels. Proposed: each metric
gets an expandable explanation row with a benchmark bar and a one-sentence
plain-language interpretation based on the current value.

---

### 8.1 Channeling Score (0–100)

**Plain language definition:**
"Measures how evenly water flowed through your coffee puck. A perfect score
is 0 — water moved through every part of the puck equally. A high score means
water found a shortcut (a 'channel') through one part of the puck and bypassed
the rest, leaving some coffee under-extracted and some over-extracted."

**Benchmark bar:**

```
0 ──────────── 20 ──────────── 40 ──────────── 65 ────── 100
  None (clean)    Mild (ok)      Moderate (⚠)    Severe (✗)
```

**Context messages by range:**

| Score | Message shown to user |
|---|---|
| 0–19 | "Clean shot — water moved evenly through the puck. No channeling detected." |
| 20–39 | "Minor flow variation — within normal range. Won't significantly affect taste." |
| 40–64 | "Moderate channeling — part of your puck was bypassed. Expect slight unevenness in the cup. Check puck prep next shot." |
| 65–100 | "Severe channeling — a major path formed through the puck. The shot was significantly compromised. Focus on distribution and grind before the next shot." |

---

### 8.2 Flow CV % (Coefficient of Variation of puck flow)

**Plain language definition:**
"Measures how smooth and steady the liquid flowed through the coffee during
extraction. Think of it like water through a garden: a well-packed puck gives
steady, laminar flow (low CV). A puck with weak spots causes surges and drops
as channels open and close (high CV). Anything above 30% is turbulent."

**Benchmark bar:**

```
0% ──── 15% ──────── 30% ──────────── 50% ──── 100%
  Excellent   Acceptable   Turbulent       Severe
```

**Context messages:**

| Flow CV | Message shown to user |
|---|---|
| < 15% | "Excellent flow consistency — your puck offered even resistance throughout." |
| 15–30% | "Normal variation — minor turbulence, within acceptable range for home espresso." |
| 30–50% | "High variation — flow was uneven. Suggests an inconsistent puck or early channeling. Improve distribution before the next shot." |
| > 50% | "Very erratic flow — the puck did not hold its structure. Combined with channeling, this indicates a puck prep or grind issue that needs attention." |

**Example (from screenshot — 57%):**
"Your flow varied wildly during extraction. At 57% CV, the puck was resisting
unevenly from start to finish — likely a combination of a too-fine grind
building excessive pressure and an uneven density in the puck that gave way
at one point."

---

### 8.3 Max Drop (bar)

**Plain language definition:**
"The biggest sudden pressure drop measured during your shot. During normal
extraction, pressure stays relatively stable. When water breaks through a weak
spot in the puck — a channel — pressure drops sharply as the path opens up.
The size of the drop tells you how significant that channel was."

**Benchmarks:**

| Max drop | Meaning |
|---|---|
| < 0.5 bar | Normal pump variation — no channel. |
| 0.5–1.0 bar | Minor flow disruption — small channel or puck settling. |
| 1.0–1.5 bar | Noticeable channel — water found a real path. Mild impact on taste. |
| > 1.5 bar | Significant channel event — puck integrity was compromised at a point. Expect uneven extraction. |

**Example (from screenshot — 1.6 bar):**
"At 1.6 bar, a real channel opened during your shot. The puck lost structural
integrity at one point and water rushed through that path, bypassing the rest
of the coffee bed."

---

### 8.4 Events Count

**Plain language definition:**
"How many separate channeling episodes occurred during the shot. Each 'event'
is a discrete moment when pressure dropped and flow spiked — a channel opening.
One event is a single weak point. Multiple events suggest the whole puck
structure is unstable."

**Context messages:**

| Events | Message |
|---|---|
| 0 | "No channeling events detected. Clean shot." |
| 1 | "One channeling event — a single weak point in the puck. Likely a distribution or tamping issue at one spot, or grind pressure became too high." |
| 2–3 | "Multiple channel events — the puck had several structural weak points. Focus on your distribution technique: spread coffee evenly before tamping." |
| ≥ 4 | "Frequent channeling — the puck had no structural integrity. Check grind freshness (stale coffee crumbles), dose weight, and basket condition." |

---

### 8.5 Expected Time

**Plain language definition:**
"How long the system expects your shot to take based on your grind and dose.
Espresso extraction time matters because too fast means under-extracted (sour,
thin), too slow means over-extracted (bitter, harsh). The ideal window for most
espresso styles is 25–35 seconds."

**Benchmarks:**

| Time | Meaning |
|---|---|
| < 18s | Far too fast — grind is much too coarse or dose too low. |
| 18–25s | Fast — slightly coarse. May taste thin or sour. |
| 25–35s | Ideal range for most espresso. |
| 35–50s | Slow — grind likely too fine. Bitter risk. |
| > 50s | Very slow — grind is too fine or dose too high. High channeling risk under pressure. |

**Example (from screenshot — 89s):**
"89 seconds is nearly three times longer than a normal espresso shot. This
strongly indicates the grind is far too fine. Under this much puck resistance,
water built up pressure until it forced a path through the puck — which is
exactly what the channeling detector recorded."

---

### 8.6 Extraction Yield %

**Plain language definition:**
"What percentage of the dry coffee mass ended up dissolved in your cup.
This tells you how thoroughly the coffee was extracted. Under-extracted coffee
(too low) tastes sour and thin; over-extracted coffee (too high) tastes bitter
and harsh. The sweet spot for most espresso is 18–22%."

**Benchmarks:**

| Yield | Meaning |
|---|---|
| < 16% | Under-extracted — sour, sharp, thin body. Go finer or slower. |
| 16–18% | Light extraction — bright acidity, lighter body. |
| 18–22% | Ideal — balanced acidity, sweetness, body. |
| 22–24% | High extraction — fuller body, possible bitter edge. |
| > 24% | Over-extracted — bitter, harsh, drying. |

**Example (from screenshot — 24.0%, shown in amber):**
"At 24%, you're at the upper limit. Combined with an 89-second shot time,
the coffee has been over-extracted — expect bitter, astringent notes. Going
coarser will bring both the time and the yield into range together."

---

### 8.7 Grind Advice Confidence

**Plain language definition:**
"How much trust to place in the grind recommendation. Confidence grows as the
system learns your specific machine, grinder, and bean combination. At low
confidence, the recommendation is a starting point based on bean type —
useful, but not yet personalised."

**Context messages:**

| Confidence | Message |
|---|---|
| 0–35% | "First shots of this bean — recommendation based on bean type only. Treat it as a starting point and adjust based on taste." |
| 35–55% | "Some data collected — recommendation is partially personalised. Getting more reliable." |
| 55–75% | "Good confidence — the model has learned your setup. Follow the recommendation." |
| 75–95% | "High confidence — strong bean model active. Recommendation is well calibrated to your machine and grinder." |

---

### 8.8 Method Badge

**Plain language definition for each tier:**

| Method | What it means |
|---|---|
| **Metadata prior** | "Using your bean type (roast level + processing) as a starting point. The system hasn't yet learned your specific machine and grinder for this bean." |
| **Bean model** | "The system has learned from your previous shots of this bean. Recommendation is personalised to your setup." |
| **Similar beans** | "No shots of this bean yet. Using data from other beans with similar characteristics as a starting point." |
| **KNN** | "Using patterns from all your shot history to predict the best grind." |
| **Rules** | "Minimal data available — applying general espresso principles. Very first shot." |

---

## 9. Shot Coach — Barista Diagnosis & Action Plan

After every shot, the system already has all the signals needed to make a
barista-level diagnosis. Currently the UI shows raw numbers. Proposed: a
**Shot Coach** section that synthesises all signals into a plain-language
explanation of what happened and exactly what to do differently next time —
phrased the way an experienced barista would explain it.

### 9.1 Shot Coach Panel Layout

```
┌─────────────────────────────────────────────┐
│  SHOT COACH                                  │
│                                              │
│  🔍 What happened                            │
│     [Primary diagnosis in 1–2 sentences]     │
│                                              │
│  🎯 Most likely cause                        │
│     [Root cause with physical explanation]   │
│                                              │
│  ✅ Next shot — do this                      │
│     [Specific, ranked action list]           │
│                                              │
│  ℹ  Why                                      │
│     [Brief explanation of the cause-effect]  │
└─────────────────────────────────────────────┘
```

---

### 9.2 Diagnosis Message Library

The system selects and composes a message from the library below based on the
combination of signals. Priority: time anomaly → channeling events → CV → yield.

---

#### Scenario A — Grind too fine (long time + channeling)
**Trigger:** Time > 50s AND channeling ≥ Moderate

**What happened:**
"Your shot ran for {time}s — well above the 25–35s target. The grind was too
fine, creating a puck that resisted water so strongly that pressure built up
until it forced a path through the weakest point."

**Most likely cause:**
"Fine grind → high puck resistance → excessive pressure build-up → water
broke through rather than flowing evenly. This is the most common cause of
channeling in very slow shots."

**Next shot — do this:**
1. Go {N} steps coarser on your grinder (suggested: 2–3 for > 70s, 1–2 for 50–70s)
2. Before tamping, distribute the coffee evenly — a fine grind amplifies any
   unevenness in the puck. Tap the portafilter or use a WDT tool.
3. Tamp level with consistent pressure (~15 kg). A tilted tamp creates a thin
   side where water channels first under high pressure.

**Why:** A fine grind is very sensitive to puck prep — small gaps in density
become the path of least resistance when pressure is high.

---

#### Scenario B — Puck prep issue (normal/fast time + multiple events)
**Trigger:** Time 20–45s AND Events ≥ 2

**What happened:**
"Shot time was {time}s — reasonable — but {n} separate channeling events
occurred. When time is correct but channeling is still present, the grind is
usually not the problem. The puck itself had structural weak points."

**Most likely cause:**
"Uneven coffee distribution before tamping, a tilted tamp, or clumping. With
multiple channel events, water found several different paths — this is a puck
structure problem, not a grind problem."

**Next shot — do this:**
1. **Distribute before tamping:** After grinding, tap the portafilter on the
   counter 2–3 times and use a finger or WDT tool to break up any clumps and
   level the coffee bed.
2. **Check your tamp angle:** Place the portafilter on a flat surface and tamp
   straight down. Even 2–3° of tilt creates a thin side that channels first.
3. **Check dose:** If the coffee is touching or nearly touching the shower
   screen, reduce dose by 0.5 g — compressed grounds can crack and channel.
4. Grind change: not needed yet — rule out puck prep first.

**Why:** Multiple events means the puck had several structurally weak zones.
This is almost always a distribution or tamping issue, not grind size.

---

#### Scenario C — Grind too coarse (fast shot)
**Trigger:** Time < 20s AND yield < 17%

**What happened:**
"Shot pulled in {time}s — much faster than the 25–35s target. Water moved
through the puck too easily, leaving the coffee under-extracted. Expect a thin,
sour, sharp cup."

**Most likely cause:**
"Grind is too coarse → puck offers too little resistance → water flows through
quickly without extracting enough."

**Next shot — do this:**
1. Go {N} steps finer (suggested: 2 steps for < 18s, 1 step for 18–22s)
2. If you recently cleaned or reassembled the grinder, check burr alignment —
   grinder disassembly can shift the grind coarser.
3. Make sure the dose matches your basket size. Under-dosing (too little coffee)
   also reduces puck resistance.

---

#### Scenario D — Single event, high pressure drop, correct time
**Trigger:** Time 25–45s AND Events = 1 AND Max drop > 1.5 bar

**What happened:**
"Shot time was {time}s — within range — but one significant channeling event
occurred (pressure dropped {drop} bar). A single strong event like this
usually points to one specific weak spot in the puck."

**Most likely cause:**
"One area of the puck had lower density than the rest — typically caused by
an uneven grind distribution or a clump that didn't break up. Water pooled
on that side and eventually punched through."

**Next shot — do this:**
1. **Distribute more carefully:** After grinding into the portafilter, look at
   the coffee bed before tamping. It should be level and consistent. Use a
   Stockfleth move (finger around the rim) or a WDT tool to redistribute.
2. **Check for clumps:** Some grinders produce clumps, especially in humid
   conditions. Break them up before tamping — they become channels.
3. Grind: probably fine. Focus on distribution first.

---

#### Scenario E — Over-extracted (high yield + long time)
**Trigger:** Time > 40s AND Yield > 22%

**What happened:**
"Your shot extracted {yield}% of the coffee's soluble content over {time}s.
Both numbers are above the ideal range. The coffee was in contact with water
for too long and has likely extracted harsh, bitter compounds."

**Most likely cause:**
"Grind too fine → slow flow → long contact time → over-extraction."

**Next shot — do this:**
1. Go {N} steps coarser to bring the time down to 25–35s.
2. Your yield will follow the time — as the shot speeds up, yield will drop
   into the 18–22% range naturally.
3. If yield stays high after adjusting grind, reduce dose by 0.5 g.

**Taste tip:** If the cup tastes bitter and drying (astringent), you can
confirm over-extraction. If it's bitter but has a clean finish, it may just
be the bean's roast profile — try a slightly lower brew temperature.

---

#### Scenario F — Clean shot
**Trigger:** Score < 20 AND Time 25–35s AND Yield 18–22%

**What happened:**
"Clean shot — all signals in range. No channeling detected, flow was steady,
and extraction yield is in the sweet spot."

**Next shot — do this:**
"Nothing to change technically. If the taste was good, save this as a
reference shot for this bean. If you want to fine-tune taste:
- Slightly sour → go 0.5 steps finer or add 1g dose
- Slightly bitter → go 0.5 steps coarser
- Flat/thin → try a 1–2 °C higher brew temperature"

---

### 9.3 Puck Prep Advisory (when channeling is persistent)

If channeling score ≥ Moderate for 3 consecutive shots of the same bean
despite grind adjustments, show an additional advisory:

"You've had moderate or higher channeling on 3 consecutive shots despite
grind changes. When the grind isn't the problem, puck preparation usually is.
Check these one at a time:

**① Distribution:** After grinding, the coffee bed must be level and uniform
before tamping. Clumps or mounds on one side become channels.
→ Try: tap portafilter 3× on a flat surface, then use a finger to level.

**② Tamp angle:** Even a slight tilt creates a thinner side. Place the
portafilter on a flat surface and press straight down — wrist locked, elbow
above the handle.

**③ Tamp pressure consistency:** Aim for ~15 kg of force on every shot.
Variable pressure produces variable puck density.

**④ Grinder alignment:** If burrs are misaligned, grounds are uneven in size —
fines clump and coarses create voids. A burr alignment kit resolves this.

**⑤ Basket condition:** Inspect the basket for cracks, dents, or holes. A
damaged basket creates channeling regardless of technique."

---

### 9.4 Shot Coach — Rendering Logic (summary)

```
signals = { time, channeling_score, events, max_drop, flow_cv, yield }

if time > 50 and channeling >= MODERATE:         → Scenario A (grind too fine)
elif events >= 2 and time in [20, 45]:           → Scenario B (puck prep)
elif time < 20 and yield < 17:                   → Scenario C (grind too coarse)
elif events == 1 and max_drop > 1.5 and time ok: → Scenario D (single weak spot)
elif time > 40 and yield > 22:                   → Scenario E (over-extracted)
elif score < 20 and time ok and yield ok:        → Scenario F (clean shot)
else:                                            → Generic: show individual metric messages
```

Multiple conditions can apply — show the highest priority match first with a
"Also note:" for secondary signals.

---

## Priority Summary

| Priority | Area | Improvement |
|---|---|---|
| ★★★ | Data entry | 1.1 — Roast date replaces manual age field |
| ★★★ | Data entry | 3 — Pre-populate shot history; user only rates |
| ★★★ | UX | 5.1 — Immediate post-shot rating overlay |
| ★★★ | Coaching | 9 — Shot Coach: barista diagnosis + action plan |
| ★★ | Data entry | 2.1 — Auto-fill Dose In from BLE scale |
| ★★ | Data entry | 1.2 — Auto-fill bean fields from profile on Bean ID select |
| ★★ | UX | 4.1 — Auto-trigger GetRecommendations when inputs ready |
| ★★ | UX | 4.3 — "Applied" confirmation button for grind advice |
| ★★ | Explanations | 8.1–8.3 — Channeling / Flow CV / Max drop plain-language tooltips |
| ★★ | Explanations | 8.5–8.6 — Expected time and yield contextual benchmarks |
| ★ | Data entry | 1.3 — Remember last bean on app open |
| ★ | UX | 4.2 — Quick-repeat collapsed mode for returning beans |
| ★ | UX | 5.2 — Taste quick-select tag palette |
| ★ | Memory | 6.1 — Session persistence across reloads |
| ★ | Memory | 6.3 — Grinder position memory |
| ★ | Notifications | 7.1 — Unsaved shot reminder |
| ★ | Coaching | 9.3 — Persistent channeling advisory (puck prep checklist) |
