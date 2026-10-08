# matrix - startup & development notes

*Teensy 4.0 + ESP32-WROVER-B, 5×5 MX pad grid, 5 pots, WiFi web UI*

## 1. Project summary

A 25-key (5×5) hardware synthesizer. The **Teensy** owns everything real-time: key scanning, potentiometers, the audio engine and the DAC output. The **ESP32** is a wireless bridge only: it hosts a WiFi access point, a web UI and preset storage, and talks to the Teensy over UART. Because the pads go straight to the Teensy, WiFi jitter never affects playing feel. The phone is for presets and sound design, not for performance.

## 2. Architecture

```
[Phone browser] --WiFi/WebSocket (JSON)--> [ESP32-WROVER-B]
                                              | UART2 (3.3V, 115200+ baud)
                                              v
[25x MX switches + 25 diodes] --> [Teensy 4.0] <-- [5x pots]
                                              | I2S (3 pins)
                                              v
                                     [PT8211 DAC shield] --> audio out
```

| Layer | Owner | Responsibility |
| --- | --- | --- |
| Input | Teensy | Matrix scan, debounce, pot reading and smoothing |
| Audio | Teensy | Oscillators, envelopes, filter, mixer, I2S output |
| Config/UI | ESP32 | WiFi AP, web server, WebSocket, preset storage, OTA |
| Link | Both | Framed binary protocol over UART |

## 3. Component breakdown

| # | Component | Qty | Role | Notes |
| --- | --- | --- | --- | --- |
| 1 | Teensy 4.0 | 1 | Audio, input, real-time control | 600 MHz Cortex-M7, 3.3V logic (pins **not** 5V tolerant) |
| 2 | PT8211 DAC shield (you wrote "PT28211"; please confirm) | 1 | I2S to analog audio | Supported by the Audio Library as `AudioOutputPT8211`. 16-bit, no headphone amp: plan for AC-coupling and a buffer if you drive headphones |
| 3 | ESP32-WROVER-B dev board | 1 | WiFi, web UI, preset storage | 3.3V logic; PSRAM on GPIO16/17 (**do not use those pins**) |
| 4 | Cherry MX switches (2-pin) | 25 | Pads | Binary on/off, so no velocity sensing |
| 5 | 1N4148 diodes | 25 | One per switch, prevents ghosting | See wiring note below |
| 6 | 10k linear pots (B10K) | 5 | Live parameter control | Wire to the Teensy 3.3V, never 5V |
| 7 | 3.5mm audio jack (and output buffer if needed) | 1 | Audio out |  |
| 8 | Hookup wire/headers, 5V supply or USB |  |  | Power budget in section 6 |
| 9 | (Later) SK6812 / WS2812 LEDs | up to 25 | Pad lighting | Reserve a data pin now |

### Teensy 4.0 vs 4.1

**Stay with the 4.0.** Both have the same 600 MHz CPU and 1 MB of RAM. The 4.1 adds 8 MB flash (vs 2 MB), an SD slot, more pins and Ethernet/PSRAM pads. This project needs roughly 20 pins and no SD, and presets live on the ESP32, so the 4.1's extras go unused. Consider the 4.1 only if you later want sample playback from SD or large wavetables. Your shield would also need adapting, which is not worth it for this build.

## 4. Pin map (proposed)

### Teensy 4.0

| Function | Pins | Notes |
| --- | --- | --- |
| I2S to PT8211 | 7 (data), 20 (WS/LRCLK), 21 (BCLK) | Audio Library defaults for `AudioOutputPT8211`; verify against the Teensy pinout card |
| UART to ESP32 | 0 (RX1), 1 (TX1) | `Serial1` |
| Matrix rows (inputs, `INPUT_PULLUP`) | 2, 3, 4, 5, 6 |  |
| Matrix columns (outputs, driven LOW one at a time) | 8, 9, 10, 11, 12 |  |
| Pots | 14, 15, 16, 17, 18 (A0–A4) |  |
| LED data (reserved) | 22 | Unused for now |
| Free | 13 (onboard LED), 19, 23 |  |

### ESP32-WROVER-B

| Function | GPIO | Notes |
| --- | --- | --- |
| UART2 RX (from Teensy TX1) | 25 | Remapped with `Serial2.begin(115200, SERIAL_8N1, 25, 26)` |
| UART2 TX (to Teensy RX1) | 26 |  |
| Avoid | 6–11, 16, 17, 0, 2, 5, 12, 15 | Flash, PSRAM and strapping pins |

