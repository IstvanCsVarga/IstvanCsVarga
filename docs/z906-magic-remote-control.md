# Controlling a Logitech Z906 with an LG OLED G5 Magic Remote — Build / Decision Document

**Author context:** Homelab user in Debrecen, HU. Comfortable with Linux, Home Assistant, ESPHome, Docker, light soldering.
**Research date:** 2026-05-31. **Status:** decision document — flags every uncertain / unconfirmed claim inline.

> **TL;DR — the plan changes.** The original architecture (TV → CEC adapter → controller sniffs CEC volume → IR blaster fires Z906 codes) is the *least* reliable path available to you. Two independent findings flip the design:
> 1. **CEC volume on 2024–2025 LG webOS is contraindicated**, not just unproven. A brand-new LG C4 (2024) with a *real LG soundbar* got **power-only, no volume** over SimpLink/HDMI-CEC; the owner had to fall back to optical + LG Sound Sync. A generic libCEC/ESP32 device is a *worse* bet than that failed real soundbar, not a better one.
> 2. **The Z906's wired-console serial protocol is fully reverse-engineered** and gives you **absolute volume (0–43) + two-way state feedback** — exactly what IR can never provide. This kills the relative-volume drift problem at the source.
>
> **Recommended build:** ESP32 controlling the Z906 over its **DE-15 serial console** (zarpli library), driven by **volume state read from the TV over the Home Assistant LG webOS WebSocket** (`subscribe_volume`). Optical keeps carrying untouched Dolby Digital 5.1 to the Z906; the Z906 attenuates internally as it always has. Absolute-in → absolute-out = **zero drift, no IR line-of-sight, full feedback.** CEC becomes an optional fallback experiment, not the linchpin.

---

## 1. Feasibility verdict (the linchpin — be skeptical)

### 1a. Does current webOS emit usable CEC volume to a *generic* audio device over eARC?

**Verdict: technically possible in the abstract, but unconfirmed on 2024–2025 LG and actively contraindicated. Do not build on it as your primary path.**

What is genuinely confirmed (brand-agnostic, protocol level):

