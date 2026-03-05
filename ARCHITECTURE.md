# Flipper Zero — AI Weather Station Protocol Classifier
*"Point it at any weather station, it figures out the protocol itself"*

---

## The Concept

Flipper Zero as a self-contained weather station protocol identifier and decoder. Known protocols decode instantly from a full rtl_433 corpus loaded off SD. Unknown protocols get classified entirely on device using signal pattern analysis. Claude API is a specialist consultant for the genuinely novel 5-10% that nobody has ever documented.

The STM32WB55 has plenty of CPU. The SD card has gigs of storage. Flash size is not a constraint — stop thinking like it is.

---

## Storage Architecture — Think Big

```
Flash (1MB): Boot + core app only
             Minimal footprint
             Just enough to init SD and start

SD Card (32GB+): Everything else — no limits

├── Protocol Library (~5-10MB)
│   Full rtl_433 corpus — ALL 200+ devices
│   Not a curated subset. All of them.
│   Loaded at startup, cached in RAM as needed
│
├── Capture Log (grows forever)
│   Every packet ever seen, raw + decoded
│   Timestamp, RSSI, GPS if available
│   Years of data at weather station rates
│   Auto-assembles training dataset
│
├── Classifier Model (~2-50MB)
│   Quantized ONNX model for pattern classification
│   TinyML on STM32 — loads to RAM on demand
│   Retrained from SD dataset as it grows
│
├── Unknown Protocol Workspace
│   Staging area for unclassified captures
│   Accumulates until confident enough to name
│   Submitted to community repo when validated
│
└── Historical Weather Data
    Full time-series for every station seen
    Query it later, export it, graph it
    It's just a database on a card
```

**The mental model shift:**

Flash is for the kernel. SD is for everything interesting. Program size was never the constraint — now act like it.

---

## Why The Flipper Is Perfect For This

```
STM32WB55RG MCU:
├── Cortex-M4 @ 64MHz   ← classification + app logic
├── Cortex-M0+ @ 32MHz  ← radio handling
├── 256KB RAM           ← runtime working memory
├── 1MB Flash           ← boot + core only
└── Hardware FPU        ← fast float for signal math

RF Hardware:
├── CC1101:   Sub-GHz RF — 433/868/915MHz ✓
├── NRF24L01: 2.4GHz for newer devices ✓
└── Both active simultaneously possible

Everything Else:
├── MicroSD:  32GB+ = essentially unlimited ✓
├── 128x64:   Clean display ✓
├── Battery:  Portable ✓
└── FCC certified ✓
```

---

## Architecture

```
Layer 1: Packet Capture (CC1101 / NRF24)
├── Scan 433, 868, 915MHz
├── Accumulate 10-20 captures from same transmitter
├── Raw bitstreams → RAM analysis buffer
└── Everything logged to SD immediately

Layer 2: Full rtl_433 Corpus Match (SD → RAM cache)
├── Load full 200+ protocol library from SD at boot
├── Sync word + packet length + modulation fingerprint
├── Match against entire corpus, not just a curated 20
├── Confidence > 90% → instant decode, display readings
└── Covers the vast majority of consumer hardware

Layer 3: On-Device Pattern Classifier (STM32)
├── For protocols not in rtl_433 corpus
├── Bit stability analysis
├── Field boundary detection
├── Encoding identification (BCD, Gray, signed, offset)
├── Checksum brute force (CRC8, XOR, Sum8, CRC16)
├── Physics validation
└── TinyML model on SD for ambiguous cases

Layer 4: Claude API Escalation (WiFi, rare — ~5-10%)
├── Genuinely novel protocols nobody has documented
├── Ambiguous field boundaries after pattern analysis
├── Unknown scaling factors
└── Sends pre-analyzed structure, not raw bits

Community Layer
├── Validated new definitions → GitHub PR
├── SD capture log → training dataset contribution
└── OTA library updates pull new rtl_433 additions
```

---

## The rtl_433 Corpus — Your Starting Point

Don't reinvent this. rtl_433 has 200+ documented decoders.
Load all of them from SD at startup.