Cross-connect TX to RX, and **tie grounds together**.

### Matrix wiring

Each switch gets a series 1N4148. Orient it so current flows from the row to the column: **cathode (stripe) on the column side**. Scan by driving one column LOW while the others are high-Z (`INPUT`), then read the five rows. Debounce at about 5 ms (e.g. Bounce2 logic or a per-key timestamp).

## 5. Potentiometers

Proposed default mapping: **cutoff, resonance, attack, release, volume** (changeable). Notes:

- Read on the Teensy with `analogReadResolution(12)` plus averaging, and smooth with an exponential filter or `ResponsiveAnalogRead` to kill jitter.
- The ESP32 ADC is noisy and ADC2 is unusable while WiFi is on, which is why the pots go on the Teensy.
- **Pickup (catch) mode:** when a preset is loaded from the phone, pot positions will not match the stored values. Ignore a pot until it crosses the stored value, or the sound will jump.
- Report pot changes up to the ESP32 so the web UI sliders can follow in real time.

## 6. Power and signal integrity

- Teensy 4.0 logic is 3.3V and its 3.3V regulator is limited, so **do not power the ESP32 from the Teensy 3.3V pin**. WiFi transmit peaks can reach several hundred mA.
- Simplest approach: one clean 5V source (USB charger, 1A or more) feeding the ESP32 dev board's 5V input and the Teensy's USB/VIN. If you ever connect the Teensy USB to a computer while externally powered, handle the VUSB/VIN isolation per the Teensy documentation.
- Add a 100 nF cap near each chip and keep audio wiring away from the WiFi antenna side of the ESP32. WiFi radio noise is the most common cause of hiss in this kind of build.
- LEDs later: 25 RGB LEDs can draw up to about 1.5 A at full white, so they will need their own 5V rail and a level shifter or series resistor on the data line.

## 7. Audio engine plan (Teensy)

The chain below uses the Teensy Audio Library (128-sample blocks, \~2.9 ms at 44.1 kHz):

```
8x AudioSynthWaveform --> 8x AudioEffectEnvelope --> AudioMixer4 x3 (cascaded)
   --> AudioFilterStateVariable --> AudioMixer4 (master gain) --> AudioOutputPT8211
```

- Start with 8 voices (round-robin voice allocation). The Teensy 4.0 has plenty of CPU headroom, so this can grow.
- Call `AudioMemory(30)` to begin with and watch `AudioProcessorUsageMax()`.
- There is no parameter struct in the library: updates are setter calls (`filter.frequency()`, `env.attack()`, `wave.begin()`). Wrap multi-parameter updates in `AudioNoInterrupts()` / `AudioInterrupts()`.
- Envelope times apply at the next note-on or stage change. Abrupt cutoff or gain jumps can click, so smooth those on the Teensy.
- Pad mapping: 25 pads is two octaves plus one note in a chromatic layout. A scale-based or isomorphic layout is possible later.

## 8. UART protocol v0.1 (bidirectional)

Frame: `[0xAA][type][len][payload...][checksum]`, where checksum is the XOR of type, len and all payload bytes. Because 0xAA can appear in payloads, the receiver resyncs using length plus checksum: on a bad checksum, discard the first byte and hunt for the next 0xAA.

| Type | Direction | Payload |
| --- | --- | --- |
| 0x01 Preset set | ESP32 to Teensy | 8 params × `uint16` little-endian (16 bytes) |
| 0x02 Single param set | ESP32 to Teensy | `uint8` id + `uint16` value |
| 0x03 State request | ESP32 to Teensy | none |
| 0x10 State report | Teensy to ESP32 | 8 params × `uint16` |
| 0x11 Param changed (pot moved) | Teensy to ESP32 | `uint8` id + `uint16` value |
| 0x20 Pad event (optional, for UI) | Teensy to ESP32 | `uint8` pad, `uint8` state |
| 0x30 Ping/ack | Both | none |

Param IDs: 0 waveform, 1 cutoff, 2 resonance, 3 attack, 4 decay, 5 sustain, 6 release, 7 gain.

Using `uint16` (0–65535) instead of 8 bits gives smooth cutoff control. The Teensy maps values to units: cutoff exponentially (about 20 Hz to \~12 kHz), resonance to roughly 0.7–5.0, and times in milliseconds. Pad note events never cross this link, since the Teensy handles pads locally.

