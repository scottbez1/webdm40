# ALIENTEK DM40 — Bluetooth LE Protocol Specification

**Reverse-engineered from `ATK-XTOOL.apk`** (build date 2025-06-25)
GATT UUIDs corroborated from independent HCI captures (§10)
Rev 1.2 — corrected secondary-slot blanking rule (§6.5)
Rev 1.3 — corrected secondary-slot unit tables (§6.6.1)
Rev 1.4 — added AC+DC and temperature secondary units (§6.6.1)
Rev 1.5 — AC+DC slot order confirmed on hardware (§6.6.1)
Document revision 1.5

---

## 0. Scope and provenance

This document specifies the BLE application protocol used by the ALIENTEK DM40
series handheld multimeter (models DM40A, DM40B, DM40C), as implemented by the
vendor's official "ATK-XTOOL" Android application.

`ATK-XTOOL.apk` is a **.NET MAUI** application. It contains no meaningful Java
bytecode; the entire protocol implementation lives in the managed assembly
`ATKMobileBLE.dll`, stored LZ4-compressed inside `assemblies/assemblies.blob`
(an `XABA`-format .NET-for-Android AssemblyStore). Everything in this document
was read from the CIL of that assembly.

Principal types of interest, for anyone re-deriving this:

| Type | Role |
|---|---|
| `ATKMobileBLE.ATKProtocol.FrameEntity` | frame container, checksum |
| `ATKMobileBLE.ATKProtocol.ManageFrame` | receive-side byte state machine |
| `ATKMobileBLE.ATKProtocol.ManageCMD` | send / await-response engine |
| `ATKMobileBLE.ATKProtocol.DM40.DM40Info` | function code constants |
| `ATKMobileBLE.ATKProtocol.DM40.ManageCMD_DM40` | DM40 command layer, heartbeat |
| `ATKMobileBLE.ATKProtocol.DM40.ParametersDM40` | measurement decoding, bit fields |
| `ATKMobileBLE.BLEUtils.BLEDev` | GATT transport |

**Confidence.** Sections 2–7 are transcribed directly from IL and are
high-confidence; §10 cross-checks them against an independent reverse
engineering of the same device done from wire captures, and the two agree on
every overlapping point. The GATT UUIDs in §1.1.1 are the one part of this
document **not** derived from the APK — they are unobtainable from it — and come
from that third-party source. Section 8 lists remaining open points. Two apparent
vendor bugs are flagged inline; they are marked clearly and should **not** be
reproduced in a clean-room implementation.

---

## 1. GATT transport layer

### 1.1 Service and characteristic discovery

The application **hardcodes no UUIDs whatsoever**. On connect it enumerates
every service, then every characteristic within each service, and selects
purely by capability flags:

| Capability test | Assignment |
|---|---|
| `CanWrite && CanRead && CanUpdate` | used as **both** read and write characteristic |
| `CanRead && CanUpdate` | used as read (notify) characteristic |
| `CanWrite` | used as write characteristic |

The first characteristic matching each test wins. Service discovery is retried
on failure with a bounded attempt counter.

The Web Bluetooth API **cannot replicate this discovery strategy.** A page must
declare service UUIDs up front in `filters` or `optionalServices` before
`requestDevice()`; `getPrimaryServices()` only returns services that were
declared. Capability-based blind enumeration is not available to web apps.

### 1.1.1 Confirmed UUIDs

The concrete UUIDs are **not present anywhere in the APK**, but they have been
independently established from BLE HCI captures by the `DM40GUI` project
(see §10):

| Role | UUID |
|---|---|
| **Service** | `0000fff0-0000-1000-8000-00805f9b34fb` |
| **Notify** (meter → host) | `0000fff1-0000-1000-8000-00805f9b34fb` |
| **Write** (host → meter) | `0000fff3-0000-1000-8000-00805f9b34fb` |

This is the common `FFF0` vendor profile (16-bit aliases `0xFFF0`, `0xFFF1`,
`0xFFF3`), consistent with a generic UART-passthrough BLE module. Note that
`FFF1` and `FFF3` are **separate** characteristics — this device does not use
the single read/write/notify characteristic that the vendor app's first
capability test would have matched.

The same layout is reported to serve the EL15 electronic load, so it is likely
a platform-wide choice by ALIENTEK rather than DM40-specific.

Web Bluetooth setup:

```js
const device = await navigator.bluetooth.requestDevice({
  filters: [{ namePrefix: 'DM40' }],
  optionalServices: [0xfff0],
});
const server  = await device.gatt.connect();
const service = await server.getPrimaryService(0xfff0);
const notify  = await service.getCharacteristic(0xfff1);
const write   = await service.getCharacteristic(0xfff3);
await notify.startNotifications();
notify.addEventListener('characteristicvaluechanged', onFrameBytes);
```

Because these come from a third party rather than from the APK, confirm them
against your own hardware before treating them as settled (§10.2).

### 1.2 Scanning

No name filter, no manufacturer-data filter, no service-UUID filter is applied.
Every advertising device with a non-empty name is surfaced to the user, who
picks manually.

The vendor manual instructs users to select the device whose name **starts with
`DM40`**, and states that the BLE name is user-modifiable only in part: the
**first 5 characters are fixed to the device model** and up to 10 further
characters are editable. A `namePrefix: 'DM40'` filter is therefore safe and
survives user renaming.

### 1.3 Write and notify

Outbound frames are written with a single `WriteAsync` call. There is **no
application-level fragmentation** — the largest defined frame is 26 bytes
(SetBleName), which fits a default 23-byte ATT MTU only after MTU negotiation.
In practice Android negotiates a larger MTU automatically. A Web Bluetooth
client should use `writeValueWithResponse()` and, if 26-byte writes fail on a
default MTU, be aware that the vendor app never handles that case.