```
Full corpus download from rtl_433 repo:
├── Convert C decoders → JSON protocol definitions
│   (one-time conversion script, run on desktop)
├── Store all definitions on SD:
│   /weather_classifier/protocols/rtl433/
│   ~5-10MB for the entire library
└── Load into RAM cache on boot
    Index by sync word + frequency + modulation
    Lookup is O(1) hash table

Coverage:
├── Acurite (5-in-1, 6-in-1, towers, lightning)
├── Ambient Weather / Fine Offset / Ecowitt family
├── Oregon Scientific (v1, v2, v3)
├── La Crosse (multiple generations)
├── Davis Instruments
├── Bresser (5-in-1, 6-in-1, 7-in-1)
├── Froggit / Pantech / rebranded Fine Offset
├── Hideki / Cresta
├── Springfield / AcuRite rebrands
└── 170+ more
```

**Practically speaking:** if someone bought their weather station at Costco, Home Depot, or Amazon in the last 15 years, it's probably already in rtl_433. The pattern classifier is for the truly exotic stuff.

---

## On-Device Pattern Classifier — For Novel Protocols

Only runs when the full rtl_433 corpus doesn't match.

### Bit Stability Analysis

```c
typedef struct {
    uint8_t fixed;       // same across all captures
    uint8_t value;       // value if fixed
    uint8_t ones_count;  // variance indicator
    float   smoothness;  // inter-capture delta smoothness
} BitProfile;

// Fixed bits   → preamble, sync, device ID, flags
// Variable     → sensor data
// Smooth delta → slowly changing sensor (temperature)
// Random delta → checksum, counter, rolling code
```

### Structural Detection

```c
// Sync word candidates: runs of fixed bits in consistent position
// Preamble: 0xAA / 0x55 alternating pattern
// Checksum: fixed position, high entropy, fails CRC brute force last

// Known sync word table still useful as fast-path:
const SyncPattern COMMON_SYNCS[] = {
    {0x2DD4, "Fine Offset",  915000000},
    {0xD391, "Oregon Sci",   433920000},
    {0xAA2D, "Acurite",      433920000},
    {0x55AA, "La Crosse",    433920000},
    {0xFEFE, "Davis",        915000000},
};
// But this is a hint layer, not the only layer
// Full corpus match happens first
```

### Checksum Brute Force

```c
// Try every common algorithm at every packet position
// All captures must agree for a match to count

for(int pos = 4; pos < packet_bytes; pos++) {
    if(verify_crc8_all(captures, n, pos))  → CRC8 at pos
    if(verify_xor_all(captures, n, pos))   → XOR at pos
    if(verify_sum8_all(captures, n, pos))  → Sum8 at pos
    if(verify_crc16_all(captures, n, pos)) → CRC16 at pos
}
```

### Physics Validation

```c
// Sanity check decoded values against physical reality
// Wrong encoding or wrong field boundaries → implausible values

const PhysicsConstraint CONSTRAINTS[] = {
    {"temperature_c",  -50,   60},
    {"humidity_pct",     0,  100},
    {"wind_speed_ms",    0,   75},
    {"pressure_hpa",   870, 1085},
    {"rain_mm",          0,  500},
};
```

### TinyML Layer (on SD, loaded to RAM)

```
For genuinely ambiguous cases:
├── Small CNN trained on bitstream patterns
├── Input: BitProfile array (stability + entropy features)
├── Output: protocol family probabilities
├── Model lives on SD (~2-10MB quantized)
├── Loaded to RAM when needed
├── Retrained as capture dataset grows
└── Built with Edge Impulse or TFLite Micro
```

---

## Capture Logging — The Dataset That Builds Itself

Every packet goes to SD. Always. No exceptions.

```
/weather_classifier/captures/
├── YYYY-MM-DD/
│   ├── raw/
│   │   └── {timestamp}_{freq}_{protocol}.bin
│   └── decoded/
│       └── {timestamp}_{protocol}.json

Decoded JSON example:
{
  "timestamp": "2026-03-05T14:23:11Z",
  "frequency_hz": 915000000,
  "modulation": "OOK_PWM",
  "rssi_dbm": -67,
  "protocol": "Fine Offset WS-2902",
  "confidence": 0.97,
  "readings": {
    "temperature_c": 22.4,
    "humidity_pct": 54,
    "wind_speed_ms": 3.6,
    "wind_dir_deg": 315,
    "rain_mm": 0.5
  },
  "raw_hex": "AA2DD4..."
}
```