On the Teensy, no interrupt code is needed: poll `Serial1.available()` in `loop()` and run a small state machine. Start at 115200 baud (a 20-byte frame takes about 2 ms); increase later if needed.

## 9. ESP32 firmware plan

- **WiFi mode:** run the ESP32 as its own access point (the phone joins it directly, no router required). Use the default 192.168.4.1; mDNS names are unreliable on Android.
- **Server:** ESPAsyncWebServer with AsyncTCP (use the actively maintained ESP32Async forks) plus a WebSocket endpoint at `/ws`.
- **Web UI:** static HTML/JS/CSS stored in LittleFS: sliders for the 8 parameters, a waveform selector, and preset save/load/rename.
- **Messages:** JSON text frames via ArduinoJson, e.g. `{"t":"param","id":1,"v":32768}` and `{"t":"preset","name":"Pad 1","p":[...]}`.
- **Preset storage:** JSON files in LittleFS on the ESP32 (which has far more flash than the Teensy 4.0 has spare), loaded and pushed to the Teensy on demand.
- **OTA:** ArduinoOTA or ElegantOTA for firmware updates without cables.
- **Latency expectation:** WiFi jitter is typically several to tens of ms, which is fine for sound design and preset recall but not for playing notes from the phone.

## 10. Software and libraries

| Side | Libraries/tools |
| --- | --- |
| Teensy | Teensyduino, Audio Library (with `AudioOutputPT8211`), Bounce2 (or custom matrix scan), ResponsiveAnalogRead |
| ESP32 | Arduino-ESP32 core, ESPAsyncWebServer + AsyncTCP (ESP32Async forks), ArduinoJson, LittleFS, ArduinoOTA |

## 11. Corrections to the earlier design notes

1. GPIO16/17 are **not** usable for UART2 on the WROVER-B because they are used by PSRAM. Remap UART2 to free pins (section 4).
2. Custom BLE GATT does not give meaningfully lower latency than BLE MIDI. Moot now, since you chose WiFi.
3. SysEx data bytes must be 7-bit, which matters only if BLE MIDI is revisited.
4. The Teensy needs no UART interrupt handler and has no parameter struct; use polling and Audio Library setters.

## 12. Build phases

1. **Bring-up:** Teensy plays a fixed tone through the PT8211, confirming I2S pins and output level.
2. **Matrix:** wire and test the 5×5 grid with the diodes, serial-print key events.
3. **Voices:** 8-voice polyphony with envelopes, filter and master gain driven by the pads.
4. **Pots:** read, smooth and map 5 pots to parameters, with pickup mode.
5. **UART link:** loopback test, then implement the framing with checksum and resync.
6. **ESP32 AP and web UI:** serve the page, WebSocket param control forwarded to the Teensy.
7. **Presets:** save/load in LittleFS, state sync both ways (pots to UI, UI to Teensy).
8. **Polish:** LEDs, OTA, enclosure, noise tuning.

## 13. Risks

- WiFi radio noise coupling into the audio path (mitigate with layout, decoupling, grounding, and optionally pausing heavy traffic).
- Brown-outs if the ESP32 is powered from an undersized source.
- Pot jitter and zipper noise on the filter (smoothing on both).
- Single-supply PT8211 output level and DC offset (verify with a scope or multimeter before connecting to an amp).

## 14. Assumptions and open questions

**Assumed unless you say otherwise:** Teensy 4.0, USB or 5V powered, pots on the Teensy mapped as in section 5, one 5×5 matrix scanned by the Teensy, fixed note velocity, no LEDs at first, presets stored on the ESP32.

**Please confirm or decide:**

1. Is it the **PT8211** DAC? Is the shield mono or stereo, and what output stage does it have?
2. Mono or stereo output, and do you need headphone drive or line level only?
3. Which note layout for the 25 pads (chromatic, scale-based, or grid/isomorphic), and do you want an octave shift?
4. Should the pots also have shifted functions (e.g. hold a pad to change what a pot controls)?
5. How many oscillator layers per voice (single, detuned pair, sub-oscillator)?
6. Do you also want MIDI (USB MIDI is built into the Teensy) for playing from a computer?
7. Preferred enclosure/PCB approach: hand-wired, perfboard, or a custom PCB (affects hot-swap sockets and connector choices)?