Inbound data arrives via characteristic notifications and is appended to a byte
buffer feeding the frame parser (§3.2). Notifications must be enabled
(`startNotifications()`) before any command is sent.

---

## 2. Frame format

All frames, both directions, share one layout:

```
 offset  0        1     2        3          4         5 .. 5+N-1    5+N
        ┌────────┬───────────────┬──────────┬────────┬────────────┬───────┐
        │  flag  │  dev (u16 LE) │ function │ len(N) │  data[N]   │ check │
        └────────┴───────────────┴──────────┴────────┴────────────┴───────┘
          1 byte      2 bytes       1 byte    1 byte    N bytes      1 byte
```

Total frame length = **N + 6**.

### 2.1 flag

| Direction | Value |
|---|---|
| Host → meter (TX) | `0xAF` |
| Meter → host (RX) | `0xDF` |

**These differ.** The receive parser syncs on `0xDF` only; the transmit path
writes `0xAF` unconditionally. Using one value for both directions will fail
silently — this is the single most likely early implementation mistake.

### 2.2 dev

16-bit device-type identifier, **little-endian** on the wire.

| Device | Value | Wire bytes |
|---|---|---|
| BLE03 | `0x0000` | `00 00` |
| HP15 | `0x0207` | `07 02` |
| **DM40** | **`0x0305`** | **`05 03`** |
| ND1 | `0x0306` | `06 03` |
| EL15 | `0x0307` | `07 03` |
| Wildcard / unknown | `0xFFFF` | `FF FF` |

`0xFFFF` is used as a broadcast address during registration, before the host
knows what it is talking to (§4.1). All subsequent DM40 traffic uses `0x0305`.

### 2.3 function

Command / response identifier. See §5 for the DM40 set. A response echoes the
function code of the request that triggered it; this echo is the **only**
request/response correlation mechanism (there is no sequence number).

### 2.4 len and data

`len` is a single unsigned byte giving the payload length N, so 0 ≤ N ≤ 255 and
the maximum theoretical frame is 261 bytes. In practice the DM40 uses N ≤ 20.

### 2.5 check

One byte, computed over **flag, dev, function, len, and data** — i.e. every byte
of the frame except the checksum itself:

```
sum   = Σ frame[0 .. 4+N]          (mod 2^16 accumulation, no truncation)
check = (256 - (sum mod 256)) and 0xFF
```

This is a two's-complement (negated modular) checksum: the sum of the **entire**
frame including the check byte is ≡ 0 (mod 256). That property gives you a
convenient one-line validity test on receive.

Reference implementation:

```js
function checksum(bytes) {                 // bytes = frame without check byte
  let sum = 0;
  for (const b of bytes) sum = (sum + b) & 0xFFFF;
  return (256 - (sum % 256)) & 0xFF;
}
```

### 2.6 Worked example

Request real-time data (function `0x09`, no payload):

```
AF 05 03 09 00 3F
│  └──┬──┘ │  │  └─ check
│     │    │  └──── len = 0
│     │    └─────── function = 0x09
│     └──────────── dev = 0x0305, little-endian
└────────────────── flag = 0xAF (TX)

sum   = 0xAF + 0x05 + 0x03 + 0x09 + 0x00 = 0xC0 (192)
check = (256 - 192) & 0xFF = 0x40
```

> Note: worked checksums here are computed from the formula above and are
> intended to illustrate the algorithm. Validate against a live device before
> relying on any hand-computed constant.

---

## 3. Framing and reliability

### 3.1 Transmit

1. Build the frame with `flag = 0xAF`, `dev = 0x0305`, the function code, the
   payload and its length.
2. Compute and append the checksum.
3. Write the whole frame in one GATT write.

### 3.2 Receive — parser state machine

Inbound bytes are accumulated in a buffer and consumed by a seven-state machine
(`AnalyFrameStatu`: `None → Start → Dev → FrameFunction → DataLenght → Data →
Check`):

| State | Action |
|---|---|
| `None` | Scan for `0xDF`. Any other byte is discarded and the parser resets. |
| `Start` | Read 2 bytes → `dev` (u16 LE). |
| `Dev` | Read 1 byte → `function`. |
| `FrameFunction` | Read 1 byte → `len` (N). |
| `DataLenght` | Wait until buffer holds ≥ N+5 bytes, then read N bytes → `data`. |
| `Data` | Read 1 byte → received checksum; recompute and compare. |
| `Check` | On match: emit frame, reset. On mismatch: **discard and reset.** |

A partially-assembled frame is abandoned if no further bytes arrive within
**500 ms**, after which the parser resets to `None`. There is no retransmission
or NAK — a corrupted frame is simply dropped and the command times out.

### 3.3 Request / response correlation

`ManageCMD` holds a single `recEntity` slot. Before sending, the slot's function
byte is set to the sentinel `0xFF`. After sending, the caller polls every
**10 ms** until `recEntity.function == requestedFunction`, or until **3000 ms**
have elapsed.

Consequences for an implementation:

- **Only one command may be in flight at a time.** There is no tag or sequence
  number; a second concurrent request would be matched by the wrong waiter.
- A response arriving after its timeout will be picked up by whatever command is
  waiting next, if the function codes happen to collide. Serialise all commands
  through a queue.

### 3.4 Unsolicited frames

`ManageCMD` routes any frame with `function >= 0xD3` to an "active frame"
handler rather than the response slot. **The DM40's implementation of that
handler is an empty method** — it does nothing. The DM40 is therefore
**poll-only**; it never pushes measurements. All data acquisition is host-driven
(§4.3).

