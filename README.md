# WebDM40

A vibe-coded web-based interface for the Alientek DM40 multimeter (DM40A / DM40B / DM40C) using [Web Bluetooth](https://developer.mozilla.org/en-US/docs/Web/API/Web_Bluetooth_API).

Turn on Bluetooth from the meter's Settings → Other Settings, open the page in Chrome, and click Connect!

## Demo
https://github.com/user-attachments/assets/d0f1f5a4-dec4-40a7-8898-e4d11bc1aab7


If you have a DM40, open Chrome and go to https://scottbez1.github.io/webdm40/ to try it yourself.

## What it does

- Live voltage, current, resistance, capacitance, frequency, temperature, diode, and continuity readings
- Switch functions, ranges, and AC/DC modes from the browser
- HOLD and REL controls
- Sparkline trace with configurable history window (clears on mode change, pauses on HOLD)
- Secondary readings (frequency + duty cycle in AC mode, AC + DC components in AC+DC mode, both °C and °F in temperature mode)
- Configurable poll rate (2 Hz to max) with a live "actual Hz" readout
- Packet inspector for debugging
- Works entirely offline — single HTML file, no dependencies, no build step

## How it was made

Claude determined the protocol from the official Android app (`ATK-XTOOL.apk`). The app is built with .NET MAUI, so there's no Java bytecode to decompile — the logic lives in a managed assembly (`ATKMobileBLE.dll`) stored LZ4-compressed inside an Android AssemblyStore. A custom ECMA-335 metadata parser and CIL disassembler were written to extract and read it.

The BLE service and characteristic UUIDs aren't present anywhere in the APK (the app discovers them by capability rather than by UUID), so those were sourced from [@maj113](https://github.com/maj113)'s [DM40GUI](https://github.com/maj113/DM40GUI) project, which was independently reverse-engineered from BLE HCI packet captures. The two approaches agree on every overlapping point (frame format, checksums, command codes, payload layout), which gives high confidence in the protocol documentation.

The full protocol spec is in [`DM40-BLE-Protocol.md`](DM40-BLE-Protocol.md).

## How it works

The meter uses a vendor-specific BLE GATT service (`0xFFF0`) with separate notify (`0xFFF1`) and write (`0xFFF3`) characteristics. All communication uses a simple framed protocol: a one-byte flag (`0xAF` outbound, `0xDF` inbound), a device ID, a function code, a length-prefixed payload, and a two's-complement checksum.

There's no sequence numbering — responses are matched to requests by echoing the function code, so only one command can be in flight at a time. All transactions go through a single async queue.

The meter is poll-only (it never pushes data), so the app requests a measurement every 200ms by default and decodes the 11-byte response into the main reading, two secondary readings, and status flags. The main value is a raw 16-bit count scaled by a 3-bit decimal exponent from a descriptor byte; `0xFFFF` means overload.

A few things that tripped me up and are worth knowing:

- **TX and RX use different flag bytes** (`0xAF` vs `0xDF`). Using one value for both will fail silently.
- **The secondary displays have their own unit table**, separate from the main display. On AC volts they carry frequency and duty cycle, not voltage. Getting this wrong labels 60 Hz as "60 mV".
- **A descriptor byte of `0x00` means the slot is unused**, not that it has a reading of zero. The meter sends `secInfo = 0x00` with `secValue = 0xFFFF` for idle secondary slots — checking the value before the descriptor produces phantom "OL mV" readings.
- **The AUTO range index is different for each function** (5 for voltage, 6 for current and resistance). There's no universal "auto" constant.
- **DM40A does not support AC+DC mode.** The vendor app blocks it client-side.

## Browser support

Web Bluetooth requires Chrome, Edge, or Opera on desktop or Android. **iOS is not supported** — Safari doesn't implement Web Bluetooth, and all iOS browsers use WebKit underneath, so Chrome on iOS won't work either. The page must be served over HTTPS or from localhost.

## Acknowledgements

- [@maj113](https://github.com/maj113)'s [DM40GUI](https://github.com/maj113/DM40GUI) for the independently-derived BLE captures and GATT UUIDs
- [EEVblog's DM40 review](https://www.eevblog.com/forum/blog/eevblog-1723-alientek-dm40-multimeteroscilloscope-review/) for general DM40 documentation

## License

This project is licensed under Apache v2

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

        http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
