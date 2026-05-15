# Adaptive Brew — Firmware Architecture Reference

## Overview

This document is the single source of truth for implementing adaptive brew and grind
prediction directly in the gaggimate ESP32-S3 firmware. No external services, no cloud,
no LLM — everything runs on the device using classical signal processing and online
learning algorithms.

---

## Hardware Target

- **MCU**: ESP32-S3 @ 240MHz dual-core
- **PSRAM**: 8MB OPI (all board variants)
- **Flash**: 8–16MB QIO
- **Framework**: Arduino via PlatformIO (espressif32@6.12.0)
- **C++ standard**: C++17

---

## Codebase Map (Key Files)

```
src/display/
├── core/
│   ├── predictive.h              ← existing: VolumetricRateCalculator (linear regression)
│   ├── Settings.h / Settings.cpp ← NVS persistence via Preferences library
│   ├── constants.h               ← PREDICTIVE_TIME, PROGRESS_INTERVAL, etc.
│   ├── process/
│   │   ├── BrewProcess.h         ← phase loop, pump control, easing functions
│   │   └── GrindProcess.h        ← grind control
│   └── adaptive/                 ← NEW: all adaptive brew modules go here
│       ├── BeanGrindRecord.h
│       ├── BeanMetadata.h
│       ├── BeanKnowledgeBase.h
│       ├── ChannelingDetector.h
│       ├── FreshnessTracker.h
│       ├── GrindLearner.h
│       ├── RLSRegressor.h
│       └── ShotMetricsExtractor.h
├── plugins/
│   ├── ShotHistoryPlugin.h/cpp   ← shot recording; trigger point for post-shot analysis
│   ├── WebUIPlugin.h/cpp         ← WebSocket API; broadcasts grind suggestion
│   └── MQTTPlugin.h/cpp          ← publishes brew state events
└── models/
    ├── shot_log_format.h         ← binary .slog format; ShotLogSample has pr, cp, fl, v fields
    └── profile.h                 ← Profile struct with UUID id field
```

---

## Shot Data Available at Analysis Time

### ShotLogSample fields (26 bytes, every 250ms)
```cpp
uint16_t t;    // sample index (0.25s ticks)
uint16_t tt;   // target temp × 10
uint16_t ct;   // current temp × 10
uint16_t tp;   // target pressure × 10 (bar)
uint16_t cp;   // current pressure × 10 (bar)
int16_t  fl;   // pump flow × 100 (ml/s)
int16_t  tf;   // target flow × 100 (ml/s)
int16_t  pf;   // puck flow × 100 (ml/s)
int16_t  vf;   // BT scale flow × 100 (ml/s)
uint16_t v;    // BT weight × 10 (g)
uint16_t ev;   // estimated weight × 10 (g)
uint16_t pr;   // puck resistance × 100
uint16_t si;   // system info flags
```

### ShotLogHeader (512 bytes)
```cpp
uint32_t durationMs;          // total shot duration
uint32_t startEpoch;          // unix timestamp
uint16_t finalWeight;         // g × 10 (from BT scale or estimate)
char     profileId[32];       // UUID of profile used
char     profileName[48];
PhaseTransition phaseTransitions[12]; // up to 12 phase changes
uint8_t  phaseTransitionCount;
```

### Shot notes JSON (per-shot, /h/{id}.json)
```json
{
  "beanType": "Ethiopian",
  "doseIn": 18.0,
  "doseOut": 36.2,
  "ratio": 2.01,
  "grindSetting": "7",
  "balanceTaste": "sour",
  "rating": 3,
  "notes": ""
}
```

---

## Module Specifications

### 1. BeanMetadata.h

Enums and struct for bean classification. Stored inside BeanGrindRecord.

```cpp
enum class BeanProcess  : uint8_t { Unknown, Washed, Natural, Honey, Anaerobic };
enum class RoastGrade   : uint8_t { Unknown, Light, MedLight, Medium, MedDark, Dark };
enum class OriginGroup  : uint8_t { Unknown, Africa, CentralAmerica, SouthAmerica, Asia };
enum class TasteBalance : uint8_t { None, Sour, Balanced, Bitter, SourBitter };

struct BeanMetadata {
    char        origin[20];      // free text, e.g. "Ethiopian"
    BeanProcess process;
    RoastGrade  roast;
    OriginGroup originGroup;
    uint8_t     daysSinceRoast;  // 0 = unknown
};
```

### 2. BeanGrindRecord.h

Per-profile learned state. 8 slots in NVS (~90 bytes × 8 = 720 bytes).