- A device advertising itself as **Audio System (logical address 0x5)** *does* receive `User Control Pressed` volume events from a TV — **but only after it satisfies an audio-status handshake**, and there is a hard gotcha. [libCEC #594 — High for mechanism]
- The TV is the *initiator* of `<User Control Pressed: Volume Up (0x44 0x41) / Down (0x44 0x42) / Mute (0x44 0x43)>` only once **System Audio Mode** is active (`Set System Audio Mode`, opcode 0x72). [Android CEC docs, CEC-O-MATIC — High]
- **The killer gotcha:** the TV *stops sending volume keypresses when it believes the volume is 0*. libCEC by default answers `Give Audio Status` (0x71) with `Report Audio Status` (0x7A) = `VOLUME_STATUS_UNKNOWN` (0x7F), which is *inadequate* — you must report a *plausible non-0x7F volume value* or the stream dries up. libCEC also auto-aborts ARC-start opcodes, breaking clean eARC negotiation. The maintainers effectively call libCEC "unsuitable for audio-device emulation" out of the box. [libCEC #594 — High]

Why "generic is enough" is the wrong question for *your* TV:

- **The decisive negative evidence:** multiple independent owners of a **2024 LG C4 + genuine LG soundbar, both on latest firmware**, report SimpLink gave **power control only, no volume** over HDMI; the same soundbar did volume fine over SimpLink on an *older* LG OLED. Fix was to abandon CEC and use **optical + LG Sound Sync**. [AVForums 2518267, AVS 3317352 — High relevance, Medium credibility]. If a real LG-branded soundbar can't get CEC volume on a 2024 C4, a homemade libCEC/ESP32 audio system is not going to do better.
- **LG ↔ libCEC is independently flaky:** libCEC has logged continuous ~30 s disconnects from LG OLEDs ("FIXME: LG seems to have bugged out"), unresolved. [libCEC #567 — High repo / Medium model-age]
- **Evidence gap:** I could not find a *single* clean report of anyone capturing **LG webOS 2023+ Magic Remote volume keys** as a generic audio system via Pi+libCEC / Pulse-Eight / pyCEC / python-cec / ESP32. The one clean success in the wild is on **Sony**, from 2022. Your exact scenario is **unverified on current LG hardware.**

If you still want a CEC handshake workaround: it's the `Palakis/esphome-native-hdmi-cec` route (you control the `Report Audio Status` reply directly, can return a non-0x7F volume, and can answer ARC opcodes) — see §2 Option C. That's the *only* CEC stack where the handshake is in your hands rather than libCEC's. But expect model-specific trial-and-error and accept it may simply not work on G5 firmware.

### 1b. The alternative: detect volume via the Home Assistant LG webOS integration (no CEC)

**Verdict: this is the stronger signal source — confirmed real-time push — with two caveats you must design around.**

- **Confirmed:** the current `aiowebostv` library (behind HA `webostv`) implements **`subscribe_volume(callback)`** — a *persistent WebSocket subscription* to `ssap://audio/getVolume`, pushing `volume` and `mute` on change. The HA `media_player` is **push-driven** (`register_state_update_callback`), updating `volume_level`, `is_volume_muted`, and `sound_output` — it does **not** poll for state. (Ignore generic "30 s polling" claims; they don't apply to this path.) [aiowebostv `webos_client.py`, HA core `webostv/media_player.py` — High, read directly]. Latency is "effectively real-time per the push design" — **no independent numeric benchmark found** (flag).
- **Caveat 1 — the subscription silently dies.** HA core #107808: the volume "watch" can lose its connection after a network blip and is **not auto-reconnected**, silently, while the rest of the integration keeps working. Workaround today is an HA restart → **you must add your own reconnect/watchdog.** [HA #107808 — High]
- **Caveat 2 — and this is the decisive unknown for the whole project:** when **Sound Out is set to optical / external**, LG remote volume keys historically control only the *internal* speaker, and a Dolby/DTS *bitstream* over optical can't be attenuated at all. In HA, users on external/ARC output report `sound_output`/volume oddities ("No matched extended item: soundOutput", missing slider). [AVForums 2151352, HA #106981/#102956 — Medium]. **The open question: when Sound Out = Optical (feeding 5.1 to the Z906), does `ssap://audio/getVolume` still fire on every Magic Remote keypress?** I could not confirm this on 2024–2025 webOS. **This is the #1 thing to bench-test before buying anything** (see §9).

The happy outcome you're hoping for: in optical-out mode the TV keeps a live 0–100 volume scalar that changes on keypress (the "meaningless OSD" you already anticipated) while the Dolby Digital bitstream passes through untouched. If true, you read that scalar over the WebSocket and map it to the Z906 — clean. If the scalar goes static in optical mode, you fall back to **LG Sound Sync (Optical)** mode or to capturing relative up/down events some other way. **Verify on the actual G5 first.**

---

## 2. Architecture options

All three share the same **insight that reshapes the project**: the Z906 should be driven over its **serial console** (absolute volume + feedback), not by IR. The options differ in *where the volume intent comes from*. Optical always keeps carrying untouched 5.1 to the Z906 — the audio path is never in the loop.

### Option A (RECOMMENDED) — HA webOS WebSocket → ESP32 → Z906 serial

```
LG G5 (Sound Out = Optical) ──optical 5.1──▶ Z906 (amp attenuates internally)
        │                                          ▲
        │ ssap://audio/getVolume (WebSocket push)  │ DE-15 TTL serial, absolute level 0–43
        ▼                                          │
   Home Assistant ──(volume 0–100 + mute)──▶ ESP32 (zarpli lib) ─┘
```

- **How volume maps:** TV volume scalar 0–100 → linear map to Z906 0–43 absolute. **Absolute-in, absolute-out = no drift, ever.** Mute and (read-back) power state sync because serial gives you the Z906's actual state.
- **Pros:** Most reliable signal (confirmed push), cleanest mapping, full two-way Z906 feedback, no IR line-of-sight, no CEC fragility, leverages your existing HA. Optical 5.1 untouched.
- **Cons:** Depends on the §1b Caveat-2 bench test passing. Two moving parts (HA + ESP32). Needs reconnect watchdog (#107808). No packaged ESPHome Z906 component exists — you wire it via MQTT (Jupsi) or port zarpli into a custom component.
- **Reliability:** High *if* the bench test passes. **Latency:** ~200–600 ms typical (WebSocket push + HA automation + MQTT/serial) — see §5.

### Option B — ESP32 all-in-one: CEC capture → Z906 serial (no Home Assistant in the loop)

```
LG G5 (Sound Out = Optical, "HDMI ARC device" handshake on a 2nd HDMI) ──optical 5.1──▶ Z906
        │ HDMI pin 13 (CEC, 3.3V)                                            ▲
        ▼                                                                    │ DE-15 serial
   ESP32 (esphome-native-hdmi-cec @ addr 0x5) ──vol up/down/mute──▶ same ESP32 (zarpli) ┘
```

- One ESP32 does CEC capture *and* Z906 serial. No HA dependency, self-contained, lowest latency.
- **Pros:** Single device, ~50–150 ms latency, no network. **Cons:** Sits squarely on the §1a contraindication — **may never produce volume events on G5 firmware.** Input is *relative* (up/down), so you maintain a counter, but serial feedback lets you correct drift. Also requires the TV to be in a mode that emits CEC volume *while* optical carries 5.1 — itself an unverified config (see §6, eARC-vs-optical tension).
- **Reliability:** Speculative on LG 2024–25. Best treated as a fallback/experiment.

### Option C — ESP32 CEC capture → IR blaster (the *original* plan)

- The plan as proposed: CEC → IR Z906 codes. **Pros:** no soldering into the Z906; reversible. **Cons:** stacks *two* fragile/indeterminate links — (1) the unconfirmed LG-CEC-volume premise, and (2) IR's relative-only, no-feedback, line-of-sight, toggle-only (power/mute) drift problem. This is the **worst** combination on reliability and the one to avoid unless you specifically don't want to touch the Z906 internals.

| | A: WebSocket → serial | B: CEC → serial | C: CEC → IR (original) |
|---|---|---|---|
| Volume signal confirmed on LG 2024–25 | ⚠️ needs bench test | ❌ contraindicated | ❌ contraindicated |
| Z906 actuation | ✅ absolute + feedback | ✅ absolute + feedback | ❌ relative, no feedback |
| Drift / desync | none | self-correcting | chronic |
| Mute/power sync | ✅ (serial readback) | ✅ (serial readback) | ❌ toggle-only, blind |
| Line-of-sight | n/a | n/a | required |
| Latency | ~200–600 ms | ~50–150 ms | ~80–250 ms + repeat lag |
| Complexity | medium (HA + ESP) | medium (1 ESP, risky) | medium (1 ESP, risky) |
| Touches Z906 internals | yes (DE-15) | yes (DE-15) | no |
| **Overall** | **Recommended** | Fallback / self-contained | Avoid |

> **A pragmatic hybrid:** build the **Z906 serial actuator first** (it's the high-value, low-risk part and works regardless of signal source). Then bench-test the WebSocket signal (Option A). Keep Option B/IR as the contingency if the WebSocket scalar turns out static in optical mode.

---

## 3. Bill of materials (Hungary sourcing)

> Prices are **indicative**, dated 2026-05-31, and several HU shop pages blocked automated price reads — **verify on the linked page before buying.** Items marked **UNCONFIRMED** are confirmed *available* but the figure is a typical-range estimate, not a quote. Do not treat unconfirmed figures as quotes.

### Recommended build (Option A) — minimal, no customs pain

| # | Item | Specific model | HU source | Approx price | Notes |
|---|------|----------------|-----------|--------------|-------|
| 1 | Controller | NodeMCU **ESP-32S** (ESP-WROOM-32) | wireless-bolt.hu (also hestore.hu, rpibolt.hu, tavir.hu) | **~3 350 Ft** | Runs zalpli serial + WiFi/MQTT to HA. 3.3 V TTL — talks to Z906 directly, no level shifter. |
| 2 | Z906 serial tap | DE-15 (VGA) male breakout / solder-tail connector | hestore.hu / AliExpress | **UNCONFIRMED** (~few hundred Ft / ~€1–3) | To reach pins 11/12/13/15. A spare VGA cable cut in half also works. |
| 3 | Dupont wires / proto | jumper set | hestore.hu / tavir.hu | <1 000 Ft | — |
| 4 | (optional) IR fallback LED | Vishay **TSAL6200** 940 nm | hestore.hu (prod_10026487) | UNCONFIRMED (~50–100 Ft) | Only if you keep an IR contingency. |
| 5 | (optional) IR driver | **2N2222A** or **BC337-40** NPN + ~100–220 Ω | hestore.hu (prod_10039410) | UNCONFIRMED (~15–40 Ft) | — |

**Total core cost: ~4 000–5 000 Ft (~€10–12).** Everything sourceable inside Hungary → no customs.

### If you also want the CEC experiment (Option B/C) or off-the-shelf IR

| Item | Model | HU source | Approx price | Notes |
|------|-------|-----------|--------------|-------|
| HDMI CEC breakout | HDMI male → screw-terminal (silk-screened CEC pin 13) | AliExpress | UNCONFIRMED (~€2–5) | Taps pin 13 (CEC) + pin 17 (CEC GND). **No level shifter — CEC is 3.3 V.** The one item worth importing. |
| Turnkey CEC board | **SMLIGHT SLWF-08** (purpose-built ESPHome HDMI-CEC board, USB-C, CEC on GPIO14, ships blank) | smartlight.me / AliExpress | ~€15 / ~$13 | Avoids HDMI soldering; runs the Palakis component. ⚠️ resellers disagree ESP32 vs ESP8266 — verify SoC before buying. |
| IR blaster (no-build) | **BroadLink RM4 Pro** (IR+RF) | kontaktor.hu / iotcentrum.hu / eMAG.hu | **16 100–17 298 Ft** | In stock HU, domestic delivery, no customs. RM4 mini (IR-only) cheaper, UNCONFIRMED ~9–12 000 Ft. |
| IR learn receiver | Vishay **TSOP38238** (38 kHz) | hestore.hu (prod_10046170) | UNCONFIRMED (~250–400 Ft) | Only needed once, to learn/verify Z906 codes. |

### Controller alternatives (if you prefer Pi over ESP32)

| Item | Model | HU source | Approx price | Notes |
|------|-------|-----------|--------------|-------|
| Pi Zero 2 W | Raspberry Pi Zero 2 W | malnapc.hu / pi-shop.hu | **6 900 / 7 990 Ft** (WH header 8 990) | Cheapest Pi; fine for libCEC + serial. |
| Pi 5 4 GB | RPi 5 4 GB (SC1111) | arukereso.hu (multi-shop) | **~56 235 Ft** | Overkill; only if you want it as the HA host too. |
| Pi 5 8 GB | RPi 5 8 GB | malnapc.hu | **69 900 Ft** | Overkill. |
| USB-CEC dongle | **Pulse-Eight USB-CEC** | pulse-eight.com (UK) / Amazon EU | UNCONFIRMED (~£35–45 / €45–55) | UK = non-EU → **import VAT + handling into HU**. Inherits libCEC's audio-emulation weakness (§1a). No advantage over ESP32 for this job. |

**For the recommended build, an ESP32 beats a Pi**: 3.3 V TTL talks to the Z906 with no level shifter, it's €1 vs €15–70, and ESPHome/MQTT integration with HA is trivial.

### EU import / customs note (AliExpress → Hungary, 2025–2026)

- **VAT (27% HU) applies to all imports** regardless of value. For consignments ≤ €150, AliExpress collects it at checkout via **IOSS** (usually no surprise bill). IOSS remains in place. [EU Taxation & Customs — High]
- **The €150 customs-duty de-minimis is being abolished** (EU Council, 13 Nov 2025). Interim **~€3 flat per-parcel handling** from ~1 Jul 2026; a separate **~€2 e-commerce handling fee** from ~Nov 2026. [taxation-customs.ec.europa.eu, Avalara — High]
- **Net advice:** source the jelly-bean parts (ESP32, transistor, IR LED, DE-15) **inside Hungary (hestore.hu / tavir.hu)** to dodge per-parcel friction. Only import the HDMI CEC breakout (no good HU equivalent), and consolidate into one order. Avoid Mouser/Digikey EU for handfuls of parts (high shipping/min-order).

---

## 4. Z906 control codes & the drift problem

### Serial (recommended actuator) — solves drift entirely

- **Electrical:** 3.3 V **TTL** UART (not RS-232, not 5 V-tolerant). **57600 baud, 8 data bits, ODD parity, 1 stop bit (8O1)** — the odd parity is an easy thing to miss. [zarpli, nomis `protocol.rst` — High]
- **DE-15 console pinout:** pin 3,6 = GND · pin 11 = 3.3 V @ 250 mA (you can power the ESP32 from this) · pin 12 = TX (amp→console) · pin 13 = RX (console→amp) · **pin 15 = Console Enable, must be tied to GND** to bring the amp out of standby.
- **Frame:** `AA <type> <length> <data…> <checksum>`, checksum chosen so the U8 sum from `type` through `checksum` ≡ 0 (mod 256). Many actions are single-byte (`08` vol+, `09` vol−, `02–07` input select, `14/15/16` 3D/4.1/2.1, `35` effect off, `3F` headphones).
- **Absolute volume + feedback (the headline):** native hardware range is **0–43** for main/rear/centre/sub. zarpli exposes absolute `MAIN_LEVEL` / `CENTER_LEVEL` / `SUB_LEVEL` / rear (`READ_LEVEL`) setters (its API documents 0–255, but the unit only resolves ~44 steps — *treat effective resolution as 0–43 and test the scaling on your unit* — flag). A **status frame (type 0x0A, ~20 bytes)** reports current main/rear/centre/sub levels, input (0–5), mute (0/1), standby state, auto-standby flag, temperature. **This two-way state is what IR fundamentally cannot give you and is why drift disappears.**
- **Functions over serial:** absolute volume + per-channel levels, 6 inputs (`SELECT_INPUT_1…5` + AUX), mute on/off, power on (`11 11`)/off (`30 37 36`) + standby readback, effects (3D/4.1/2.1/off/headphones), `EEPROM_SAVE`, firmware version, temperature. Test-tone over serial: **not documented as a first-class command** (flag — Low confidence).
- **Gotchas (the catch):** ① **Auto-standby** — the amp powers itself off after an idle timeout; mitigate with periodic keep-alive/reset-idle commands, by disabling auto-standby, or with nomis's firmware mod. ② **Pin-15 enable / console-coexistence** is the documented sore spot if you want the physical knob to keep working (Jupsi: "all problems are due to pin 8 and pin 15"); **replacing the console outright sidesteps it.** ③ Console restarts emit serial garbage — make the parser tolerate framing errors. ④ Firmware/model variance — one project couldn't read version/status on his unit; **test readback on your specific Z906.** ⑤ A documented power-register bug: 1→0 powers on, 0→1 does *not* go to standby.

**Key serial projects:**
- **zarpli/Logitech-Z906** (~118★) — canonical Arduino/C++ library, command constants, pinout, CAD. [High] https://github.com/zarpli/Logitech-Z906
- **nomis/logitech-z906** (~75★) — definitive `protocol.rst` + firmware-mod notes. [High] https://github.com/nomis/logitech-z906
- **Jupsi/logi_z906_wifi** (~63★) — ESP32 + MQTT + HA auto-discovery, *keeps the physical console working in parallel*. Caveat: mute tracking incomplete. [Medium-High] https://github.com/Jupsi/logi_z906_wifi
- **LewisSmallwood/IoT-Logitech-Z906** (~32★) — ESP8266, *replaces* the console, HTTP REST API. [Medium] https://github.com/LewisSmallwood/IoT-Logitech-Z906
- **BreadJS/Z906-Home-Assistant** (~10★) — ESP32 + MQTT, early/rough. [Low-Medium]
- **No packaged ESPHome external component exists** (flag) — you wire MQTT (Jupsi/Lewis) or port zarpli into a custom ESPHome component yourself. The biggest "homelab tax" of this route.

### IR (fallback only) — codes are documented but drift is inherent

- **Protocol: NEC** (NEC1 / Flipper "NECext"), **address `0xA002`** (device 2 / subdevice 160), carrier **~38 kHz** (JP1 says 38.65 kHz; 38 works). Discrete per-button codes are **well documented and mutually corroborated** across JP1, Flipper's official `audio.ir`, and multiple gists:

| Function | Cmd | 32-bit NEC frame |
|---|---|---|
| Power (**toggle**) | 0x80 | 0x400501FE |
| Mute (**toggle**) | 0xEA | 0x400557A8 |
| Volume + | 0xAA | 0x400555AA |
| Volume − | 0x6A | 0x400556A9 |
| Input (cycle) | 0x08 | 0x400510EF |
| Level (cycle target) | 0x0A | 0x400550AF |
| Effect | 0x0E | 0x4005708F |
| Input 1–5 / AUX / Test | DF/BE/CF/CE/BF / BD / 7F | `0x4005<cmd><~cmd>` |

- **The drift problem is real and confirmed:** volume is **relative-only** (no "set to N" IR code), **no IR feedback**, and **power & mute are single TOGGLE codes** (no discrete on/off). An IR-only controller *cannot guarantee* volume, mute, or power state.
- **The canonical IR workaround** (it's literally what Logitech's own console does internally for mute): **ramp to a known floor, then count up** — send ~44 `Vol−` to guarantee level 0, then N `Vol+` to reach a known absolute level. Crude, slow, audible.
- **IR code dumps:** pladaria gist (NEC + lircd.conf + ESPHome `transmit_lg`), Gh61 / mattwebboz gists (Broadlink base64 for HA), sakai-labo Arduino (IRremoteESP8266), Flipper-IRDB + official `audio.ir`, SmartIR #301. Implementation note: some users found the standard NEC encoder unreliable and had to **send raw captures** — keep that in your back pocket if you DIY the encoder. ESPHome quirk: pladaria uses `remote_transmitter.transmit_lg` with the 32-bit values rather than `transmit_nec` (LG/NEC timings are close enough).

**Bottom line:** IR is fine for *commands* but structurally bad for *stateful* volume/mute/power. Serial removes the problem. Only choose IR if you refuse to open the Z906 / tap the console cable.

---

## 5. Latency budget (remote press → Z906 reacts)

| Stage | Option A (WebSocket→serial) | Option B (CEC→serial) | Option C (CEC→IR) |
|---|---|---|---|
| Magic Remote → TV registers keypress | ~20–50 ms | ~20–50 ms | ~20–50 ms |
| TV emits signal | WebSocket push ~50–200 ms* | CEC frame ~20–60 ms | CEC frame ~20–60 ms |
| Transport to controller | HA automation + MQTT ~50–200 ms | on-chip ~0 ms | on-chip ~0 ms |
| Controller → Z906 | serial frame ~5–20 ms | serial ~5–20 ms | IR burst ~25–70 ms/code |
| **End-to-end (typical)** | **~200–600 ms** | **~50–150 ms** | **~80–250 ms + repeat lag** |

\* WebSocket latency is *push/real-time by design* but **not independently benchmarked** in sources (flag).

- **What feels laggy:** anything past ~300–400 ms feels sluggish on a volume tap. Option A is acceptable but not snappy; Option B is the snappiest. If Option A's HA round-trip feels slow, move the logic onto the ESP32 (subscribe to the WebSocket directly, or have HA fire MQTT with minimal automation overhead).
- **Held-button / repeat:** on a *hold*, the TV repeats volume events. With **serial absolute** you can debounce/coalesce: read the latest TV volume and set the Z906 once per ~50–100 ms instead of stepping — smooth, no overshoot. With **IR**, you must fire one `Vol±` per repeat and watch for **stuck-repeat** (a dropped key-release = runaway volume); add a max-repeat guard and a release timeout.

---

## 6. Edge cases / failure-mode checklist

- [ ] **The eARC-vs-optical tension (most important).** Your plan wants the TV to *believe a CEC/eARC audio system is the output* (to emit volume) **while** optical simultaneously carries 5.1 to the Z906. Many LG sets output to **one** Sound-Out device at a time, and an ESP32/libCEC CEC device is **not** a real eARC audio *sink* — eARC audio negotiation will fail, so the TV has nowhere to actually send audio. **This is why Option A (read the TV's volume scalar while Sound Out = Optical) is cleaner than Option B/C.** Bench-test whether optical stays live and whether the volume scalar still moves.
- [ ] **Volume desync (no absolute state).** Eliminated by serial absolute + readback. With IR, only the ramp-to-floor trick mitigates it.
- [ ] **Mute sync.** Serial reports mute state; map TV mute → Z906 mute directly. IR mute is a blind toggle — can invert. (Note: Jupsi reports mute tracking over serial is partial — verify on your unit.)
- [ ] **Power on/off sync.** Serial has power/standby readback (mind the 1→0 / 0→1 bug). IR power is a blind toggle. Decide whether TV-off should standby the Z906 (HA automation on TV power state).
- [ ] **The TV's volume OSD is meaningless.** In optical/external mode the on-screen bar doesn't reflect the Z906. In Option A you *exploit* this (the scalar is your control signal); just expect the OSD number to not match the Z906's actual level. Consider hiding it or accepting it.
- [ ] **eARC handshake dropping audio.** A webOS update (~Apr 2024) broke ARC audio for many (dropouts/freezes). If you go anywhere near eARC, you risk this. Optical-only audio path sidesteps it entirely — another point for Option A.
- [ ] **PCM-vs-passthrough killing 5.1.** To keep true Dolby Digital 5.1, set the TV's digital audio out to **Pass-Through / Bitstream / Auto**, *not* PCM (PCM downmixes to 2.0 over optical — optical TOSLINK can't carry multichannel LPCM). Verify the Z906 lights its Dolby Digital indicator. ⚠️ Note the tension: bitstream over optical means the *TV* cannot attenuate the audio (volume is fixed) — which is exactly why the Z906 must do the attenuation (via your serial control) and why you only need the TV's volume *number*, not its audio-domain volume.
- [ ] **CEC bus conflicts** with Apple TV / consoles on other HDMI ports. SimpLink can route source-switching and power oddly; a rogue CEC audio device can cause unexpected `Set System Audio Mode` fights. If you run Option B/C, isolate the CEC device and watch for input auto-switching.
- [ ] **Unexpected device wake.** CEC `One Touch Play` / `Active Source` can wake the TV or switch inputs. Test that your CEC device doesn't wake the TV at night.
- [ ] **IR line-of-sight / placement** (IR fallback only). The Z906 IR receiver is on the control console; the blaster needs LOS or a taped-on emitter over the sensor.
- [ ] **Held-button stuck-repeat** (see §5). Add a release timeout / max-repeat guard, especially for IR.
- [ ] **WebSocket subscription silently dies** (HA #107808). Add a reconnect watchdog; consider an availability sensor + auto-reload.
- [ ] **Z906 auto-standby** pulling the amp off mid-session. Keep-alive, disable auto-standby, or nomis firmware mod.
- [ ] **Pin-15 / console coexistence** instability if you keep the physical knob. Replacing the console is more stable.

---

## 7. Annotated resources

**CEC feasibility (read these before betting on CEC):**
- libCEC #594 — audio-system emulation, the 0x7F "volume unknown" stop-sending gotcha. **High (mechanism).** https://github.com/Pulse-Eight/libcec/issues/594
- AVForums 2518267 / AVS 3317352 — **2024 C4 + real LG soundbar: power-only, no CEC volume.** The decisive negative evidence. **High relevance / Medium credibility.** https://www.avforums.com/threads/anyone-got-lg-magic-remote-to-control-soundbar-volume.2518267/ · https://www.avsforum.com/threads/soundbar-issue-with-my-new-lg-c4-oled.3317352/
- libCEC #567 — libCEC ~30 s disconnects from LG OLED. **High repo / Medium age.** https://github.com/Pulse-Eight/libcec/issues/567
- Android CEC docs / CEC-O-MATIC — opcode/System-Audio-Mode reference. **High.** https://source.android.com/docs/devices/tv/hdmi-cec · https://www.cec-o-matic.com/

**HA webOS volume detection (the recommended signal source):**
- aiowebostv `webos_client.py` — `subscribe_volume`, push. **High (primary).** https://github.com/home-assistant-libs/aiowebostv
- HA core `webostv/media_player.py` — push, not polling. **High (primary).**
- HA core #107808 — volume subscription silently drops. **High.** https://github.com/home-assistant/core/issues/107808
- HA core #106981 / #102956 — sound_output/volume oddities on external out. **Medium-High.**

**CEC controller software:**
- **Palakis/esphome-native-hdmi-cec** (~315★) — the maintained ESPHome CEC component; can act as audio system 0x5 and you control the audio-status reply. **High.** https://github.com/Palakis/esphome-native-hdmi-cec
- johnboiles/esphome-hdmi-cec — **deprecated**, points to Palakis. **High.**
- HA HDMI-CEC integration still exists but Pi onboard-CEC support was **removed in 2021.7.0** (#52802); now needs pyCEC-over-TCP or Pulse-Eight. Treat as "supported but stagnant." **High.**
- SMLIGHT SLWF-08 — turnkey ESPHome CEC board. **Medium (vendor).** https://github.com/DocBig/HDMI-CEC-Gateway-with-SMLIGHT-SLWF-08-for-HomeAssistant
- HA community "CEC volume control for IR devices by pretending to be an HDMI ARC device" — the canonical writeup of this exact technique (body 403'd; corroborated via excerpts). **Medium.** https://community.home-assistant.io/t/.../323047

**Z906 control:**
- zarpli/Logitech-Z906 (lib), nomis/logitech-z906 (`protocol.rst`), Jupsi/logi_z906_wifi (HA+console-coexist), LewisSmallwood/IoT-Logitech-Z906 (console-replace HTTP). **High / High / Medium-High / Medium.** (URLs in §4.)
- IR dumps: pladaria gist (NEC/lircd/ESPHome), Gh61 & mattwebboz gists (Broadlink), sakai-labo (Arduino), Flipper official `audio.ir` + Flipper-IRDB. **High.**
- Hackaday Z906 hack overview (2022). **Medium** (summary; fetch 403'd, widely cited).

**Customs:** EU Taxation & Customs de-minimis removal + Avalara summary. **High.**

---

## 8. Recommended path & build order

1. **Build the Z906 serial actuator first** (highest value, lowest risk, works regardless of signal source). ESP32 + zarpli, **replace the console** (ground pin 15, power the ESP32 from pin 11), **disable auto-standby**, expose to HA via MQTT. Budget an afternoon for the standby/enable quirks. Verify absolute volume, mute, input, and status readback on *your* unit.
2. **Bench-test the §9 decisive unknown** before relying on Option A.
3. **Wire the signal source:**
   - If the bench test passes → **Option A**: HA automation maps `media_player` `volume_level`/`is_volume_muted` → MQTT → ESP32 → Z906 absolute level. Add a reconnect watchdog for #107808.
   - If the scalar is static in optical mode → try **LG Sound Sync (Optical)** mode, or fall back to **Option B** (esphome-native-hdmi-cec capture, you control the 0x7A reply), accepting it may not work on G5 firmware.
4. **Keep IR as a last-ditch contingency** (BroadLink RM4 or ESP32+TSAL6200) only if you won't touch the Z906 internals.

**Audio path stays trivial and untouched throughout:** TV optical out (Bitstream/Pass-Through) → Z906 optical in → true Dolby Digital 5.1. You never insert anything into the audio chain; you only attenuate inside the Z906 via serial.

---

## 9. The one thing to test before buying anything

> **Put the G5 in Sound Out = Optical, feed the Z906 5.1, and watch whether `ssap://audio/getVolume` still pushes a changing value on every Magic Remote volume keypress** (e.g. via HA Developer Tools → States on the `media_player`, or a quick `pywebostv`/`aiowebostv` `subscribe_volume` script).
>
> - **If yes** → Option A is green. Proceed; the whole project is reliable.
> - **If the value is static / no event** → the WebSocket signal won't work in optical mode. Fall back to LG Sound Sync (Optical) mode, or to CEC capture (Option B), or to capturing the remote some other way — and re-scope expectations.
>
> This single test de-risks the entire build. Everything else (the Z906 serial actuator) is already well-proven.

---

### Confidence & uncertainty summary

- **High confidence:** Z906 serial protocol (absolute volume + feedback) is real and turnkey; IR codes are documented; HA `webostv` pushes volume in *internal-speaker* mode; CEC volume on 2024 LG is contraindicated by real-soundbar failure reports; EU customs changes.
- **Must bench-test (unconfirmed):** whether `getVolume` fires on keypress in **optical-out** mode on G5 (§9); whether optical stays live while any CEC audio device is "present"; exact zarpli 0–255↔0–43 volume scaling; WebSocket latency numbers; SLWF-08 SoC (ESP32 vs ESP8266); several HU prices (marked UNCONFIRMED in §3).
- **Avoid building on:** generic-CEC-volume on 2024–2025 LG as a *primary* mechanism; IR-only stateful volume/mute/power.