### 3.5 Result codes

The command layer returns an `APPState`:

| Value | Name | Meaning |
|---|---|---|
| `0` | `APPState_OK` | success |
| `1` | `APPState_SendCMD_Err` | GATT write failed |
| `2` | `APPState_CMD_TimeOut` | no matching response within 3000 ms |
| `3` | `APPState_CMD_RetValERR` | response received but contents invalid |
| `4` | `APPState_CMD_HeartWaitERR` | could not pause the heartbeat task |
| `0xFFFF` | `APPState_ERROR` | exception thrown |

---

## 4. Session lifecycle

### 4.1 Registration handshake

Immediately after GATT connection and notification enable, the host sends
function `0x00` (Register) to the **wildcard** device address `0xFFFF`:

```
TX:  AF FF FF 00 00 <ck>
```

The response payload identifies the device:

| Offset | Size | Field |
|---|---|---|
| 0 | 2 | device type, u16 LE (`0x0305` for DM40) |
| 2 | 1 | model byte |

The model byte is ASCII:

| Byte | Model | Display counts | `devModelIndex` |
|---|---|---|---|
| `0x41` `'A'` | DM40A | 4000 | 0 |
| `0x42` `'B'` | DM40B | 5000 | 1 |
| `0x43` `'C'` | DM40C | 6000 | 2 |

The app validates the type against its known set and aborts with
`APPState_CMD_RetValERR` if unrecognised. The model byte is read only when the
type is `0x0305`.

`devModelIndex` is computed as `model - 0x41`, clamped to 0 for any byte above
`0x43`. It selects range labels throughout §6.

### 4.2 Optional identification

After registration the host may query:

- **`0x02` GetVer** — firmware versions (§5.2)
- **`0x08` GetBleName** — advertised BLE name (§5.3)

Neither is required for measurement.

### 4.3 Measurement loop (heartbeat)

The app starts a background task that polls `0x09` GetRealTimeData
continuously, with a **200 ms** delay between iterations. Each successful
response updates the display model.

Failure handling: a counter tracks consecutive failures. It is initialised to
100 and reset to 0 on each success; **100 consecutive failures** raise the
"heart lost" event and the session is treated as disconnected.

### 4.4 Heartbeat coordination — important

Because only one command can be in flight (§3.3), every user-initiated command
(SetGear, SetHOLD, SetREL, SetAuto, SetBleName, GetBleName) is wrapped in a
"stop the heartbeat, run the command, restart the heartbeat" sequence:

1. Set the stop flag.
2. Await confirmation that the polling task has yielded.
3. Issue the command and await its response.
4. Clear the stop flag; polling resumes.

If step 2 times out the command returns `APPState_CMD_HeartWaitERR` without
being sent. **Any reimplementation must serialise polling against commands the
same way**, or responses will be mismatched. A simple async mutex around all
frame transactions is sufficient and cleaner than the vendor's flag dance.

---

## 5. Command reference

All DM40 commands use `dev = 0x0305`, `flag = 0xAF`.

| Fn | Name | TX payload | RX payload | Notes |
|---|---|---|---|---|
| `0x00` | Register | none | 3 B | sent to `dev=0xFFFF` |
| `0x01` | Conn | — | — | declared but unused in the DM40 path |
| `0x02` | GetVer | none | 3 B | |
| `0x03` | SetAuto | 1 B | echo | auto-range toggle |
| `0x04` | SetHOLD | 1 B | echo | display hold |
| `0x05` | SetREL | 1 B | echo | relative/zero mode |
| `0x06` | SetGear | 1 B packed | echo | **function + range + AC/DC** |
| `0x07` | SetBleName | 20 B | echo | |
| `0x08` | GetBleName | none | ≤20 B | |
| `0x09` | GetRealTimeData | none | 11 B | the measurement poll |

Function `0x01` (`Function_Conn`) is defined in `DM40Info` but never invoked by
the DM40 code path. Its payload is unknown.

### 5.1 `0x09` GetRealTimeData

TX payload: none. RX payload: 11 bytes. See §6 — this is the core of the
protocol.

### 5.2 `0x02` GetVer

TX payload: none. RX payload, 3 bytes:

| Offset | Field |
|---|---|
| 0 | bootloader version |
| 1 | application version |
| 2 | hardware version |

Each is a single byte rendered as `value / 10` with one decimal place, e.g.
`0x0A` (10) → `"V 1.0"`, `0x17` (23) → `"V 2.3"`.

### 5.3 `0x07` / `0x08` BLE name

**SetBleName (`0x07`)** — payload is exactly **20 bytes**: the name encoded as
ASCII, truncated to at most 19 characters, zero-padded to 20. (The truncation
is to `length - 1` when the encoded name meets or exceeds 20 bytes, guaranteeing
at least one terminating zero.)

**GetBleName (`0x08`)** — response payload is the name as bytes; decoding stops
at the first zero byte. A response length of `0xFFFF` is treated as an error
condition.

### 5.4 `0x04` SetHOLD, `0x05` SetREL, `0x03` SetAuto

Each takes a single byte payload carrying the desired new state (0 or 1). These
are **absolute, not toggles** — the app computes the inverse of the currently
reported state and sends that:

```js
setHoldValue = (currentHold === 0) ? 1 : 0;   // then send 0x04 with [setHoldValue]
```

Because the current state comes from the most recent `0x09` poll, always issue a
fresh poll (or use the cached value from the last 200 ms tick) before computing
the inverse.

### 5.5 `0x06` SetGear

Single byte payload, packed identically to the `gearStatus` byte in the
measurement response (§6.2). This one command sets **measurement function,
range, and AC/DC mode together** — there are no separate commands for them.
See §7 for the full procedure.

