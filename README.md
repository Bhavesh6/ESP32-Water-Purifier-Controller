# 💧 Smart RO Water Purifier Controller

> ESP32-based fully autonomous RO water purifier controller with TDS monitoring, dual filter life tracking, service mode, and smart display — built from scratch with no cloud dependency.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Hardware](#hardware)
  - [Pin Map](#pin-map)
  - [Relay Wiring](#relay-wiring)
  - [LED Driver Circuits](#led-driver-circuits)
  - [TDS Sensor](#tds-sensor)
  - [Display](#display)
- [How It Works](#how-it-works)
  - [Purification Cycle](#purification-cycle)
  - [Dispensing](#dispensing)
  - [TDS Monitoring](#tds-monitoring)
  - [Filter Life Tracking](#filter-life-tracking)
  - [Display Logic](#display-logic)
  - [Service Mode (Filter Reset)](#service-mode-filter-reset)
  - [Filter Life Check](#filter-life-check)
  - [Auto Sleep](#auto-sleep)
  - [EEPROM Persistence](#eeprom-persistence)
- [Display Messages Reference](#display-messages-reference)
- [Current Known Bugs](#current-known-bugs)
- [Known Real-World Issues](#known-real-world-issues)
- [Planned Improvements](#planned-improvements)
- [Libraries Required](#libraries-required)
- [File Structure](#file-structure)

---

## Overview

This project replaces the stock controller board of a residential RO water purifier with a custom ESP32-based system. The goal is complete onboard autonomous control — no cloud, no Wi-Fi required — with real-time TDS output monitoring, dual filter life estimation, a dot-matrix status display, and a technician service mode for filter resets.

The system controls the full purification cycle (inlet solenoid → pump → UV lamp), monitors output water quality via a TDS sensor, tracks filter degradation over time, and provides visual feedback via a MAX7219 dot-matrix display and multiple indicator LEDs.

**Key design principles:**
- Fully standalone — no internet connection required
- Hardware-first safety — all relays default OFF on power loss
- Modular filter life tracking — RO membrane and pre/post filters tracked separately
- Non-volatile memory — filter life survives power cuts via EEPROM
- Minimal UI — everything communicated through a single 64×8 dot-matrix display and button interactions

---

## Hardware

### Pin Map

| GPIO | Component | Direction | Logic | Notes |
|------|-----------|-----------|-------|-------|
| 4 | Dispense Button | INPUT_PULLUP | LOW = pressed | Short press = dispense. Hold 5s = filter life check |
| 35 | Float Sensor | INPUT (no pullup) | HIGH = tank full | ADC-capable input-only pin. Needs external 10kΩ pull-down |
| 32 | Water Presence Sensor | INPUT_PULLUP | LOW = water present | Detects input supply water. Active-LOW sensor |
| 33 | Filter Reset Button | INPUT_PULLUP | LOW = pressed | Service mode entry. Long hold = enter, short = toggle, long inside = confirm reset |
| 26 | Dispense Solenoid (R3) | OUTPUT | Active-LOW relay | Opens output dispense valve |
| 27 | Inlet Solenoid (R2) | OUTPUT | Active-LOW relay | Opens input water supply valve |
| 14 | Pump Relay (R1) | OUTPUT | Active-LOW relay | Controls RO pump motor |
| 25 | UV Relay (R4) | OUTPUT | Active-LOW relay | Controls UV sterilisation lamp |
| 16 | LED Strip | OUTPUT | Inverted (LOW = ON) | Driven by 2N3904 + IRFZ44N MOSFET |
| 17 | Nozzle LED | OUTPUT | Normal (HIGH = ON) | Driven by 2N7000 MOSFET |
| 19 | Button LED | OUTPUT | Normal (HIGH = ON) | Driven by 2N3904 + IRF9540N MOSFET |
| 23 | MAX7219 DIN | SPI MOSI | — | Display data |
| 18 | MAX7219 CLK | SPI SCK | — | Display clock |
| 5 | MAX7219 CS | SPI CS | — | Display chip select |
| 36 | TDS Sensor | ADC1_CH0 | Analog | Input-only pin. No internal pullup available |

### Relay Wiring

All four relays (R1–R4) are active-LOW. This means:
- `HIGH` on the GPIO → relay coil OFF → load is disconnected (safe default)
- `LOW` on the GPIO → relay coil ON → load is connected

The ESP32 drives relay coils through the relay module's optocoupler inputs directly. All relays are initialised `HIGH` in `setup()` so a power-on or reset state is always safe (all loads off).

**Relay assignment:**

| Relay | Label | Controls | Activation Order |
|-------|-------|----------|-----------------|
| R1 | PUMP_RELAY | RO pump motor | 3rd (2s after inlet) |
| R2 | INLET_SOLENOID | Input water valve | 1st |
| R3 | DISPENSE_SOLENOID | Output dispense valve | On button press only |
| R4 | UV_RELAY | UV sterilisation lamp | 2nd (1s after pump) |

### LED Driver Circuits

| LED | Driver | Logic | Reason |
|-----|--------|-------|--------|
| LED Strip | 2N3904 (NPN) + IRFZ44N (N-ch MOSFET) | Inverted — LOW = ON | NPN inverts signal to drive N-ch gate. Strip draws high current |
| Nozzle LED | 2N7000 (N-ch MOSFET) | Normal — HIGH = ON | Direct GPIO → gate drive, low current LED |
| Button LED | 2N3904 (NPN) + IRF9540N (P-ch MOSFET) | Normal — HIGH = ON | NPN drives gate of P-ch for higher current capability |

### TDS Sensor

- **Type:** Keyestudio TDS Meter V1.0 (or compatible analog TDS probe module)
- **Output:** 0–2.3V analog proportional to TDS
- **Connected to:** GPIO36 (ADC1_CH0) — input-only, no internal pullup
- **Supply:** 3.3V
- **Current firmware formula:** Linear — `tds = (voltage × 354.84) + 41.97` ← **incorrect, see Known Bugs**
- **Correct formula:** Keyestudio cubic polynomial with temperature compensation applied to voltage

### Display

- **Module:** MAX7219 64×8 dot-matrix LED display
- **Interface:** Hardware SPI (MOSI=GPIO23, CLK=GPIO18, CS=GPIO5)
- **Library:** U8g2
- **Font:** Custom 3×5 pixel bitmap font (defined in firmware, uppercase + digits + symbols)
- **Layout:** Display split into two 32×8 halves. Top half = status label, bottom half = value or sub-label
- **Brightness:** MAX7219 default intensity

---

## How It Works

### Purification Cycle

The purification cycle runs fully automatically based on sensor readings. It starts, runs, and stops without any user input.

**Conditions to START purification:**
- Float sensor reads LOW (tank not full) — stable for 2+ seconds
- Water presence sensor detects supply water is available
- RO filter life > 0%
- Sediment/carbon filter life > 0%

**Start sequence (staggered to prevent inrush):**

```
Conditions met
     │
     ▼
Inlet Solenoid ON  ──── 1 second ────►  Pump ON  ──── 1 second ────►  UV Lamp ON
(water enters)                          (pressure builds)              (sterilisation)
     │
     ▼
Purification runs continuously
     │
     ▼ (any of these)
Float HIGH (tank full)          → stop all relays
Water presence lost             → stop all relays
RO filter life hits 0%          → stop all relays + show RO OVER
Sediment filter life hits 0%   → stop all relays + show FILTER OVER
```

**Stop sequence:** All three relays (UV → Pump → Inlet) are turned off simultaneously by `turnOffAllNewRelays()`. There is currently no staggered shutdown.

**Note:** Purification stopping does NOT affect dispensing. The output tank still has clean water and the dispense solenoid operates independently.

---

### Dispensing

The user presses the Dispense Button to get water from the output tank.

**On first press:**
- Dispense solenoid opens (R3)
- Nozzle LED turns ON
- Button LED turns ON
- LED strip turns ON

**On second press:**
- Solenoid closes
- Nozzle LED and Button LED turn OFF
- LED strip remains ON (runs on its own 1-minute auto-off timer)

**Auto shutoff:** Dispense solenoid auto-closes after `solenoidDuration` (currently set to 300,000ms = 5 minutes) if user forgets to press again.

**LED strip auto-off:** LED strip turns off independently after 60 seconds (`ledStripDuration`).

**Dispensing is independent of purification state.** If purification is paused (no input water, filter life over, tank full), the output tank still contains filtered water and dispensing works normally.

---

### TDS Monitoring

A TDS sensor on the output side of the filter continuously monitors purified water quality.

- **Read interval:** Every 1,000ms
- **Smoothing:** 5-sample moving average
- **Warning threshold:** 150 ppm — if TDS exceeds this, the display value blinks as a visual alert
- **Filter impact:** High TDS during active purification increments the RO TDS event counter, which contributes to faster RO membrane life degradation (see Filter Life Tracking)
- **Display:** TDS value shown on bottom half of display as `TDS:XX` during IDEAL and FULL states

> ⚠️ **Known issue:** The current TDS formula is a simple linear equation. The correct formula from Keyestudio/DFRobot is a cubic polynomial with temperature compensation applied to voltage before the formula, not after. This causes significant reading errors compared to a calibrated TDS pen meter. See [Known Bugs](#current-known-bugs).

---

### Filter Life Tracking

The system tracks two separate filter components and displays warnings when either is running low or expired.

#### RO Membrane Life

RO membrane life degrades based on **two factors** — whichever depletes faster:

| Factor | Max Value | How It Depletes |
|--------|-----------|-----------------|
| Runtime | 1000 hours | +1 second per second of active purification |
| TDS Events | 1000 events | +1 per fill session where output TDS > 150 ppm |

```
RO Life % = min(runtime_remaining%, tds_events_remaining%)
```

This reflects real RO membrane behaviour — a membrane used with consistently high input TDS degrades faster than runtime alone would suggest.

#### Sediment / Carbon Filter Life

Tracks only runtime:

| Factor | Max Value | How It Depletes |
|--------|-----------|-----------------|
| Runtime | 800 hours | +1 second per second of active purification |

```
Filter Life % = runtime_remaining%
```

#### Warnings

| Condition | Display | Behaviour |
|-----------|---------|-----------|
| Life ≤ 20% | `RO LOW` or `FILTER LOW` (blinking) | Warning only, purification continues |
| Life = 0% | `RO OVER` or `FILTER OVER` (blinking) | Purification stops entirely |

When life hits 0%, the system locks out purification until the corresponding filter is physically replaced and the life is reset via Service Mode.

---

### Display Logic

The display shows one status at a time. Priority order from highest to lowest:

| Priority | Condition | Top Half | Bottom Half | Blink? |
|----------|-----------|----------|-------------|--------|
| 1 | Service Mode — RO selected | `RO` | `RESET` | Top blinks |
| 2 | Service Mode — FILTER selected | `FILTER` | `RESET` | Top blinks |
| 3 | Service Mode — RO confirm | `RESET` | `RO` | No |
| 4 | Service Mode — FILTER confirm | `RESET` | `FILTER` | No |
| 5 | Filter Life Mode — RO | `RO` | `XX%` | No |
| 6 | Filter Life Mode — FILTER | `FILTER` | `XX%` | No |
| 7 | Both filters expired | `RO` | `OVER` | Both blink |
| 8 | RO expired | `RO` | `OVER` | Both blink |
| 9 | Filter expired | `FILTER` | `OVER` | Both blink |
| 10 | RO life ≤ 20% | `RO` | `LOW` | Both blink |
| 11 | Filter life ≤ 20% | `FILTER` | `LOW` | Both blink |
| 12 | No input water | `CHECK` | `INPUT` | No |
| 13 | Tank full / post-dispense | `FULL` | `TDS:XX` | TDS blinks if >150 |
| 14 | Normal purifying state | `IDEAL` | `TDS:XX` | TDS blinks if >150 |

Display auto-sleeps after 15 seconds of inactivity. It wakes automatically on any button press, sensor change, or state transition.

---

### Service Mode (Filter Reset)

Used by a technician after physically replacing a filter cartridge to reset the life counter back to 100%.

**Entry:** Hold Filter Reset Button for ≥200ms while display is active. System enters service mode with `RO` selected.

**Navigation:**
- **Short press** (< 2000ms): Toggle selection between `RO` and `FILTER`
- **Long press** (≥ 2000ms): Move to confirmation state (`RESET / RO` or `RESET / FILTER`)

**Confirmation:** After entering confirm state, the reset executes automatically after 2 seconds (no further action needed). This prevents accidental resets while still requiring deliberate intent.

**Timeout:** If no button activity for 10 seconds, service mode exits automatically without making any changes.

**Service mode flow:**

```
Hold Filter Reset Button (>200ms)
          │
          ▼
  Display: [RO blink] / RESET
          │
   Short press → Toggle → [FILTER blink] / RESET
          │
   Long press
          │
          ▼
  Display: RESET / RO  (or RESET / FILTER)
          │
   Wait 2 seconds
          │
          ▼
  Filter life reset to 100%
  Exit service mode
```

---

### Filter Life Check

The current life percentage of both filters can be checked at any time without entering service mode.

**Activation:** Hold Dispense Button for ≥5 seconds.

**Display sequence:**
1. `RO / XX%` shown for 5 seconds
2. `FILTER / XX%` shown for 5 seconds
3. Returns to normal display

Works in all states except when service mode is already active.

---

### Auto Sleep

To conserve power and extend display life, the MAX7219 enters power save mode after 15 seconds of inactivity.

**Display stays ON (auto-sleep blocked) during:**
- Active service mode
- Filter life display mode
- Input water fault (CHECK/INPUT state)
- Purification running (tank filling, sensors active)

**Display wakes on:**
- Any button press
- Float sensor transition (tank fills or empties)
- Water presence sensor change

---

### EEPROM Persistence

Filter life data is stored in ESP32 internal flash via the EEPROM emulation library. Data survives power cuts and resets.

**Memory layout:**

| Address | Size | Data |
|---------|------|------|
| 0 | 4 bytes | `roRuntimeUsed` (float, hours) |
| 4 | 4 bytes | `roTdsEvents` (int, event count) |
| 8 | 4 bytes | `filterRuntimeUsed` (float, hours) |
| 12 | 1 byte | Initialisation flag (`0xFF` = initialised) |

**First boot detection:** On first boot (flag ≠ `0xFF`), all counters are zeroed and the flag is written. On subsequent boots, stored values are loaded.

> ⚠️ **Known issue:** The initialisation flag uses `0xFF` which is also the erased state of flash. On a brand-new ESP32, the flag already reads `0xFF`, so the `else` branch runs and loads garbage values as runtime hours. See [Known Bugs](#current-known-bugs).

> ⚠️ **Known issue:** `EEPROM.commit()` is called every second during purification (inside `updateFilterLife()`). ESP32 flash is rated for ~100,000 write cycles. At 1 write/second this exhausts flash in approximately 27 hours of total purification runtime. See [Known Bugs](#current-known-bugs).

---

## Display Messages Reference

| Top | Bottom | Meaning |
|-----|--------|---------|
| `IDEAL` | `TDS:XX` | Normal operation, purification running or tank filling |
| `FULL` | `TDS:XX` | Tank at full level |
| `CHECK` | `INPUT` | No input supply water detected |
| `RO` | `LOW` | RO membrane below 20% life (blinking) |
| `FILTER` | `LOW` | Sediment/carbon filter below 20% life (blinking) |
| `RO` | `OVER` | RO membrane life expired — purification locked out (blinking) |
| `FILTER` | `OVER` | Sediment/carbon filter expired — purification locked out (blinking) |
| `RO` | `RESET` | Service mode active, RO selected (top blinks) |
| `FILTER` | `RESET` | Service mode active, FILTER selected (top blinks) |
| `RESET` | `RO` | Service mode confirm — RO life will be reset in 2s |
| `RESET` | `FILTER` | Service mode confirm — FILTER life will be reset in 2s |
| `RO` | `XX%` | Filter life check — RO membrane remaining life |
| `FILTER` | `XX%` | Filter life check — sediment/carbon filter remaining life |

> If `TDS:XX` is blinking, output TDS is above 150 ppm — water quality alert.

---

## Current Known Bugs

These are confirmed bugs present in the current version of the firmware (`ro_sep17b.ino`). The next version will address all of these.

---

### 🔴 BUG-01 — Water Sensor Logic Inverted: Purification Never Starts

**File:** `loop()`, line 413  
**Severity:** Critical — purification is completely non-functional

```cpp
// Current (WRONG):
bool waterPresent = digitalRead(WATER_SENSOR) == HIGH;
```

`WATER_SENSOR` uses `INPUT_PULLUP` (line 332). With an internal pull-up, the pin reads `HIGH` by default when no sensor signal is present. This means `waterPresent` is almost always `true`, which immediately triggers `turnOffAllNewRelays()` every loop iteration. The pump, inlet, and UV relay never turn on.

**Fix:**
```cpp
bool waterPresent = digitalRead(WATER_SENSOR) == LOW; // active-LOW sensor
```

---

### 🔴 BUG-02 — TDS Formula Wrong: Linear Instead of Cubic Polynomial

**File:** `readTDS()`, lines 302–304  
**Severity:** Critical — TDS readings are significantly inaccurate

```cpp
// Current (WRONG) — simple linear formula:
float tdsValue = (voltage * 354.84) + 41.97;
// Temperature compensation applied to result — also wrong:
tdsValue = tdsValue * (1.0 + 0.02 * (waterTemp - 25.0));
```

The official Keyestudio/DFRobot formula uses a **cubic polynomial** and applies temperature compensation to the **voltage** before the formula, not to the result after. The linear formula gives readings that diverge significantly from real values, which explains the large difference compared to a calibrated TDS pen meter.

**Fix:**
```cpp
float compCoeff = 1.0f + 0.02f * (waterTemp - 25.0f);
float compVoltage = voltage / compCoeff;  // compensate voltage FIRST
float tds = (133.42f * compVoltage * compVoltage * compVoltage
           - 255.86f * compVoltage * compVoltage
           + 857.39f * compVoltage) * 0.5f;
if (tds < 0) tds = 0;
```

---

### 🔴 BUG-03 — `readTDS()` Blocks 500ms Every Second

**File:** `readTDS()`, lines 297–299  
**Severity:** Critical — entire loop halts 500ms out of every 1000ms

```cpp
for (int i = 0; i < NUM_SAMPLES; i++) {
    analogSum += analogRead(TDS_SENSOR_PIN);
    delay(10);  // 50 samples × 10ms = 500ms blocked
}
```

During this 500ms: button presses are not detected, relay sequencing cannot advance, display cannot update, float sensor debounce timer is frozen.

**Fix:** Replace with a non-blocking ring buffer sampled every 40ms using `millis()`. Use a median filter on the 30-sample buffer instead of simple average — median also rejects EMF-induced ADC spikes.

---

### 🔴 BUG-04 — EEPROM Flash Wear: `commit()` Every Second

**File:** `updateFilterLife()`, line 267  
**Severity:** Critical — destroys ESP32 flash in ~27 hours of purification runtime

`EEPROM.commit()` is called inside `updateFilterLife()`, which is called every second during active purification and on every TDS event. ESP32 internal flash is rated for approximately 100,000 write cycles. At 1 commit/second, flash exhaustion occurs in ~27 hours of total cumulative purification time.

**Fix:** Gate commits to at most once every 5 minutes. Also commit immediately on `turnOffAllNewRelays()` to capture runtime before power loss.

---

### 🔴 BUG-05 — RO TDS Events Drain Filter Life in ~17 Minutes

**File:** `loop()`, lines 451–454  
**Severity:** Critical — RO filter shows 0% life after one fill session with elevated TDS

```cpp
if (blinkTds && relaySequenceStep > 0) {
    roTdsEvents++;  // increments every second when TDS > 150
}
// RO_TDS_EVENTS_MAX = 1000 → life expires in 1000 seconds ≈ 17 minutes
```

**Fix:** Increment `roTdsEvents` only once per fill session, not once per second. Use a per-session flag that resets when `relaySequenceStep` returns to 0.

---

### 🟠 BUG-06 — U8G2 HW SPI Constructor Wrong Pin Arguments

**File:** Global, line 41  
**Severity:** High — display may not initialise correctly on all boards

```cpp
// Current (WRONG):
U8G2_MAX7219_64X8_F_4W_HW_SPI u8g2(U8G2_R0, MAX7219_CS, MAX7219_DIN, MAX7219_CLK);
// MAX7219_DIN is passed as 'dc' and MAX7219_CLK as 'reset' — MAX7219 has neither
```

For HW SPI mode, the ESP32 uses its hardware MOSI/SCK pins automatically. DIN and CLK should not be passed here.

**Fix:**
```cpp
U8G2_MAX7219_64X8_F_4W_HW_SPI u8g2(U8G2_R0, MAX7219_CS, U8X8_PIN_NONE, U8X8_PIN_NONE);
```

---

### 🟠 BUG-07 — Float Sensor Debounce Too Short + No Fill Cooldown

**File:** `loop()`, line 122 and `turnOffAllNewRelays()`  
**Severity:** High — causes overflow event (see [Known Real-World Issues](#known-real-world-issues))

2 seconds is insufficient for water surface to settle after the pump stops. Pump vibration causes the float to oscillate, resetting the debounce timer repeatedly, so `floatSensorStable` never becomes `true`. When `turnOffAllNewRelays()` runs, the pump stopping causes additional vibration that immediately bounces the float LOW, and with no cooldown period the fill sequence restarts instantly.

**Fix:** Increase `floatSensorDelay` to 5000ms. Add a 15-second fill cooldown timer in `turnOffAllNewRelays()` that blocks the relay sequence from restarting too soon.

---

### 🟠 BUG-08 — Boot Always Shows `FULL/TDS` Regardless of Actual Tank State

**File:** `setup()`, line 380  
**Severity:** Medium — misleading display on startup

```cpp
dispenseDisplayActive = true;  // hardcoded — ignores actual float sensor
```

On boot, the display shows `FULL / TDS:XX` even if the tank is empty.

**Fix:** Read the float sensor at boot and set the initial display state accordingly.

---

### 🟠 BUG-09 — EEPROM Initialisation Flag Uses `0xFF` (Erased Flash Default)

**File:** `setup()`, lines 352–353  
**Severity:** Medium — loads garbage runtime values on a brand-new ESP32

```cpp
if (initializedFlag != 0xFF) { // 0xFF is the erased state of flash
```

On a new ESP32, flash reads `0xFF` everywhere. The condition `!= 0xFF` is `false`, so the code jumps to the `else` branch and calls `EEPROM.get()` on uninitialised memory, loading random float values as filter runtime hours.

**Fix:** Use a custom magic number (e.g. `0xA5`) instead of `0xFF`.

---

### 🟡 BUG-10 — Serial Debug Prints Every Loop Iteration

**File:** `loop()`, line 416  
**Severity:** Low — floods serial monitor, slightly slows execution

```cpp
Serial.println("Button States: DISPENSE=...");  // runs thousands of times per second
```

**Fix:** Rate-limit to once per 500ms using a static `millis()` timer.

---

### 🟡 BUG-11 — Both Filters OVER Shows Only `RO/OVER`

**File:** `loop()`, lines 729–732  
**Severity:** Low — `FILTER OVER` warning is suppressed when both are expired

When both `roLifePercent` and `filterLifePercent` are 0, the first branch matches and only shows `RO/OVER`. `FILTER/OVER` is never displayed.

**Fix:** Alternate between `RO/OVER` and `FILTER/OVER` on each blink cycle.

---

### 🟡 BUG-12 — `filterResetActive` Flag Is Never Set True

**File:** throughout  
**Severity:** Low — dead display branch

`filterResetActive` is declared, used in display logic, and reset in `turnDisplayOff()` but is **never assigned `true`** anywhere. The `FILTER/RESET` display case (line 725) is unreachable dead code.

---

## Known Real-World Issues

### Overflow Event (Occurred Once)

The tank overflowed while the system continued running normally. The display appeared to show a normal status. The root cause was not definitively identified at the time but after cleaning and restarting, the system worked normally.

**Most likely causes, in order of probability:**

1. **Water entered the float sensor connector** — causing the signal line to short to GND, permanently reading LOW (tank empty). The system saw no "tank full" signal and kept filling. Consistent with the fact that cleaning and drying fixed it.

2. **Float sensor oscillation defeating debounce** — pump vibration caused the float to bounce between HIGH and LOW continuously, resetting the 2-second debounce timer each time. `floatSensorStable` never reached `true`. System kept filling indefinitely.

3. **EMF spike corrupting GPIO35 reading** — relay/solenoid switching generates back-EMF that can corrupt ADC input readings on GPIO35 (input-only pin with no filtering). A repeated EMF event could prevent the float sensor from ever stabilising.

4. **Fill cooldown bug** — after pump stopped briefly, vibration retriggered the fill sequence immediately before the float could settle.

**What the display likely showed:** `IDEAL / TDS:XX` — because `floatSensorStable = false` and `waterPresent = false` lands at the final `else` branch. The system genuinely believed everything was normal.

**Fix status:** Addressed by BUG-07 fix (longer debounce + cooldown) plus the planned max fill watchdog and hardware watchdog in the next version.

---

## Planned Improvements

The following improvements are planned for the next firmware version.

### Safety

| ID | Feature | Description |
|----|---------|-------------|
| S-01 | Max fill duration watchdog | If purification runs continuously for more than 90 minutes, shut all relays off and latch a fault state. Prevents overflow if float sensor fails or connector gets wet. |
| S-02 | ESP32 hardware watchdog (WDT) | Use `esp_task_wdt` to reset the ESP32 if `loop()` freezes for more than 30 seconds. All relays go HIGH (safe) on reset. Prevents stuck-ON relays from software crash. |
| S-03 | Power-loss runtime save | Commit EEPROM immediately when purification stops (not just on a timer). Ensures runtime is not lost if power cuts during a fill session. |
| S-04 | Fill cooldown after pump stops | 15-second lockout after `turnOffAllNewRelays()` before fill can restart. Prevents immediate retrigger from pump vibration. |

### Bug Fixes

All bugs listed in [Current Known Bugs](#current-known-bugs) will be resolved.

### Code Quality

| ID | Improvement | Description |
|----|------------|-------------|
| C-01 | Non-blocking TDS sampling | Replace `delay(10)` loop with millis-based 40ms ring buffer sampling. |
| C-02 | Median filter for TDS | Replace moving average with median filter — rejects EMF-induced ADC spikes. |
| C-03 | EEPROM commit rate limiting | Commit at most once per 5 minutes during normal operation. |
| C-04 | Boot display state from sensors | Read float sensor at boot to decide initial display content. |
| C-05 | Serial rate limiting | Debug prints rate-limited to 500ms. |
| C-06 | Display reinit after relay toggle | Call `u8g2.begin()` after any relay switching event to recover from EMF-corrupted MAX7219 registers. |
| C-07 | Display stays ON during fill | Prevent auto-sleep while purification is actively running. |
| C-08 | Fix dual-filter OVER display | Alternate `RO/OVER` and `FILTER/OVER` on blink cycle when both are expired. |

### Hardware Recommendations

| ID | Fix | Benefit |
|----|-----|---------|
| H-01 | 100nF ceramic cap on GPIO35 to GND | Filters EMF transients that corrupt float sensor reading |
| H-02 | 10kΩ pull-down on GPIO35 to GND | Ensures definitive LOW when float switch is open |
| H-03 | 1N4007 freewheeling diode on each relay coil | Eliminates back-EMF at source before it reaches GPIO lines or display supply |
| H-04 | Secondary mechanical overflow switch | Hardware-level overflow cutoff wired in series with pump relay coil, independent of ESP32 |

---

## Libraries Required

Install via Arduino Library Manager or PlatformIO:

| Library | Purpose | Version Tested |
|---------|---------|----------------|
| U8g2 | MAX7219 display driver | 2.x |
| EEPROM | Filter life persistence | Built-in (ESP32 Arduino core) |
| SPI | Hardware SPI for display | Built-in |

**Board:** ESP32 Arduino Core (espressif/arduino-esp32)  
**Tested on:** ESP32 DevKitC v4

---

## File Structure

```
smart-ro-purifier/
│
├── ro_sep17b.ino          ← Current firmware (has known bugs, see above)
└── README.md              ← This file
```

---

> **Project by:** Bhavesh Waghmare (Ghost)  
> **Platform:** ESP32  
> **Status:** Active development — v2 in progress with all bug fixes and safety features
