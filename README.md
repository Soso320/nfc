

# STM32G0 NFC Irrigation System - Developer Documentation

## 1. System Architecture Overview

This device operates entirely offline without Wi-Fi or Bluetooth. It uses an **ST25DV Dynamic NFC Tag** as a dual-interface memory bridge between the smartphone (RF) and the STM32 microcontroller (I2C).

The communication relies on an **Asymmetric Cryptographic Handshake**:

* **Write Protocol (Phone → STM32):** The frontend writes commands as compact `application/json` NDEF MIME records.
* **Read Protocol (STM32 → Phone):** The STM32 parses the JSON, executes it, and instantly overwrites the NFC tag with a 128-byte `application/octet-stream` NDEF binary record containing the entire system state.
* **The Handshake:** A frontend app must *never* blindly trust a write. It must write the JSON, keep the NFC listener active, and wait for the tag to switch from JSON to Binary format. The presence of the Binary file is the hardware's ACK (Acknowledgement).

---

## 2. Global Time Synchronization (RTC Epochs)

The STM32 relies on a DS3231 RTC for scheduling, which calculates UNIX epochs based on **Local Time**, not UTC.

**CRITICAL RULE:** The frontend must *never* use standard `Date.now() / 1000` to generate epochs, or schedules will drift by your timezone offset. Epochs must be calculated using exact local time components.

**Javascript Local Epoch Calculator Example:**

```javascript
function getLocalEpoch(dateObj) {
    const year = dateObj.getFullYear(); const mo = dateObj.getMonth() + 1; const date = dateObj.getDate();
    const h = dateObj.getHours(); const m = dateObj.getMinutes(); const s = dateObj.getSeconds();
    const days_in_month = [31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31];
    let days = 0;
    for (let i = 1970; i < year; i++) { days += 365 + ((i % 4 === 0 && (i % 100 !== 0 || i % 400 === 0)) ? 1 : 0); }
    for (let i = 1; i < mo; i++) {
        days += days_in_month[i - 1];
        if (i === 2 && (year % 4 === 0 && (year % 100 !== 0 || year % 400 === 0))) days++;
    }
    days += (date - 1);
    return (days * 86400) + (h * 3600) + (m * 60) + s;
}

```

---

## 3. Write Protocol (JSON Commands)

To write to the device, wrap the following JSON structures into an `application/json` NDEF MIME record.

**MANDATORY INJECTION:** To prevent RTC drift, *every* command payload must have the current local time injected into it before writing.

* *Required Keys:* `"y"` (Year offset from 2000), `"mo"` (1-12), `"d"` (1-31), `"dw"` (1-7), `"h"` (0-23), `"m"` (0-59), `"s"` (0-59).

### 3.1. Add/Update Schedule (`cmd: "sch"`)

Configures one of the 10 memory slots.

```json
{
  "cmd": "sch",
  "idx": 0,           // Slot index (0-9)
  "z": "A",           // Zone ('A' or 'B')
  "int": 86400,       // Interval in seconds (0 = One-Shot mode)
  "dur": 60,          // Watering duration in seconds
  "strt": 1789419420, // Starting local epoch
  "y": 26, "mo": 9, "d": 14, "dw": 1, "h": 20, "m": 53, "s": 0 // Injected RTC Sync
}

```

### 3.2. Manual Hardware Overrides

Overrides bypass standard logic and execute instantly.

* **Pump Command:** `dir` mapping is `0`=Forward, `1`=Reverse.

```json
{
  "cmd": "pump",
  "dir": 0,
  "spd": 100, 
  "sec": 10,
  "y": 26, "mo": 9, "d": 14, "dw": 1, "h": 20, "m": 53, "s": 0 
}

```

* **Solenoid Command:** `state` mapping is `1`=Open, `0`=Closed.

```json
{
  "cmd": "sol",
  "zone": "A",
  "state": 1,
  "y": 26, "mo": 9, "d": 14, "dw": 1, "h": 20, "m": 53, "s": 0 
}

```