**What this gives you over time:**

```
1 week:    Diurnal temperature patterns visible
1 month:   Storm events captured, wind rose data
1 year:    Seasonal patterns, frost date tracking,
           rainfall totals, microclimate data
           that no weather service has for your yard

Historical query via Claude (WiFi mode):
"How many hours above 90°F last July?"
"What was the wettest week in 2026?"
"Compare this winter to last winter"
```

The Flipper becomes a long-term personal weather station data logger as a side effect of being a protocol classifier.

---

## Memory Budget

```
STM32WB55: 256KB RAM

At runtime:
├── Flipper OS:               ~100KB
├── Active capture buffer:    ~320 bytes (20×16)
├── BitProfile analysis:      ~512 bytes
├── Protocol cache (hot):     ~20KB  (10 recently seen)
├── TinyML model (if loaded): ~50KB  (quantized)
├── Working buffers:          ~10KB
└── Available headroom:       ~75KB  ✓ comfortable

SD Card: as big as you put in
└── Treat it as infinite for this project
```

---

## Validation Test Case — Ambient Weather WS-2902

Your station. Ground truth for verifying everything works.

```
Protocol:     Fine Offset / Ecowitt family
Frequency:    915MHz (US version)
Modulation:   OOK PWM
In rtl_433:   "Fine Offset WH65B" — near identical

Packet structure:
├── Preamble:    0xAA repeated
├── Sync:        0x2DD4
├── Length:      80 bits
├── Device ID:   8 bits
├── Flags:       4 bits
├── Temp:        12 bits  (offset 400, scale 0.1°C)
├── Humidity:    8 bits
├── Wind speed:  8 bits  (scale 0.34 m/s)
├── Wind gust:   8 bits  (scale 0.34 m/s)
├── Wind dir:    9 bits  (0-359°)
├── Rain:        16 bits (scale 0.1mm)
└── CRC8:        8 bits

⚠️ Verify your Flipper antenna is tuned for 915MHz.
   Some ship with 433MHz antenna — still works nearby,
   weaker at range.
```

**Validation sequence:**

```
Step 1: Boot with full rtl_433 corpus on SD
        WS-2902 matches Fine Offset definition
        instantly on first packet ✓

Step 2: Remove Fine Offset from corpus
        Pattern classifier runs
        Reconstructs protocol from scratch
        Readings match Step 1 ✓

Step 3: Test on unknown neighbor's station
        Full pipeline runs
        New definition generated and validated
        Submit to community repo
```

---

## Flipper App UI

### Known Protocol — Live Readings
```
┌──────────────────────────┐
│ 🌡 WEATHER STATION       │
│ Fine Offset / WS-2902    │
│ ──────────────────────── │
│ Temp:     72.4°F         │
│ Humidity:  54%           │
│ Wind:    8.2 mph NW      │
│ Rain:     0.02"          │
│ ──────────────────────── │
│ RSSI: -67dBm  ID: 0x4A2 │
│ [SAVE] [SCAN] [EXPORT]   │
└──────────────────────────┘
```

### Pattern Classifier Running
```
┌──────────────────────────┐
│ ⚡ ANALYZING...          │
│ 915MHz OOK               │
│ ──────────────────────── │
│ rtl_433 corpus: no match │
│ Captures:  17/20         │
│ Sync:      0x2DD4 ✓      │
│ Checksum:  CRC8 ✓        │
│ Fields:    resolving...  │
│ ──────────────────────── │
│ Confidence: 88%          │
│ [CONFIRM] [REJECT]       │
└──────────────────────────┘
```

### Unverified — Needs Confirmation
```
┌──────────────────────────┐
│ ⚠️  UNVERIFIED PROTOCOL  │
│ Confidence: 74%          │
│ ──────────────────────── │
│ Temp:     71.8°F  ?      │
│ Humidity:  51%    ?      │
│ Wind:    7.9 mph  ?      │
│ ──────────────────────── │
│ Do readings look right?  │
│ ──────────────────────── │
│ [YES ✓] [NO ✗] [CLOUD?] │
└──────────────────────────┘
```