---

## 6. Real-time measurement data (`0x09` response)

### 6.1 Payload layout — 11 bytes

| Offset | Size | Field | Description |
|---|---|---|---|
| 0 | 1 | `gearStatus` | function / range / mode — §6.2 |
| 1 | 1 | `status` | hold, REL, battery, charging — §6.3 |
| 2 | 1 | `secInfo1` | secondary display 1 descriptor — §6.4 |
| 3 | 1 | `secInfo2` | secondary display 2 descriptor — §6.4 |
| 4 | 1 | `mainInfo` | main display descriptor — §6.4 |
| 5 | 2 | `secValue1` | u16 LE, secondary reading 1 |
| 7 | 2 | `secValue2` | u16 LE, secondary reading 2 |
| 9 | 2 | `mainValue` | u16 LE, **main reading** |

Note the ordering: the descriptor bytes run *secondary, secondary, main*, while
the value words run *secondary, secondary, main*. The main reading is the
**last** field, at offsets 9–10.

### 6.2 `gearStatus` bit field

```
 bit   7   6   5   4   3   2   1   0
     ┌───────┬───────────┬───────────┐
     │ mode  │   range   │ function  │
     └───────┴───────────┴───────────┘
```

| Bits | Field | Extraction |
|---|---|---|
| 2:0 | **function** | `gearStatus & 0x07` |
| 5:3 | **range** | `(gearStatus >> 3) & 0x07` |
| 7:6 | **mode** | `(gearStatus >> 6) & 0x03` |

**function** — which measurement the meter is performing:

| Value | Function | Chinese label in app |
|---|---|---|
| 0 | Voltage | 电压 |
| 1 | Current | 电流 |
| 2 | Resistance | 电阻 |
| 3 | Capacitance | 电容 |
| 4 | Diode / Continuity | 二极管/蜂鸣器 |
| 5 | Frequency / Temperature | 频率/温度 |

**mode** — meaning depends on function:

| Function | mode 0 | mode 1 | mode 2 |
|---|---|---|---|
| 0 Voltage | DC | AC | AC+DC |
| 1 Current | DC | AC | AC+DC |
| 2 Resistance | resistance | (alternate — see §8) | — |
| 3 Capacitance | capacitance | — | — |
| 4 Diode/Continuity | **diode** (unit V) | **continuity** (unit Ω) | — |
| 5 Freq/Temp | **frequency** | **temperature** | — |

> **DM40A does not support AC+DC.** The app explicitly rejects mode 2 on model
> `0x41` with the message "DM40A不支持AC+DC功能" ("DM40A does not support AC+DC")
> and refuses to send the command. Enforce this client-side; the device's
> behaviour if commanded anyway is untested.

**range** — index into a function-specific table, §6.6.

### 6.3 `status` bit field

```
 bit    7      6       5     4     3      2   1   0
     ┌──────┬──────┬───────┬─────┬──────┬───────────┐
     │ HOLD │ LOCK │  REL  │  ?  │ CHG  │  battery  │
     └──────┴──────┴───────┴─────┴──────┴───────────┘
```

| Bits | Field | Extraction |
|---|---|---|
| 2:0 | battery level | `status & 0x07` |
| 3 | charging | `(status >> 3) & 1` |
| 4 | *unknown* | never read by the app — see §8 |
| 5 | REL active | `(status >> 5) & 1` |
| 6 | screen locked | `(status >> 6) & 1` |
| 7 | HOLD active | `(status >> 7) & 1` |

Battery level is a 3-bit value used to pick a battery icon; the app maps it to
one of several images. The exact number of levels in use was not determined.

### 6.4 `mainInfo` / `secInfo1` / `secInfo2` bit field

All three descriptor bytes share an identical layout:

```
 bit   7   6   5   4   3   2   1   0
     ┌───────┬───────┬───────────┬───┐
     │ mode  │ unit  │   power   │ ± │
     └───────┴───────┴───────────┴───┘
```

| Bits | Field | Extraction |
|---|---|---|
| 0 | **sign** | `info & 1` — 1 means negative |
| 3:1 | **power** | `(info >> 1) & 0x07` — decimal exponent |
| 5:4 | **unit** | `(info >> 4) & 0x03` — index into unit table |
| 7:6 | **mode** | `(info >> 6) & 0x03` |

Note this `mode` is a per-reading descriptor and is **distinct** from the
`gearStatus` mode, though they usually agree.

The initial/invalid value for all three descriptors is `0xFF`.

### 6.5 Converting a raw reading to a value

For the main display:

```js
const sign  =  mainInfo        & 0x01;
const power = (mainInfo >> 1)  & 0x07;
const unitI = (mainInfo >> 4)  & 0x03;

// Overload check FIRST — see below
if (mainValue === 0xFFFF) {
  display = "OL";                       // over-limit / out of range
} else {
  let v = mainValue / Math.pow(10, power);
  if (sign) v = -v;
  display = v.toFixed(power) + " " + unitFor(gearFunction, gearMode, unitI);
}
```