### 3.3. Delete Schedule & Options

* **Delete Schedule:** `{"cmd":"del", "idx":0, "y":26, ...}`
* **Update Config:** `{"cmd":"opt", "delay":12, "calib":45, "y":26, ...}`
* **Force RTC Sync Only:** `{"cmd":"rtc", "y":26, ...}`

---

## 4. Read Protocol (Binary Telemetry)

The STM32 automatically overwrites the JSON with a 128-byte `application/octet-stream` NDEF MIME record. All multi-byte integers are formatted in **Big-Endian**.

### Byte Map (128 Bytes Total):

| Offset | Size | Type | Description |
| --- | --- | --- | --- |
| `0` | 1 byte | `uint8_t` | Pump Direction (`0`=Fwd, `1`=Rev) |
| `1` | 1 byte | `uint8_t` | Pump Speed (0-100%) |
| `2` | 1 byte | `uint8_t` | Solenoid A State (`1`=Open, `0`=Closed) |
| `3` | 1 byte | `uint8_t` | Solenoid B State (`1`=Open, `0`=Closed) |
| `4` | 2 bytes | `uint16_t` | Pipe Travel Delay (Seconds) |
| `6` | 2 bytes | `uint16_t` | Flow Calibration Offset |
| `8` | 120 bytes | `Array` | 10 Schedule Slots (12 bytes per slot) |

### Schedule Slot Map (12 Bytes per slot):

| Local Offset | Size | Type | Description |
| --- | --- | --- | --- |
| `0` | 1 byte | `uint8_t` | Active Flag (`1`=Active, `0`=Empty) |
| `1` | 1 byte | `uint8_t` | Zone Code (`65`='A', `66`='B') |
| `2` | 4 bytes | `uint32_t` | Interval in Seconds (`0` = One-Shot) |
| `6` | 4 bytes | `uint32_t` | Last Run Epoch |
| `10` | 2 bytes | `uint16_t` | Duration in Seconds |

---

## 5. Embedded Failsafes & Hardcoded Limits

The frontend UI must be designed around these hardware rules:

1. **One-Shot Auto-Erase:** If a schedule's interval is set to `0`, the firmware treats it as a One-Shot event. It will trigger once at the specified epoch, and the STM32 will permanently delete it from memory automatically.
2. **Anti-Flood Clamping:** The STM32 hard-clamps all watering durations to **3600 seconds (1 hour) max**. Sending a duration of `9000` will be silently clamped to `3600` by the hardware.
3. **Deadheading Interlocks:** The firmware will reject a `cmd: pump` ON request if both solenoids are currently closed. The frontend must open a solenoid first.
4. **Schedule Grace Period:** If the system is unpowered and misses a schedule by > 300 seconds (5 minutes), it skips the watering entirely. If it's a repeating schedule, it fast-forwards the epoch to the next phase. If it's a one-shot, it deletes it.
5. **Universal Boot Wipe:** On power-up, the STM32 instantly overwrites the NFC tag with the binary state, wiping any offline commands to prevent ghost executions.

---

## 6. Known WebNFC Limitations & Implementation Quirks

If building a Progressive Web App (PWA) using WebNFC:

* **Write Delay:** The STM32 requires ~1 second to parse a JSON payload, burn it to its internal EEPROM via active polling, and push the binary data back to the NFC tag. If sending queued commands, the frontend **must inject a 1.2-second delay** between writes, or the I2C bus will collide and drop commands.
* **Aggressive Android Caching:** Android's NFC OS intercepts the tag before the browser gets it. To read the live binary data instead of a stale OS cache, the frontend must force an active listener using an `AbortController`: `await ndef.scan({ signal: abortController.signal });`
* **EEPROM Wear:** The ST25DV EEPROM wears out after ~1,000,000 writes. The frontend should strongly discourage interval schedules shorter than 1 hour to prevent the hardware from burning out its flash memory prematurely.
