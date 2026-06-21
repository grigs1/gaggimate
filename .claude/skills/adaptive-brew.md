# Skill: /adaptive-brew

Firmware-native adaptive brew and grind prediction for gaggimate ESP32-S3.
No cloud, no external services, no LLM — everything runs on the device.

---

## What This Skill Does

When invoked, guides the implementation, extension, or debugging of the adaptive
brew system in the gaggimate C++ firmware. This includes:

- **Channeling detection**: real-time puck resistance monitoring during a shot,
  reduces pump power when channeling is detected
- **Grind prediction**: post-shot analysis of telemetry (yield ratio, extraction
  time, puck resistance stability) to recommend grind direction and magnitude
- **Bean metadata learning**: uses process type, roast grade, and bean age to
  estimate good starting parameters for a new bag before the first shot
- **Online learning**: RLS (Recursive Least Squares) and EMA algorithms that
  improve accuracy shot over shot, persisted in NVS flash (~1.2 KB total)

---

## Always Load First

Before responding, read the full architecture reference:

```
.claude/knowledge/ADAPTIVE_BREW_ARCHITECTURE.md
```

This file contains: exact file paths, data structures, algorithm specifications,
the bean knowledge matrix, integration points in existing files, and memory budget.
Do not guess file paths or data structure names — use those defined in that document.

---

## Invocation Modes

The user may call this skill with or without an argument:

### `/adaptive-brew` (no argument)
Ask the user what they want to do, then proceed:
- "Implement" → run the full implementation flow (see below)
- "Explain" → describe how a specific component works in plain language
- "Extend" → add a new capability to the existing system
- "Debug" → diagnose a problem with the adaptive brew behaviour
- "Status" → check what has already been implemented

### `/adaptive-brew implement`
Generate and write the complete implementation. Follow the priority order from
the architecture document exactly:
1. Create `src/display/core/adaptive/` directory
2. Generate each new file in priority order
3. Show the diff for each existing file that needs modification
4. Confirm memory budget is respected

Always write actual files using the Edit and Write tools — do not just print code.
After writing, confirm the files exist with a brief directory listing.

### `/adaptive-brew implement <module>`
Implement only the named module. Valid module names:
`channeling-detector`, `shot-metrics`, `bean-record`, `grind-learner`,
`knowledge-base`, `rls-regressor`, `freshness-tracker`, `webui`, `settings`

### `/adaptive-brew extend <feature>`
Add a new capability. Before writing any code:
1. Explain what the feature requires (new data, new signals, new NVS keys)
2. Identify which existing modules it touches
3. Confirm with the user before writing

Examples of valid extension requests:
- "Add dose tracking"
- "Add temperature auto-suggestion per roast"
- "Add inter-shot cooling guidance"
- "Add profile auto-selection based on bean type"
- "Add a staleness warning"

### `/adaptive-brew explain <component>`
Explain in plain language how a component works. Use no jargon beyond what the
user has already shown they understand. Reference actual line numbers from the
existing codebase where relevant.

### `/adaptive-brew status`
Read the `src/display/core/adaptive/` directory and existing modified files to
report what has been implemented vs what remains. Output a checklist.

### `/adaptive-brew debug`
Ask the user to describe the problem (wrong grind direction, no suggestions shown,
crash, NVS full, etc.), then trace through the relevant code path and identify
the likely cause. Propose a fix before writing any code.

---

## Implementation Rules

### File placement
All new modules go in `src/display/core/adaptive/`.
Do not create files elsewhere unless modifying an existing file.

### Existing files to modify
Only these three existing files need changes:
- `src/display/plugins/ShotHistoryPlugin.cpp` — trigger GrindLearner after shot
- `src/display/plugins/WebUIPlugin.cpp` — broadcast grind suggestion in status
- `src/display/core/Settings.h` — add `adaptiveBrew` bool and `activeBeanSlot` int

### Safety constraints — never violate these
- Never mutate a Profile struct — deltas are overlays only
- Never write NVS in the brew loop (Core 1 hot path) — use dirty flag + background save
- Discard shots shorter than 7.5 seconds
- Clamp all deltas: ±0.5 bar, ±0.4 ml/s, ±3g, max ±0.12 per shot step
- ChannelingDetector must be allocation-free (no heap in interrupt/timer context)