### Scan Mode
```
┌──────────────────────────┐
│ 📡 SCANNING...           │
│ 433 / 915 MHz            │
│ ──────────────────────── │
│ Signals detected: 4      │
│ ──────────────────────── │
│ ✓ Fine Offset   -64dBm  │
│ ✓ Oregon v3     -71dBm  │
│ ✓ Acurite 6in1  -79dBm  │
│ ? Unknown       -88dBm  │
│ ──────────────────────── │
│ [SELECT] [CLASSIFY ALL]  │
└──────────────────────────┘
```

### Historical Query Mode (WiFi)
```
┌──────────────────────────┐
│ 🔍 QUERY HISTORY         │
│ ──────────────────────── │
│ > Hottest day this month │
│                          │
│ Mar 2: 94.2°F at 3:40pm  │
│ Mar 7: 91.8°F at 2:15pm  │
│ Mar 12: 89.4°F at 4:00pm │
│ ──────────────────────── │
│ Powered by Claude API    │
│ [NEW QUERY] [EXPORT]     │
└──────────────────────────┘
```

---

## Protocol Definition Format

```json
{
  "name": "Ambient Weather WS-2902",
  "family": "Fine Offset",
  "frequency_mhz": 915.0,
  "modulation": "OOK_PWM",
  "bitrate_bps": 17241,
  "packet_length_bits": 80,
  "sync_word": "0x2DD4",
  "checksum": "CRC8",
  "fields": {
    "device_id":     {"bits": "0:7",   "transform": "raw"},
    "temperature_c": {"bits": "16:27", "transform": "(raw - 400) * 0.1"},
    "humidity_pct":  {"bits": "28:35", "transform": "raw"},
    "wind_speed_ms": {"bits": "36:43", "transform": "raw * 0.34"},
    "wind_gust_ms":  {"bits": "44:51", "transform": "raw * 0.34"},
    "wind_dir_deg":  {"bits": "52:60", "transform": "raw"},
    "rain_mm":       {"bits": "61:76", "transform": "raw * 0.1"}
  },
  "validated": true,
  "source": "rtl_433",
  "rtl433_compatible": true
}
```

---

## SD Card File Structure

```
/weather_classifier/
│
├── protocols/
│   ├── rtl433/              ← full corpus, all 200+ devices
│   │   ├── fine_offset_ws2902.json
│   │   ├── acurite_5in1.json
│   │   ├── oregon_v3.json
│   │   └── ... (200+ files, ~5-10MB total)
│   ├── community/           ← user-contributed new protocols
│   └── pending/             ← classified but not yet validated
│
├── captures/
│   ├── 2026-03-05/
│   │   ├── raw/
│   │   └── decoded/
│   └── ... (grows forever, it's fine)
│
├── models/
│   └── protocol_classifier.onnx   ← TinyML model (~5-50MB)
│       updated as training data grows
│
└── weather_history/
    └── fine_offset_0x4A2/   ← per-station time series
        └── 2026-03.jsonl    ← one reading per line
```

---

## Repo Structure for Claude Code Sessions

```
flipper-weather-ai/
├── README.md
├── ARCHITECTURE.md
├── docs/
│   ├── cc1101_registers.md      ← from datasheet, verbatim
│   │                               prevents hallucinated values
│   ├── fine_offset_protocol.md  ← WS-2902 ground truth
│   ├── flipper_subghz_api.md    ← SDK patterns + examples
│   └── rtl433_reference/
│       └── fineoffset.c         ← battle-tested decode logic
│
├── tools/
│   └── rtl433_to_json.py        ← converts rtl_433 C decoders
│                                   to JSON protocol definitions
│                                   run once on desktop
│
├── flipper_app/
│   ├── weather_classifier.c
│   ├── weather_classifier.h
│   ├── classifier/
│   │   ├── corpus_loader.c      ← SD → RAM cache
│   │   ├── bit_stability.c
│   │   ├── sync_detect.c
│   │   ├── field_boundary.c
│   │   ├── encoding_detect.c
│   │   ├── checksum_brute.c
│   │   └── tinyml_infer.c       ← ONNX model inference
│   ├── storage/
│   │   ├── capture_log.c        ← everything to SD
│   │   └── history_db.c         ← time series queries
│   └── application.fam
│
├── host_bridge/                 ← Claude API for hard cases
│   ├── bridge.py
│   ├── classifier.py
│   └── requirements.txt
│
└── model_training/
    ├── build_dataset.py         ← from SD capture logs
    ├── train.py                 ← Edge Impulse or TFLite
    └── quantize_export.py       ← → ONNX for Flipper
```