```cpp
enum class GrindDirection : uint8_t { None, Finer, Coarser, CheckPrep, OnTarget };
enum class GrindMagnitude : uint8_t { None, Slight, Moderate, Significant };

struct BeanGrindRecord {
    char           profileId[33];        // UUID (32 chars + null)
    BeanMetadata   meta;                 // bean type info
    float          targetResistanceEMA;  // resistance from confirmed good shots
    float          resistanceEMA;        // running EMA of all shots
    float          timeEMA;              // running EMA of extraction time (s)
    float          yieldEMA;             // running EMA of yield ratio
    uint8_t        shotCount;            // total shots (caps at 255)
    uint8_t        goodShotCount;        // shots within the good zone
    GrindDirection lastDirection;
    GrindMagnitude lastMagnitude;
    bool           calibrated;           // true once >= 2 good shots observed
    bool           active;               // slot is in use
};
```

### 3. ShotMetrics (extracted from sample buffer)

```cpp
struct ShotMetrics {
    float extractionTime;   // seconds (durationMs / 1000)
    float yieldRatio;       // doseOut / doseIn (0 if dose unknown)
    float avgResistance;    // mean(pr / 100.0f)
    float resistanceCV;     // std(pr) / mean(pr) — channeling indicator
    float pressureRatio;    // mean(cp) / mean(tp) — pressure achievement
    float earlyFlowAvg;     // mean fl in first 8s (ml/s)
    bool  hasScale;         // BT scale was connected
};
```

### 4. BeanKnowledgeBase.h

Compile-time lookup matrix. Zero RAM cost (stored in flash as const).

#### Knowledge Matrix Layout: [process][roast] → {resistanceOffset, tempOffset, targetTime, targetYield}

```
                  Unknown  Light   MedLight  Medium  MedDark  Dark
Washed:           0        +0.5°   +0.3      0       -0.2     -0.5
                           +2°C    +1°C      +0°C    -0.5°C   -1°C
                           28s     27s       26s     25s      24s
                           1.95    1.97      2.0     2.03     2.1

Natural:          0        +0.3    +0.1      -0.1    -0.3     -0.6
                           +1°C    +0.5°C    -1°C    -1.5°C   -2°C
                           30s     28s       27s     25s      22s
                           1.9     1.95      2.0     2.1      2.2

Honey:            0        +0.2    +0.1      0       -0.2     -0.4
                           +1°C    0°C       0°C     -1°C     -1°C
                           28s     27s       26s     24s      23s
                           1.95    1.97      2.0     2.05     2.1

Anaerobic:        0        -0.2    -0.3      -0.4    -0.5     -0.6
                           -1°C    -2°C      -2°C    -2°C     -3°C
                           22s     21s       20s     19s      18s
                           2.1     2.1       2.15    2.2      2.3
```

#### Freshness Resistance Offsets
```
< 7 days:   -0.3  (CO₂ degassing inflates apparent resistance → correct coarser)
7–21 days:   0.0  (optimal zone)
21–35 days: +0.1  (beginning to stale → go slightly finer)
> 35 days:  +0.2  (staling → go finer + issue warning)
```

#### EMA Alpha per freshness
```
< 7 days:   0.5   (high variance, adapt fast)
7–21 days:  0.3   (normal)
21–35 days: 0.2   (conservative)
> 35 days:  0.15  (slow convergence)
```

### 5. RLSRegressor.h

Recursive Least Squares online learner. One instance per output (pressure, flow, yield).

- Input features (6): yieldError, timeError, avgResistance, resistanceCV, pressureCV, phaseExitType
- Forgetting factor λ = 0.97
- Initial covariance P = 100·I
- State: θ[6] + P[6][6] = 42 floats = 168 bytes per regressor
- 3 regressors total = 504 bytes in NVS under keys "rls_p", "rls_f", "rls_y"

Update equations:
```
Px   = P · x
denom = λ + x^T · Px
K     = Px / denom
θ     = θ + K · (y_actual - x^T · θ)
P     = (P - K · x^T · P) / λ
```

### 6. ChannelingDetector.h

Real-time, runs every 250ms during brew.

- Maintains deque of (timestamp, puck_resistance) for last 1000ms
- Channeling condition: peak_in_window > 0.5 AND (peak - current) / peak > 0.30
- Action: signal BrewProcess to reduce pump power to 85%
- Reset on shot start

### 7. GrindLearner.h

Main orchestrator. Owns 8 BeanGrindRecord slots + 3 RLSRegressors.

#### Good shot zone
```
yieldRatio   1.80 – 2.45
extractionTime  22 – 36 s
resistanceCV  < 0.30
```

#### Diagnosis logic (in priority order)
```
1. resistanceCV > 0.30 AND (underYield OR timeSlow) → CheckPrep
2. yieldRatio < 1.75 OR timeFast (< 21s)            → Finer
3. yieldRatio > 2.50 OR timeSlow (> 38s)            → Coarser
4. taste == Sour                                     → Finer  (overrides telemetry)
5. taste == Bitter                                   → Coarser (overrides telemetry)
6. taste == SourBitter                               → CheckPrep
7. otherwise                                         → OnTarget
```

