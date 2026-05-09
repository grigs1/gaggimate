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

## Priority Summary

| Priority | Improvement |
|---|---|
| ★★★ | 1.1 — Roast date replaces manual age field |
| ★★★ | 3 — Pre-populate shot history; user only rates |
| ★★★ | 5.1 — Immediate post-shot rating overlay |
| ★★ | 2.1 — Auto-fill Dose In from BLE scale |
| ★★ | 1.2 — Auto-fill bean fields from profile on Bean ID select |
| ★★ | 4.1 — Auto-trigger GetRecommendations when inputs ready |
| ★★ | 4.3 — "Applied" confirmation button for grind advice |
| ★ | 1.3 — Remember last bean on app open |
| ★ | 4.2 — Quick-repeat collapsed mode for returning beans |
| ★ | 5.2 — Taste quick-select tag palette |
| ★ | 6.1 — Session persistence across reloads |
| ★ | 6.3 — Grinder position memory |
| ★ | 7.1 — Unsaved shot reminder |