### No new libraries
Do not add entries to `platformio.ini` lib_deps. Everything uses:
- Arduino stdlib (std::deque, std::vector)
- ESP-IDF Preferences (already used in Settings.cpp)
- Standard math (fabsf, sqrtf, constrain)

### Code style
Match the existing codebase style:
- Header-only classes where possible (`.h` files)
- No raw pointers where references or values work
- `explicit` constructors
- `[[nodiscard]]` on getters that return computed values
- Include guards with `#ifndef / #define / #endif`

---

## WebUI Integration

The grind suggestion is surfaced through the existing WebSocket `evt:status` message
by adding a `grind` JSON object. No new WebSocket event type is required unless the
user explicitly asks for a dedicated event.

The bean metadata input extends the existing shot notes system via a new request
type `req:notes:bean`. This saves to the existing `/h/{id}.json` file AND updates
the GrindLearner's in-memory record.

Direction encoding for the frontend:
```
0 = None, 1 = Finer, 2 = Coarser, 3 = CheckPrep, 4 = OnTarget
```
Magnitude encoding:
```
0 = None, 1 = Slight, 2 = Moderate, 3 = Significant
```

---

## How the Grind Prediction Works (Plain Summary)

After every shot the firmware extracts three signals from the sample buffer:

1. **Yield ratio** — how much came out vs how much went in. Below 1.80 = under-extracted
   (grind finer). Above 2.45 = over-extracted (grind coarser).

2. **Extraction time** — how long the shot took. Below 22s = too fast (coarser grind
   pushed water through too easily). Above 36s = too slow (fine grind created too much
   resistance).

3. **Resistance stability** (coefficient of variation of puck resistance samples) —
   high variation means uneven flow = channeling. When this is high AND yield is low,
   the answer is puck prep, not grind.

If the user also provides taste feedback (sour/balanced/bitter), that overrides the
telemetry inference — it is the ground truth signal.

Over several shots the system builds an EMA (exponential moving average) of what
"good" looks like for each profile. With bean metadata, it seeds the first-shot
estimate from a compile-time knowledge matrix (process × roast grade) before any
shots have been pulled.

---

## How the Channeling Detector Works (Plain Summary)

Puck resistance is already measured every 250ms during a shot and stored in the
`pr` field of each ShotLogSample. When water finds a crack in the puck, resistance
drops sharply — the path of least resistance suddenly becomes much easier.

The detector keeps a 1-second rolling window of resistance values. If the peak
in that window is more than 30% higher than the current value, channeling is flagged.
The response is to reduce pump output to 85% — lower pressure means less force
pushing water through the crack, giving the puck a chance to reseal.

---

## Worked Example — New Bean Onboarding

User inputs: Ethiopian, Washed, Light, 14 days old.

1. `BeanKnowledgeBase::estimate()` looks up (Washed, Light):
   resistance offset +0.5, temp +2°C, target time 28s, yield 1.95

2. Freshness check: 14 days → optimal zone, no offset, EMA alpha = 0.3

3. Similarity search: scans NVS records for (Washed, Light) matches.
   Finds "Kenyan Washed Light" with 8 shots, converged resistance 3.8.
   Blends: 40% matrix + 60% past record → initial target resistance ≈ 4.0

4. WebUI shows before first shot:
   "Estimated start: aim for resistance ~4.0, time 26–29s, temp +2°C"

5. First shot: actual resistance 3.2, time 18s, yield 1.6 → under-extracted.
   Diagnosis: Finer, Significant.

6. EMA updates: resistanceEMA = 0.3×3.2 + 0.7×4.0 = 3.76.
   Record saved to NVS.

7. Second shot with finer grind: resistance 3.9, time 27s, yield 1.98 → good zone.
   Diagnosis: OnTarget. goodShotCount → 1.
   targetResistanceEMA seeded at 3.9.

8. Third shot: resistance 4.1, time 29s, yield 2.05 → good zone.
   goodShotCount → 2. calibrated = true.
   Diagnosis: OnTarget. Confidence: calibrated.