#### Magnitude from deviation score
```
dev = max(|yieldError| / 0.5, |timeError| / 7.0)
dev < 0.5  → Slight
dev < 1.5  → Moderate
dev >= 1.5 → Significant
```

#### Rating fast-path
```
rating >= 4 → lock this shot's resistance as target immediately (goodShotCount++)
rating <= 2 → set EMA alpha = 0.5 for next shot (learn fast from bad shots)
```

#### Similarity matching for first-shot estimate
When a new bean is registered (shotCount == 0):
1. Scan all active BeanGrindRecord slots
2. Score similarity: +3 if process matches, +2 if roast matches, +1 if originGroup matches
3. Best match with score >= 3: blend 40% knowledge matrix + 60% past record
4. No match: use knowledge matrix only

---

## Integration Points

### ShotHistoryPlugin.cpp — trigger analysis

After `header.durationMs` and `header.finalWeight` are patched (shot finalized),
and before the sample buffer is freed:

```cpp
// Get dose from notes if available, otherwise use settings default
float doseIn = settings->getTargetGrindVolume();
// ... load notes JSON for doseIn override if /h/{id}.json exists

ShotMetrics metrics = ShotMetricsExtractor::extract(
    sampleBuffer.data(),
    sampleBuffer.size(),
    doseIn,
    header.finalWeight / 10.0f
);
grindLearner->processShotData(metrics, String(header.profileId));
```

### WebUIPlugin.cpp — broadcast suggestion

Add to the `evt:status` JSON object (or a new `evt:grind-suggestion` after shot ends):

```cpp
const BeanGrindRecord* rec = grindLearner->getRecord(selectedProfile.id);
if (rec && rec->shotCount > 0) {
    JsonObject g = doc.createNestedObject("grind");
    g["dir"]   = (uint8_t)rec->lastDirection;   // 0-4
    g["mag"]   = (uint8_t)rec->lastMagnitude;   // 0-3
    g["shots"] = rec->shotCount;
    g["cal"]   = rec->calibrated;
    g["ryield"]= rec->yieldEMA;
    g["rtime"] = rec->timeEMA;
}
```

### New WebSocket request — save bean metadata

Request type: `req:notes:bean`
```json
{
  "tp": "req:notes:bean",
  "rid": "abc",
  "shotId": "000047",
  "origin": "Ethiopian",
  "process": 1,
  "roast": 1,
  "days": 12,
  "taste": 1,
  "rating": 3
}
```
Handler in WebUIPlugin updates the shot .json notes AND calls
`grindLearner->updateWithFeedback(shotId, taste, rating)`.

---

## Memory Budget

| Component              | RAM (runtime) | NVS flash |
|------------------------|---------------|-----------|
| BeanGrindRecord[8]     | 720 bytes     | 720 bytes |
| RLSRegressor × 3       | 504 bytes     | 504 bytes |
| ChannelingDetector     | ~200 bytes    | —         |
| ShotMetrics (temp)     | ~32 bytes     | —         |
| FreshnessTracker       | ~20 bytes     | ~20 bytes |
| BeanKnowledgeBase      | 0 (flash ROM) | —         |
| **Total**              | **~1.5 KB**   | **~1.2 KB**|

No new libraries required. No PSRAM dependency.

---

## Implementation Priority Order

1. `ChannelingDetector.h` — standalone, testable immediately, zero dependencies
2. `ShotMetricsExtractor.h` — reads existing sample buffer, no persistence needed
3. `BeanGrindRecord.h` + `BeanMetadata.h` — data structures only
4. `GrindLearner.h` (EMA-only first, RLS optional later) — wire into ShotHistoryPlugin
5. `BeanKnowledgeBase.h` — compile-time matrix for first-shot estimates
6. `RLSRegressor.h` — optional enhancement, adds shot-to-shot weight learning
7. `FreshnessTracker.h` — optional, tracks bean age over days
8. WebUI `grind` payload + `req:notes:bean` handler in WebUIPlugin
9. Settings: `adaptiveBrew` bool toggle, `activeBeanSlot` int

---

## Key Design Constraints

- Never mutate the original profile — apply deltas as runtime overlays only
- Maximum delta limits: ±0.5 bar pressure, ±0.4 ml/s flow, ±3g yield
- Maximum per-shot step: ±0.12 units (prevents overcorrection)
- Discard shots shorter than 7.5s (flush / abort)
- All NVS writes are deferred through a dirty flag — never write in hot path
- The ChannelingDetector runs on Core 1 (brew loop) — keep it allocation-free