**Overload.** `mainValue === 0xFFFF` means the reading is over-range. The app
renders the literal string `"0L"` (a stylised "OL" in the app's font) followed
by the unit. Check this **before** any arithmetic — `0xFFFF` is otherwise a
valid-looking 65535 count.

**Unused slots — descriptor `0x00`.** A descriptor byte of **0** means the slot
carries no reading. Both `get_SecValueStr*` and `get_SecValueUnitStr*` test the
descriptor first and return an empty string when it is zero, blanking the value
*and* the unit. This is the normal state of both secondary slots in a mode that
has no sub-readings (plain DC voltage, for example), where the meter sends
`secInfo = 0x00` alongside `secValue = 0xFFFF`.

Treating `0xFFFF` as overload without checking the descriptor first therefore
produces a phantom "OL mV" on both secondary rows whenever they are idle —
`mV` because unit index `(0x00 >> 4) & 3` is 0. **Check the descriptor before
the value.**

Note the asymmetry in the vendor app: `get_SecValueStr2` tests `secValue2 ==
0xFFFF` and renders "OL", but `get_SecValueStr1` has no such test and would
format `65535 / 10^power` as a literal reading. That looks like an oversight; a
clean implementation should apply the overload check to both secondary slots,
but only after the descriptor test.

Ordering that satisfies all three cases:

```js
if (info === 0x00) return BLANK;        // slot unused — no value, no unit
if (raw === 0xFFFF) return OVERLOAD;    // slot active but over-range
return value(raw, info);                // normal reading
```

One ambiguity worth recording: `info === 0` is also what a genuine reading of
"sign +, power 0, unit index 0, mode 0" would encode. The vendor app treats that
bit pattern as "unused" unconditionally, so the meter evidently never emits it
for a live slot. Follow the app here.

**Decimal places.** The `power` field doubles as the display precision: a reading
with `power = 3` is formatted with exactly 3 decimal places. So `power` is not
merely a scale factor, it is the count resolution — the raw `mainValue` is the
meter's count reading and `power` places the decimal point.

Example: function = 0 (voltage), `mainValue = 4231`, `mainInfo = 0b00_01_011_0`
→ sign 0, power 3, unit 1 → `4231 / 10³ = 4.231`, unit `V` → **"4.231 V"**.

### 6.6 Unit tables

Selected by `gearStatus.function`, indexed by the descriptor's `unit` field:

| Function | Unit table |
|---|---|
| 0 Voltage | `["mV", "V"]` |
| 1 Current | `["uA", "mA", "A"]` |
| 2 Resistance | `["Ω", "KΩ", "MΩ"]` |
| 3 Capacitance | `["nF", "uF", "mF"]` |
| 4 Diode/Continuity | mode 0 → `"V"`; mode 1 → `"Ω"` (not indexed) |
| 5 Freq/Temp | mode 0 → `["Hz", "KHz", "MHz"]`; mode 1 → `["℃", "℉"]` |

Anything outside this set renders as `"*"`.

### 6.6.1 Secondary slots use a *different* unit table

⚠️ **The secondary slots do not share the main display's unit table.** They carry
sub-measurements that are unrelated to the main function — on AC volts the meter
puts **frequency** in slot 1 and **duty cycle** in slot 2. Applying the main
table to them mislabels a 60 Hz reading as "60 mV".

The vendor app makes this visible in its asset naming. Main units come from a
4-digit family `dm40unit{function}{unit}.png`; secondary units come from a
6-digit family `dm40unit{function}{mode}{unit}.png`, plus one hardcoded asset
`dm40unit000001.png` which — despite a name that looks like function 0 / mode 0 /
unit 1 — is the **percent** glyph. The filenames cannot be decoded as a formula;
the mapping below was read from `get_SecValueUnitStr1` and
`get_SecValueUnitStr2` and confirmed against the rendered images.

| Function | Mode | Main | Slot 1 | Slot 2 |
|---|---|---|---|---|
| 0 Voltage | DC | mV / V | mV / V | mV / V |
| 0 Voltage | **AC** | mV / V | **Hz / kHz / MHz** | **%** |
| 0 Voltage | **AC+DC** | mV / V | **mV / V** | **mV / V** |
| 1 Current | DC | µA / mA / A | µA / mA / A | % |
| 1 Current | **AC** | µA / mA / A | **Hz / kHz / MHz** | **%** |
| 1 Current | **AC+DC** | µA / mA / A | **µA / mA / A** | **µA / mA / A** |
| 2 Resistance | — | Ω / kΩ / MΩ | Ω / kΩ / MΩ | Ω / kΩ / MΩ |
| 3 Capacitance | — | nF / µF / mF | nF / µF / mF | nF / µF / mF |
| 4 Diode | diode | V | V | Ω |
| 4 Diode | continuity | Ω | — | — |
| 5 Freq/Temp | frequency | Hz / kHz / MHz | Hz / kHz / MHz | % |
| 5 Freq/Temp | **temperature** | °C / °F | **°C** | **°F** |

**AC+DC secondaries carry the AC and DC components.** In AC+DC coupling the main
display shows the combined value and the two secondary slots break out the
separate AC and DC parts — both in the *main* unit, not frequency or duty cycle.
This falls out of the app's dispatch: every case that is not explicitly handled
drops through to a **4-digit** `dm40unit{function}{unit}.png` name, which is the
main unit table. Mode 2 is never handled explicitly, so it always lands there.

**Temperature shows both scales at once.** Slot 1 is hardcoded to the °C asset
and slot 2 to the °F asset, ignoring the descriptor's unit index in both cases.

Where a slot resolves to a unit *table*, it is indexed by that slot's own
descriptor unit bits. The special-cased strings (`%`, `Ω`, `°C`, `°F`) are fixed
and carry no index.

Selection logic, in order:

```js
// slot 1
if ((fn === 0 || fn === 1) && gearMode === 1) return FREQ[unitIdx];  // AC -> frequency
if (fn === 5 && gearMode === 1)               return "°C";
if (gearMode === 1)                           return "";             // no unit asset
return mainUnit(fn, gearMode, unitIdx);                              // incl. AC+DC

// slot 2
if (fn === 1 && gearMode === 0)               return "%";            // DC current
if ((fn === 0 || fn === 1) && gearMode === 1) return "%";            // AC
if (fn === 4)                                 return gearMode === 0 ? "Ω" : "";
if (fn === 5)                                 return gearMode === 1 ? "°F" : "%";
return mainUnit(fn, gearMode, unitIdx);                              // incl. AC+DC
```

The trailing `mainUnit(...)` in each branch is the 4-digit fallback and is what
makes AC+DC work. Omitting it — treating unhandled cases as "no unit" — silently
blanks the AC and DC component readings.

Note the mode used here is the **gear** mode from `gearStatus`, not the
descriptor's own mode bits.

**AC+DC slot order — confirmed on hardware: slot 1 is AC, slot 2 is DC.**

Nothing in the payload distinguishes them, and the vendor app does not label
them either, so this cannot be derived from the APK. It was established by
observation against a real DM40:

| Slot | Payload offset | AC+DC component |
|---|---|---|
| 1 | `secValue1` @ 5–6, `secInfo1` @ 2 | **AC** |
| 2 | `secValue2` @ 7–8, `secInfo2` @ 3 | **DC** |

A client should label these in its own UI, since the reading is meaningless
without knowing which component it is.

### 6.7 Range tables

The range label depends on `devModelIndex` (§4.1), because full-scale counts
differ between models: DM40A = 4000, DM40B = 5000, DM40C = 6000.

**Voltage** (`function = 0`), indexed by `range`:

| range | DM40A | DM40B | DM40C |
|---|---|---|---|
| 0 | 400 mV | 500 mV | 600 mV |
| 1 | 4 V | 5 V | 6 V |
| 2 | 40 V | 50 V | 60 V |
| 3 | 400 V | 500 V | 600 V |
| 4 | 1000 V | 1000 V | 1000 V |
| 5 | AUTO | AUTO | AUTO |
| 6 | AUTO+ | AUTO+ | AUTO+ |

**Current** (`function = 1`):

| range | DM40A | DM40B | DM40C |
|---|---|---|---|
| 0 | 400 µA | 500 µA | 600 µA |
| 1 | 4000 µA | 5000 µA | 6000 µA |
| 2 | 40 mA | 50 mA | 60 mA |
| 3 | 400 mA | 500 mA | 600 mA |
| 4 | 4 A | 5 A | 6 A |
| 5 | 10 A | 10 A | 10 A |
| 6 | AUTO | AUTO | AUTO |
| 7 | AUTO+ | AUTO+ | AUTO+ |

**Resistance** (`function = 2`):

| range | DM40A | DM40B | DM40C |
|---|---|---|---|
| 0 | 400 Ω | 500 Ω | 600 Ω |
| 1 | 4 KΩ | 5 KΩ | 6 KΩ |
| 2 | 40 KΩ | 50 KΩ | 60 KΩ |
| 3 | 400 KΩ | 500 KΩ | 600 KΩ |
| 4 | 4 MΩ | 5 MΩ | 6 MΩ |
| 5 | 40 MΩ | 50 MΩ | 60 MΩ |
| 6 | AUTO | AUTO | AUTO |

**Capacitance, Diode/Continuity, Frequency/Temperature** (`function = 3, 4, 5`)
have no manual ranges — the app displays "AUTO" unconditionally and sends
`range = 0`.

Note the asymmetry: **`AUTO` sits at a different index for each function** —
voltage 5, current 6, resistance 6. Do not hardcode a single "auto" constant.
`AUTO+` (voltage 6, current 7) appears to be an extended auto-range mode; its
precise behaviour was not determined from the app.

---

## 7. Changing function, range, and mode

All three are set by a **single** command, `0x06` SetGear, carrying one byte
packed exactly like `gearStatus`:

```js
const gearByte = (setGearFunction & 0x07)
               | ((setGearRange & 0x07) << 3)
               | ((setGearMode  & 0x03) << 6);
// send function 0x06 with payload [gearByte]
```

The device echoes function `0x06` on success. The new state is not reflected in
your model until the next `0x09` poll returns an updated `gearStatus` — treat
the echo as "accepted", not "applied", and re-read.

### 7.1 Changing the range within the current function

This is the simple case, and matches what the app's range buttons do:

1. Take the current `gearStatus` from the most recent poll.
2. Extract current function and mode.
3. Substitute the new range index.
4. Send `0x06`.

```js
async function setRange(rangeIndex) {
  const fn   =  gearStatus       & 0x07;
  const mode = (gearStatus >> 6) & 0x03;
  await sendGear(fn, rangeIndex, mode);
}
```

Validate `rangeIndex` against the table for `fn` in §6.7 — the tables have
different lengths (voltage 7, current 8, resistance 7) and out-of-range indices
were not tested against hardware.

### 7.2 Changing AC/DC mode

Same pattern, substituting mode instead of range:

```js
async function setMode(mode) {           // 0 = DC, 1 = AC, 2 = AC+DC
  const fn    =  gearStatus       & 0x07;
  const range = (gearStatus >> 3) & 0x07;

  if (mode === 2 && devModel === 0x41)   // DM40A
    throw new Error("DM40A does not support AC+DC");

  await sendGear(fn, range, mode);
}
```

Meaningful only for functions 0 (voltage) and 1 (current). For function 4 and 5
the mode field selects sub-function rather than coupling (§7.4).

### 7.3 Changing the measurement function

More involved, because the app applies per-function defaults rather than
blindly reusing the current range and mode. The logic, transcribed from
`SetGearVoltage`, `SetGearCurrent`, `SetGearResistance`, `SetGearCapacitance`,
`SetGearDiode`, `SetGearFT`:

**Switching to Voltage (function 0):**
- If the meter is **not already** on voltage: `range = 6` (AUTO+), `mode = 0` (DC).
- If it **is** already on voltage: keep the current range; on **DM40A only**,
  toggle mode between DC and AC (`mode = (currentMode === 0) ? 1 : 0`). On
  DM40B/C the mode cycles through the three couplings.

**Switching to Current (function 1):**
- If not already on current: `range = 7` (AUTO+), `mode = 0` (DC).
- If already on current: keep range; DM40A toggles DC↔AC as above.

**Switching to Resistance (function 2):**
- If not already on resistance: `range = 6` (AUTO), `mode` = the app's retained
  `Rgearmode` value.
- If already on resistance: keep range and toggle `Rgearmode` between 0 and 1.

**Switching to Capacitance (function 3):** `range = 0`, `mode = 0`.

**Switching to Diode/Continuity (function 4):** `range = 0`. If not already on
this function, `mode = 0` (diode). If already on it, toggle mode 0↔1
(diode ↔ continuity).

**Switching to Frequency/Temperature (function 5):** `range = 0`. If not already
on this function, `mode = 0` (frequency). If already on it, toggle mode 0↔1
(frequency ↔ temperature).

The unifying pattern: **pressing a function button that is already active
cycles that function's sub-mode.** A clean implementation is free to expose
function and mode as independent controls instead — the wire format supports it.

### 7.4 Practical sequence

Because commands must be serialised against the poll loop (§4.4):

```
 1. acquire transaction lock (pauses polling)
 2. send 0x06 with the packed gear byte
 3. await echo of function 0x06, or 3000 ms timeout
 4. release lock (polling resumes)
 5. next 0x09 response carries the updated gearStatus — update UI from that
```

Do not optimistically update your UI from the value you sent. The meter may
clamp or reject a range; the authoritative state is always the next poll.

---

## 8. Open questions and known defects

### 8.1 Requires hardware to resolve

| # | Item | Status | Notes |
|---|---|---|---|
| 1 | Service / characteristic UUIDs | **Resolved** (§1.1.1) | Obtained from third-party HCI captures, not from the APK. Worth confirming on your own unit. |
| 2 | `status` bit 4 | Open | The vendor app reads bits 0–3 and 5–7 but never bit 4. The independent `DM40GUI` decode also omits it. Likely reserved. |
| 3 | Resistance mode 1 | Open | `SetGearResistance` toggles a retained `Rgearmode` between 0 and 1, but the unit table for function 2 does not vary by mode. Possibly conductance; unresolved. |
| 4 | `AUTO+` semantics | Open | Distinct range index from `AUTO` for voltage and current. Behavioural difference not determinable statically. |
| 5 | Function `0x01` (`Conn`) | Open | Constant defined, never called in the DM40 path. Payload unknown. |
| 6 | Battery level scale | **Likely resolved** | 3 bits available; `DM40GUI` reports the meter renders **0–5 bars**. |
| 7 | Out-of-range `range` index | Open | Device behaviour when commanded an index beyond the table is untested. |
| 8 | REL bit confirmation | Open | Bit 5 = REL is derived from the vendor app's `get_Rel`. `DM40GUI` does not decode this bit, so it is single-sourced. |
| 9 | AC+DC slot ordering | **Resolved** | Slot 1 = AC component, slot 2 = DC component. Confirmed on hardware; not derivable from the APK. |
| 10 | Frequency-mode slot 1 | Open | Resolves to a 4-digit `dm40unit05{unit}` name with no matching asset in the APK. Rendered as Hz here, but the slot is probably unused. |
| 11 | Descriptor `0x00` collision | Open | `info == 0` marks a slot unused, but is also the encoding for a legitimate "+, power 0, unit 0, mode 0" reading. The app never disambiguates, implying the meter never sends it for a live slot. |

### 8.2 Apparent defects in the vendor app — do not reproduce

**Defect A — temperature/continuity offset arithmetic.**

In `get_MainValueR` and `get_MainValueStr`, when
`gearFunction == 4 && gearMode == 1 && mainInfo.mode == 2`, the app computes:

```csharp
(MainValue + 0xFFF0) / Math.Pow(10, MainValuePower)
```

`MainValue` is a `ushort` but the expression is evaluated in `int`, so
`0xFFF0` (65520) is **added**, not applied as a 16-bit wraparound. The evident
intent was `(short)(mainValue - 16)` — a signed offset of −16. As written the
branch yields values around +65520, which cannot be correct for any physical
reading.

**Recommendation:** treat this branch as unverified. Implement it as a signed
16-bit offset (`((mainValue - 16) << 16 >> 16) / 10^power`) *only* if hardware
testing confirms an offset is needed at all; otherwise fall through to the
normal conversion path.

**Defect B — checksum accumulator width.**

`CalculationCheck` accumulates into a `ushort` and the result is taken mod 256,
so any frame long enough to overflow 65535 in the running sum would compute a
wrong checksum. At the DM40's maximum payload (20 bytes) this cannot occur, so
it is latent rather than active. Accumulating in a wider type, or masking to 8
bits each step, is equivalent for all real frames and safer.

---

## 9. Implementation checklist

For a Web Bluetooth client, in dependency order:

1. `requestDevice()` with `namePrefix: 'DM40'` and `optionalServices: [0xfff0]`;
   get characteristics `0xfff1` (notify) and `0xfff3` (write) — §1.1.1.
2. `startNotifications()` and attach the frame parser (§3.2) to
   `characteristicvaluechanged`.
3. Implement frame encode/decode and the checksum (§2).
4. Wrap all transactions in an async mutex — one command in flight, 3000 ms
   timeout, match on echoed function code (§3.3, §4.4).
5. Registration handshake to `0xFFFF`; capture model byte (§4.1). Alternatively
   use `0x08` GetBleName, which `DM40GUI` uses for identification.
6. Poll `0x09` every 200 ms; decode per §6. Check `mainValue === 0xFFFF` for
   overload before converting.
7. Layer controls on top: SetGear (§7), HOLD/REL/Auto (§5.4).

A minimal read-only client needs only steps 1–6; the control surface in §7 can
follow once the read path is verified against the vendor app's displayed values.

---

## 10. Cross-validation against independent work

### 10.1 Source

**`maj113/DM40GUI`** — a Windows GUI for DM40/EL15 over BLE, reverse-engineered
from **BLE HCI captures**, i.e. by observing live traffic rather than reading
the app's code. MIT licensed.

> https://github.com/maj113/DM40GUI

That makes it a genuinely independent derivation: this document comes from
static analysis of the vendor app's CIL, that project comes from wire captures.
Where the two agree, confidence is high.

### 10.2 Points of agreement

| Item | This document | `DM40GUI` | Agree |
|---|---|---|---|
| TX flag / device ID prefix | `AF 05 03` | `AF 05 03` | ✅ |
| RX flag prefix | `DF 05 03` | `DF 05 03` | ✅ |
| Checksum | `(256 − Σ) & 0xFF` | `(−sum(first 5)) & 0xFF` | ✅ identical |
| Whole-frame validity test | Σ all bytes ≡ 0 (mod 256) | `(sum(all) & 0xFF) == 0` | ✅ |
| GetRealTimeData | fn `0x09` | `af 05 03 09 00 40` | ✅ checksum matches §2.6 |
| GetBleName | fn `0x08`, 20-byte reply | `af 05 03 08 00 41`, prefix `DF 05 03 08 14` (`0x14` = 20) | ✅ |
| HOLD / AUTO / REL | fn `0x04` / `0x03` / `0x05`, 1-byte arg | same | ✅ |
| SetGear | fn `0x06`, packed byte | same | ✅ |
| `gearStatus` at payload[0] | function / range / mode | "mode and range flag" at frame[5] | ✅ |
| `status` at payload[1] | bits per §6.3 | battery `&0x07`, charge `&0x08`, lock `&0x40`, hold `&0x80` | ✅ |
| `mainValue` at payload[9:11] | u16 LE | frame[14:16] u16 LE (same bytes) | ✅ |
| Descriptor bytes | payload[2..4] | frame[−10, −9, −8] (same bytes) | ✅ |

The strongest confirmation is the **gear byte packing** (§6.2). `DM40GUI`'s
captured commands decode exactly under this document's bit layout:

| Their command | Byte | function | range | mode | Meaning |
|---|---|---|---|---|---|
| `CMD_CAP` | `0x03` | 3 | 0 | 0 | Capacitance |
| `CMD_DIODE` | `0x04` | 4 | 0 | 0 | Diode |
| `CMD_CONT` | `0x44` | 4 | 0 | **1** | Continuity |
| `CMD_HZ` | `0x05` | 5 | 0 | 0 | Frequency |
| `CMD_TEMP` | `0x45` | 5 | 0 | **1** | Temperature |

Two independently-derived facts landing on the same bit boundaries — including
the mode-toggle pairs diode↔continuity and frequency↔temperature — is about as
good as static confirmation gets short of hardware.

### 10.3 Where this document goes further

`DM40GUI`'s README does not cover:

- the **`0x00` registration handshake** to `dev = 0xFFFF` and the device-type /
  model-byte reply (§4.1);
- the **model-dependent range tables** (4000/5000/6000 counts, §6.7);
- the `mainValue == 0xFFFF` **overload sentinel** (§6.5);
- **REL** at `status` bit 5 (§6.3);
- the **decimal-exponent / precision** semantics of the descriptor bytes (§6.4);
- **AC+DC being rejected on DM40A** (§6.2);
- the 3000 ms / 10 ms **response-matching** and 200 ms **poll** timings (§3.3, §4.3);
- the per-function **`AUTO` index asymmetry** (§6.7).

### 10.4 Presentational discrepancy — not a conflict

`DM40GUI`'s README describes DM40 command frames as **6 bytes**, `AF 05 03 <cmd>
<arg> <checksum>`. That is accurate only for **zero-payload** commands, where the
byte it calls `<arg>` is in fact the **length** field (`0x00`) and the sixth byte
is the checksum. For commands that carry a payload, its own examples are 6 bytes
with the checksum omitted:

```
CMD_HOLD_ON   af 05 03 04 01 01
                        │  │  └── data[0] = 0x01
                        │  └───── len = 1
                        └──────── function = 0x04
                                  ← checksum (0x43) not shown
```

Under this document's format that frame is 7 bytes: `AF 05 03 04 01 01 43`.
Both descriptions denote the same wire bytes; the README simply doesn't
distinguish the length field from the payload. **Use the §2 layout** — it
generalises to the 20-byte SetBleName frame, which the fixed 6-byte model
cannot express.

### 10.5 EEVblog

EEVblog has substantial DM40 coverage — a full review (blog #1723, December 2025)
and an active forum thread — but no protocol or UUID documentation surfaced in
searching. The site is behind a CrowdSec challenge that blocks automated
fetching, so the forum thread could not be read directly. If you want to check it
by hand:

- https://www.eevblog.com/forum/testgear/alientek-dm40c-multimeter-(dm40a-dm40b-dm40c)/
- https://www.eevblog.com/forum/blog/eevblog-1723-alientek-dm40-multimeteroscilloscope-review/

The review video timestamps Bluetooth functionality at 29:09, though it covers
the app's features rather than the wire protocol.