**The docs folder is the most important thing in the repo.**
Claude Code with `cc1101_registers.md` in context generates
correct register values. Without it — plausible but wrong,
fails silently on real hardware. Put the datasheet excerpts in.

---

## Weekend Build Plan

### Saturday Morning — Foundation
- [ ] Repo setup, all reference docs assembled
- [ ] `rtl433_to_json.py` conversion tool — populate SD corpus
- [ ] Flipper app skeleton + application.fam
- [ ] CC1101 init for 915MHz OOK
- [ ] Raw packet capture to buffer + SD log

### Saturday Afternoon — Corpus Match
- [ ] SD protocol corpus loader into RAM cache
- [ ] Hash-based lookup (sync word + freq + modulation)
- [ ] WS-2902 matched from rtl_433 corpus ✓
- [ ] Live readings displaying correctly

### Saturday Evening — Pattern Classifier
- [ ] Bit stability analysis module
- [ ] Field boundary detection
- [ ] Encoding detection + checksum brute force
- [ ] Physics validation
- [ ] End-to-end: remove WS-2902 from corpus → reclassify ✓

### Sunday Morning — Storage + History
- [ ] Full capture logging to SD working
- [ ] Protocol definition save/load from SD
- [ ] Historical query via Claude API (WiFi mode)
- [ ] All UI states polished

### Sunday Afternoon — Show Off
- [ ] Scan mode across 433 + 915MHz simultaneously
- [ ] Test against unknown station if available
- [ ] Demo video recorded
- [ ] Pushed to GitHub

---

## The Community Flywheel

```
User runs app near unknown station
    ↓
Full rtl_433 corpus: no match
    ↓
On-device pattern classifier runs
    ↓
Confidence > 90%: display readings
User confirms ✓
    ↓
"Submit to community repo?" → Yes
    ↓
New definition → GitHub PR
Community validates
    ↓
Next rtl_433 corpus update includes it
    ↓
All Flippers OTA-pull the update
    ↓
repeat
```

---

## What Makes This Novel

| Existing Tools | This Project |
|---|---|
| rtl_433: PC only, CLI | Self-contained on Flipper |
| Flipper weather apps: ~5 hardcoded protocols | Full 200+ device corpus from SD |
| Manual reverse engineering | On-device pattern classification |
| No capture history | Permanent SD time-series logging |
| No natural language queries | Claude API historical analysis |
| Siloed protocol knowledge | Community contribution flywheel |
| Requires RF expertise | Point at station, press button |

---

## Future Ideas (Post-Weekend)

- [ ] NRF24L01 2.4GHz support for newer WiFi-adjacent stations
- [ ] TinyML model trained on accumulated SD dataset
- [ ] GPS tagging of captures (with GPIO module)
- [ ] "Wardriving" mode — log all weather stations while walking
- [ ] Integration with universal weather console project
- [ ] BLE companion app for phone display
- [ ] Automatic Weather Underground / CWOP submission
- [ ] Personal weather pattern analysis via Claude API
- [ ] Cross-reference FCC ID database for manufacturer hints

---

## Hardware Needed
- Flipper Zero (you have one)
- Ambient Weather WS-2902 (you have one — ground truth)
- MicroSD card (32GB+ — treat storage as free)
- Dev machine for firmware build (Docker, easy setup)

## Flipper Dev Environment

```
Firmware repo:
github.com/flipperdevices/flipperzero-firmware

Docker build — no painful ARM toolchain:
docker run --rm -v .:/ext \
  flipperdevices/flipperzero-toolchain \
  fbt

Your app:
applications/external/weather_classifier/

Study these before writing any code:
lib/subghz/protocols/fine_offset.c   ← your target
lib/subghz/protocols/acurite.c       ← similar pattern
lib/subghz/devices/cc1101.c          ← chip driver
```

---

*Started: March 2026 — vibe coded for the hacker homies*
*Architecture: full rtl_433 corpus on SD, on-device pattern*
*classification for unknowns, Claude API for the exotic 5%*
