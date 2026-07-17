# Eveus EV Charger — Local HTTP API (community reference)

Community-maintained reference for the local HTTP API exposed by Eveus /
United Chargers EV charging stations, as observed on devices on the local
network. **Unofficial — not affiliated with or endorsed by the manufacturer,
and not an official vendor document.** Field names and behaviour vary between
firmware releases. Verify against your own device before relying on anything here.

- Platform: **ESP32 (ESP-IDF)**, embedded web UI served from an on-flash filesystem.
- Transport: **HTTP** (no TLS) on port **80**, LAN only.
- Auth: **HTTP Basic** (`Authorization: Basic …`). Realm: `Please enter login and password`.
- **Every** request is `POST`, even pure reads. `GET` is not used by the web UI.
- Command bodies are `application/x-www-form-urlencoded`.
- Responses are `application/json` (data endpoints) or plain text (config-save endpoints, returning an error string or empty on success).

> Firmware covered: `1PGRW001A-R3.02.7` and `-R3.05.5` (single-phase),
> `3PGRW001A-R3.05.6` (three-phase); response examples captured on R3.02.9.
> Example values come from real captures with serial numbers, IPs and any secrets
> replaced by placeholders. §1.x lists what changed across these versions.

---

## Disclaimer — use at your own risk

⚠️ **Read this before sending any command to your charger.**

- **Unofficial & independent.** This document is a community reference. It is not
  affiliated with, authorised by, or endorsed by Eveus, United Chargers, or any
  manufacturer or distributor. All product names and trademarks belong to their
  respective owners and are used only to identify the hardware being described.
- **No warranty.** The information is provided "as is", without warranty of any
  kind, express or implied, including but not limited to fitness for a particular
  purpose. It may be incomplete, inaccurate, or outdated, and may not match your
  firmware.
- **You assume all risk.** You are solely responsible for anything you do with
  this information. To the maximum extent permitted by law, the authors and
  contributors accept **no liability** for any damage, loss, injury, fire,
  electric shock, voided warranty, data loss, or other harm arising from its use.
- **Danger — high-voltage equipment.** An EV charger switches mains voltage and
  high current. Sending commands you do not fully understand — especially
  calibration parameters (`kV*`, `kC*`, `voltKoef*`, `curKoef*`, `leakKoef`),
  `factoryReset`, or hardware-type/relay settings — **can permanently damage the
  device, corrupt its metering or safety calibration, or create a safety hazard.**
  Do not experiment on a charger connected to a vehicle or live load.
- **Local, authorised use only.** Use this only against hardware you own or are
  explicitly authorised to operate, on your own network. The API is unauthenticated-
  by-default and unencrypted (plain HTTP) — treat the device as untrusted on shared
  networks.
- **Not professional advice.** Nothing here is electrical, safety, or engineering
  advice. For anything involving mains wiring or hardware modification, consult a
  qualified electrician and your local regulations.

See also the [full license](#11-license). Corrections and additions from the
community are welcome.

---

## 1. Firmware naming & generations

A station runs **two** microcontrollers, each with its own firmware version, both
reported in `/main` and `/init`:

| Field | Example | Meaning |
|---|---|---|
| `verFWWifi` | `1PGRW001A-R3.05.5` | Wi-Fi/comms module (the one exposing this HTTP API) |
| `verFWMain` | `GRM070A-R3.05.4` | Power/status controller (metering, relay, safety) |
| `verFWStatus` | `1` | Status-controller protocol/state flag |

Wi-Fi firmware prefix encodes the phase count:

| Prefix | Hardware |
|---|---|
| `1PGRW…` | **single-phase** station |
| `3PGRW…` | **three-phase** station |

### Two API generations

Independently of phase count, there are two clearly different API *generations*.
This matters far more than 1P vs 3P:

| | **Legacy generation** | **Current generation** |
|---|---|---|
| `/main` size | ~37 fields | ~95+ fields |
| Numeric units | **scaled integers** (see §6) | **floats in real units** |
| `systemTime` | `"HH:MM:SS"` string | unix epoch (integer, UTC) |
| Charge state | `state` only (0–22) | `state` (0–7) **+** `subState` |
| Calibration | `voltKoef*/curKoef*/freqKoef/leakKoef` | `kV*/kC*/kCl` (in `/init`/`/debug`) |
| OCPP / tariffs / schedules | absent | present |
| Energy meters | `sessionEnergy/totalEnergy` | `+ IEM1/IEM2 (+_money)` |

> **Naming note.** "Legacy" / "current" are labels used *in this document* to
> group two observed API shapes. They are **not** vendor terms, and the device
> does not report them.

### There is no API-version field — key off the firmware version

The firmware exposes **no explicit API-version parameter**. `verFWStatus` is a
status-controller flag, not an API version. So identify the generation by the
**firmware version** and/or by feature-probing the `/main` response:

- **By firmware:** anchor to `verFWWifi` / `verFWMain` (e.g. `1PGRW001A-R3.05.5`,
  `GRM070A-R3.05.4`). The current generation corresponds to the `?PGRW001A-R3.0x`
  Wi-Fi firmware family this document covers.
- **By runtime probe (more robust across future releases):**
  - `subState` present in `/main` → **current** generation.
  - `systemTime` is a string (`"HH:MM:SS"`) → **legacy**; integer epoch → current.
  - `curMeas1` is a float → current; integer → legacy (scaled ×0.1).

Prefer the runtime probe: it survives firmware version-string changes and doesn't
require a lookup table of versions.

The rest of this document describes the **current generation** unless stated
otherwise; §7 covers the legacy differences.

### 1.x Observed changes across current-generation firmware

Differences observed between R3.02.x and R3.05.x releases (single-phase):

| Area | R3.02.x | R3.05.x |
|---|---|---|
| Endpoint set | 12 endpoints | **identical** — 12 endpoints, no additions/removals |
| `/main` schema | ~95 fields (matches this doc's example) | same fields + minor additions |
| Hardware-type field | **`typeEvse`** | renamed to **`evseType`** |
| `/config` body (Wi-Fi STA) | **single network** (`ssidName`, `ssidPassword`, `WifiMode`, `mac_bind`, `STA_MAC`) | **up to 3 networks** (`ssidName2/3`, static-IP & auto-connect per network — see §5.4) |
| `broadcastMode` command | absent | present (broadcast/master mode) |
| OCPP `connectorID` param | present | removed / superseded |

Practical takeaways for a client:
- Accept **both** `typeEvse` and `evseType` (read whichever is present).
- Don't assume `/config` echoes networks 2/3 on older firmware.
- The core `/main` telemetry and the charge-control commands (`currentSet`,
  `evseEnabled`, `aiMode`, limits, schedules, tariffs) are stable across R3.02–R3.05.

> The `/main` contract in this document was cross-checked across R3.02.x and
> R3.05.x releases; the documented fields are consistent between them.

---

## 2. Authentication

```
Authorization: Basic base64(username:password)
```

Default credentials are configured on the device. If no password is set the
device may accept unauthenticated requests. `/configHttp` changes the web
username/password (see below).

---

## 3. Endpoint overview

All endpoints are **identical between single-phase and three-phase firmware** —
same paths, same request bodies, same response schema.

| Endpoint | Purpose | Request body | Response |
|---|---|---|---|
| `POST /main` | Live status + full config snapshot | *(empty)* | JSON |
| `POST /pageEvent` | Set a single parameter / fire a command | `name=value` | text (empty ok) |
| `POST /init` | Full init dump (config incl. Wi-Fi/HTTP/OCPP) | *(empty)* | JSON |
| `POST /config` | Save Wi-Fi station config (up to **3 networks**) | form (see §5.4) | text |
| `POST /configAP` | Save access-point (hotspot) config | form | text |
| `POST /configHttp` | Save web login / password / auth key | form | text |
| `POST /scan` | Start a Wi-Fi scan | *(empty)* | text |
| `POST /scanResult` | Poll Wi-Fi scan results | *(empty)* | JSON array |
| `POST /debug` | Diagnostic snapshot | *(empty)* | JSON |
| `POST /get_logResult` | Retrieve device log | *(empty)* | JSON |
| `POST /ocppEvent` | Save OCPP settings / fire OCPP action | form | text |
| `POST /ocppData` | OCPP status/data | *(empty)* | JSON |

---

## 4. `POST /main` — live status

Returns a single JSON object combining live telemetry and the current
configuration. The station's own web UI polls this on an interval.

### 4.1 Example (current generation, single-phase; sanitised)

> Captured on R3.02.9 firmware — note it still uses `typeEvse` (renamed to
> `evseType` in R3.05.x, see §1.x).

```json
{
  "verFWMain": "GRM070A-R3.02.9",
  "verFWWifi": "1PGRW001A-R3.02.9",
  "verFWStatus": 1,
  "serialNum": "EVS-XXXXXXXXXXXX",
  "stationId": "EVS-XXXXXXXXXXXX",
  "fwCRC32": "8CF51E06",
  "pilot": 2,
  "state": 4,
  "subState": 0,
  "evseEnabled": 0,
  "currentSet": 30,
  "curDesign": 32,
  "minCurrent": 7,
  "minVoltage": 180,
  "gridRange": 0,
  "typeEvse": 1,
  "typeRelay": 0,
  "aiStatus": 3,
  "aiVoltage": 195,
  "ground": 1,
  "groundCtrl": 0,
  "tarif": 712,
  "activeTarif": 1,
  "tarifAEnable": 1, "tarifBEnable": 0,
  "tarifAValue": 513, "tarifBValue": 100,
  "tarifAStart": 0, "tarifAStop": 0, "tarifBStart": 0, "tarifBStop": 0,
  "timerType": 0,
  "timeZone": 2,
  "systemTime": 1734022761,
  "timeLimitS": 0, "energyLimitS": 0, "moneyLimitS": 0,
  "timeLimit": 0, "energyLimit": 0, "moneyLimit": 0,
  "delayedLimit": 0,
  "suspendErrors": 0, "suspendLimits": 0,
  "oneCharge": 0,
  "sh1Enabled": 0, "sh1Start": 0, "sh1Stop": 0,
  "sh1CurrentEnable": 0, "sh1CurrentValue": 12,
  "sh1EnergyEnable": 0, "sh1EnergyValue": 0,
  "sh2Enabled": 0, "sh2Start": 0, "sh2Stop": 0,
  "sh2CurrentEnable": 0, "sh2CurrentValue": 12,
  "sh2EnergyEnable": 0, "sh2EnergyValue": 0,
  "totalEnergy": 2123.99,
  "sessionTime": 11183,
  "sessionEnergy": 16.01,
  "sessionMoney": 82.12,
  "sessionStarted": 1,
  "IEM1": 103.96, "IEM2": 2123.99,
  "IEM1_money": 533.22, "IEM2_money": 11079.88,
  "curMeas1": 28.52, "curMeas2": 0, "curMeas3": 0,
  "voltMeas1": 212, "voltMeas2": 0, "voltMeas3": 0,
  "powerMeas": 6048,
  "temperature1": 18, "temperature2": 6,
  "leakValue": 0, "leakValueH": 44,
  "vBat": 2.82,
  "adapter": 255,
  "lang": 1,
  "ocppconnected": 0, "ocppEnabled": 0, "ocppOfflineAva": 1,
  "RSSI": -63,
  "logReady": 1
}
```

### 4.2 Field reference (current generation)

**Live telemetry**

| Field | Unit | Notes |
|---|---|---|
| `state` | enum | Charge state, see §6.1 |
| `subState` | enum | Meaning depends on `state`: error codes when `state==7`, otherwise limit codes. See §6.2 |
| `pilot` | enum | Control-Pilot level / connection state |
| `evseEnabled` | 0/1 | **0 = charging enabled, 1 = disabled** on the current generation (inverted vs intuition) |
| `curMeas1` / `curMeas2` / `curMeas3` | A | Per-phase current. **1-phase: `curMeas2/3` always 0** |
| `voltMeas1` / `voltMeas2` / `voltMeas3` | V | Per-phase voltage. **1-phase: `voltMeas2/3` always 0** |
| `powerMeas` | W | Active power |
| `temperature1` / `temperature2` | °C | Box / plug temperatures |
| `leakValue` | mA | Earth-leakage current |
| `leakValueH` | mA | Leakage high-water threshold |
| `ground` | 0/1 | Ground present |
| `vBat` | V | RTC backup battery |
| `RSSI` | dBm | Wi-Fi signal (present in `/main`; the current UI reads it here) |
| `sessionTime` | s | Current session duration |
| `sessionEnergy` | kWh | Current session energy |
| `sessionMoney` | currency | Current session cost |
| `sessionStarted` | 0/1 | Session active flag |
| `totalEnergy` | kWh | Lifetime energy |
| `IEM1` / `IEM2` | kWh | Two independent internal energy meters |
| `IEM1_money` / `IEM2_money` | currency | Cost accumulated per meter |
| `activeTarif` | int | Currently active tariff |

**Configuration echoed in `/main`** (writable via `/pageEvent`, see §5.2)

| Field | Unit | Notes |
|---|---|---|
| `currentSet` | A | Charge current setpoint |
| `curDesign` | A | Hardware max current |
| `minCurrent` | A | Minimum current |
| `minVoltage` | V | Under-voltage cutoff |
| `aiStatus` | enum | Adaptive/AI mode, see §6.3 |
| `aiVoltage` | V | AI reference voltage |
| `groundCtrl` | 0/1 | Ground control enable |
| `timeLimitS` / `energyLimitS` / `moneyLimitS` | s / kWh / currency | Session limit **setpoints** |
| `timeLimit` / `energyLimit` / `moneyLimit` | — | Current limit state |
| `delayedLimit` | — | Delayed-start limit |
| `oneCharge` | 0/1 | One-shot charge |
| `suspendLimits` / `suspendErrors` | 0/1 | Temporarily ignore limits / errors |
| `tarif`, `tarifAEnable/AValue/AStart/AStop`, `tarifB…` | — | Two-tariff pricing schedule |
| `sh1*`, `sh2*` | — | Two charging schedules (see §5.2) |
| `timerType`, `timeZone`, `lang` | — | Clock / locale |
| `evseType`, `typeRelay` | enum | Hardware type / relay type. ⚠️ `evseType` was named **`typeEvse`** through R3.02.x — see §1.x |
| `adapter` | int | Adapter code (`255` = none) |

**Identity / firmware**

| Field | Notes |
|---|---|
| `serialNum`, `stationId` | Device serial / station id |
| `fwCRC32` | Firmware CRC |
| `verFWMain`, `verFWWifi`, `verFWStatus` | See §1 |

**OCPP**

| Field | Notes |
|---|---|
| `ocppEnabled` | OCPP client enabled |
| `ocppconnected` | Connected to central system |
| `ocppOfflineAva` | Offline-availability flag |

---

## 5. `POST /pageEvent` — set parameter / fire command

Writes a single parameter or fires an action. The web UI sends both a form body
**and** a custom header naming the parameter:

```
POST /pageEvent HTTP/1.1
Authorization: Basic …
Content-Type: application/x-www-form-urlencoded
pageEvent: currentSet

currentSet=32
```

Minimal form (the header is set by the UI; the body `name=value` is the payload):

```bash
curl -s -u USER:PASS -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'currentSet=32' \
  http://CHARGER_IP/pageEvent
```

Several parameters can be sent at once as an `&`-joined body (the UI's
`postPageEventSeveral`), e.g. `sh1Start=1380&sh1Stop=360&sh1Enabled=1`.

### 5.1 Common controls

| Command | Values | Effect |
|---|---|---|
| `currentSet` | min…`curDesign` (A) | Charge current. UI sends 2-digit zero-padded (`08`) |
| `evseEnabled` | 0/1 | Enable/disable charging (**current generation: 0=on, 1=off**) |
| `oneCharge` | 0/1 | One-shot charge for current session |
| `chargeNow` | 0/1 | Force charge now (override limits) |
| `aiMode` | 0/1/2/3 | Adaptive mode, see §6.3 |
| `broadcastMode` | 0/1 | Broadcast/master mode |

### 5.2 Full command list (49 commands, identical on 1P & 3P)

Charging / current: `currentSet`, `minCurrent`, `curDesign`, `evseEnabled`,
`oneCharge`, `chargeNow`, `aiMode`, `broadcastMode`

Session limits: `energyLimitS`, `timeLimitS`(*), `moneyLimitS`(*), `delayedLimit`,
`suspendLimits`, `suspendErrors`

Tariffs: `tarif`, `tarifAValue`, `tarifAEnable`, `tarifAStart`,
`tarifBValue`, `tarifBEnable`, `tarifBStart`, `tarifBStop`

Schedule 1: `sh1Enabled`, `sh1Start`, `sh1Stop`, `sh1CurrentEnable`,
`sh1CurrentValue`, `sh1EnergyEnable`, `sh1EnergyValue`

Schedule 2: `sh2Enabled`, `sh2Start`, `sh2Stop`, `sh2CurrentEnable`,
`sh2CurrentValue`, `sh2EnergyEnable`, `sh2EnergyValue`

Meters / reset: `rstEM1`, `rstEM2` (reset internal meters), `factoryReset`

Clock / locale: `systemTime` (unix epoch on current generation), `timeZone`, `timerType`, `lang`

UI/nav: `pageOpen`, `pageevent`

⚠️ **Hardware / calibration — do not send unless you know exactly what you are
doing; wrong values can damage metering or hardware:**
`kC1`, `kC2`, `kC3`, `kCl`, `kV1`, `kV3` (current generation 3-phase also `kV2`),
`evseType`, `typeRelay`, `groundCtrl`, `minVoltage`, `curDesign`, `Cmax`,
`factoryReset`

(*) `timeLimitS`/`moneyLimitS` are present in `/main` and the UI; the exact
`pageEvent` key spelling should be confirmed against `/init` on your firmware.

### 5.3 Schedule fields

Each of the two schedules (`sh1*`, `sh2*`):

| Suffix | Meaning |
|---|---|
| `Enabled` | schedule on/off |
| `Start` / `Stop` | window start/stop (minutes since midnight) |
| `CurrentEnable` + `CurrentValue` | cap current during window (A) |
| `EnergyEnable` + `EnergyValue` | stop at energy limit (kWh) |

> `sh*EnergyValue`/`sh*EnergyEnable` exist since at least R3.02.9; R3.05.x fixed
> their behaviour (per vendor release notes). The fields themselves are not new.

### 5.4 Wi-Fi station config — `POST /config` (up to 3 networks)

The current firmware supports three saved Wi-Fi networks. Body (form-urlencoded):

```
ssidName, ssidPassword,        WifiMode, mac_bind,  STA_MAC,  auto_conn,  stat_ip_s, stat_ip,  stat_gw   # primary
ssidName2, ssidPassword2,      mac_bind2, STA_MAC2, auto_conn2, stat_ip_s2, stat_ip2, stat_gw2          # secondary
ssidName3, ssidPassword3,      mac_bind3, STA_MAC3, auto_conn3, stat_ip_s3, stat_ip3, stat_gw3          # third
```

- `auto_conn*` — auto-connect toggle
- `stat_ip_s*` — static-IP enable; `stat_ip*`/`stat_gw*` — address / gateway
- `mac_bind*` / `STA_MAC*` — bind connection to a specific router MAC

> **Firmware note.** The multi-network form (`*2`/`*3` fields, static-IP & auto-connect)
> is R3.05.x. R3.02.x accepted a single network only:
> `ssidName`, `ssidPassword`, `ssidPasswordConf`, `WifiMode`, `mac_bind`, `STA_MAC`.

### 5.5 Access point — `POST /configAP`

```
ssidNameAP, ssidPasswordAP, ssidPasswordAPConf
```

### 5.6 Web credentials — `POST /configHttp`

```
httpUsername, httpPassword, httpPasswordConf, authenticationKey
```

Returns a non-empty text body on error, empty on success.

---

## 6. Enumerations

> Enum meanings are community-contributed and best-effort. Confirm against your
> own device before depending on a specific code.

### 6.1 `state` (current generation)

| # | Meaning |
|---|---|
| 0 | startup |
| 1 | system_test |
| 2 | standby |
| 3 | connected |
| 4 | charging |
| 5 | charge_complete |
| 6 | paused |
| 7 | error (see `subState` §6.2) |

### 6.2 `subState` (current generation)

When `state == 7` (**error codes**):

| # | Meaning | | # | Meaning |
|---|---|---|---|---|
| 0 | no_error | | 8 | low_voltage |
| 1 | grounding_error | | 9 | diode_error |
| 2 | current_leak_high | | 10 | overcurrent |
| 3 | relay_error | | 11 | interface_timeout |
| 4 | current_leak_low | | 12 | software_failure |
| 5 | box_overheat | | 13 | gfci_test_failure |
| 6 | plug_overheat | | 14 | high_voltage |
| 7 | pilot_error | | | |

Otherwise (**limit / status codes**):

| # | Meaning | | # | Meaning |
|---|---|---|---|---|
| 0 | no_limits | | 6 | schedule1_energy_limit |
| 1 | limited_by_user | | 7 | schedule2_limit |
| 2 | energy_limit | | 8 | schedule2_energy_limit |
| 3 | time_limit | | 9 | waiting_for_activation |
| 4 | cost_limit | | 10 | paused_by_adaptive_mode |
| 5 | schedule1_limit | | | |

### 6.3 `aiStatus` / `aiMode`

| # | Meaning |
|---|---|
| 0 | off |
| 1 | voltage (adaptive by grid voltage) |
| 2 | tesla_auto |
| 3 | power (adaptive by power) |

---

## 7. Legacy generation differences

Older units expose a smaller, differently-scaled `/main`. Example (sanitised):

```json
{
  "evseEnabled": 1, "state": 6, "currentSet": 24, "curDesign": 32,
  "curMeas1": 238, "curMeas2": 0, "curMeas3": 0,
  "voltMeas1": 212, "voltMeas2": 0, "voltMeas3": 0,
  "aiModecurrent": 24, "aiStatus": 0, "aiVoltage": 200,
  "ground": 1, "groundCtrl": 0,
  "timeLimit": 500000, "energyLimit": 10000,
  "temperature1": 43, "temperature2": -60,
  "oneCharge": 0,
  "sessionEnergy": 63, "sessionTime": 4561, "totalEnergy": 11620,
  "timerType": 2, "systemTime": "01:25:50",
  "typeEvse": 1, "typeRelay": 0,
  "voltKoef1": 342, "voltKoef2": 342, "voltKoef3": 342,
  "curKoef1": 350, "curKoef2": 350, "curKoef3": 350,
  "freqKoef": 16, "leakKoef": 15, "leakValue": 0
}
```

Key differences vs the current generation:

- **`evseEnabled`: 1 = charging enabled, 0 = disabled** (opposite of current generation).
- **`state`** uses a wider 0–22 enum (states *and* error/limit reasons combined):
  0 no_data · 1 ready · 2 waiting · 3–6 charging · 7 current_leak · 8 cpu_error ·
  9 no_ground · 10 overheat_plug · 11 overheat_relay · 12 overcurrent ·
  13 overvoltage · 14 undervoltage · 15 limited_by_time · 16 limited_by_energy ·
  17 limited_by_money · 18 limited_by_schedule1 · 19 limited_by_schedule2 ·
  20 disabled_by_user · 21 relay_stuck · 22 limited_by_ai_mode.
  There is **no** `subState`.
- **Scaled integers** (see §8): current in 0.1 A, energy in 0.1 kWh.
- `systemTime` is a `"HH:MM:SS"` string, not a unix epoch.
- `temperature2 == -60` is a common "sensor absent / not connected" sentinel.
- No OCPP, tariffs, schedules, or `IEM*` meters.
- Calibration exposed as `voltKoef*/curKoef*/freqKoef/leakKoef`.

`/pageEvent` on legacy generation accepts at least `currentSet` (zero-padded 2-digit) and
`evseEnabled`; the richer command set is current-generation only.

---

## 8. Units & scaling cheatsheet

| Quantity | Legacy generation | Current generation |
|---|---|---|
| Phase current `curMeasN` | integer ×0.1 → A (`238` = 23.8 A) | float A (`28.52`) |
| Phase voltage `voltMeasN` | integer V | integer V |
| Power `powerMeas` | (compute V×I) | integer W |
| Session/total energy | integer ×0.1 → kWh (`63` = 6.3 kWh) | float kWh (`16.01`) |
| `systemTime` | `"HH:MM:SS"` string | unix epoch (UTC, seconds) |
| Temperatures | integer °C (`-60` = absent) | integer °C |
| `sessionTime` | seconds | seconds |

---

## 9. Single-phase vs three-phase — summary

**The HTTP API is identical.** Same endpoints, same `/pageEvent` command set, same
`/main` JSON schema. Firmware `1PGRW*` and `3PGRW*` differ only in:

1. **Runtime values** — on 3-phase units `curMeas2/3` and `voltMeas2/3` carry real
   readings; on single-phase units they are always `0`.
2. **Calibration** — 3-phase firmware adds per-phase calibration coefficients
   (`kV2/kV3`, `kC2/kC3`) used by config/debug endpoints, not `/main`.

A consumer written against single-phase will work unchanged on three-phase; it
only needs to start reading `curMeas2/3` / `voltMeas2/3` to expose the extra phases.

---

## 10. Other endpoints (brief)

- **`POST /init`** — JSON superset of `/main` config, including `httpUsername`,
  Wi-Fi SSIDs, OCPP URL (`urlUrl`/`urlPort`/`urlPath`/`urlProtocol`),
  `authenticationKey`. Used by the UI to populate settings pages.
- **`POST /scan`** then **`POST /scanResult`** — start a Wi-Fi scan, then poll.
  `/scanResult` returns an array: `[{ "name": "<ssid>", "rssi": -62, "mac": "aa:bb:…" }, …]`.
- **`POST /debug`** — diagnostic JSON (`dataDebug`): calibration trims (`tr_*`),
  action flags (`a_*`), CP diagnostics (`cp_*`), etc.
- **`POST /get_logResult`** — device log as JSON.
- **`POST /ocppEvent`** — save OCPP config (`ocppEnabled`, `urlUrl`, `urlPort`,
  `urlPath`, `authenticationKey`). **`POST /ocppData`** — OCPP status/data.
  Default central-system host observed in UI: `ocpp.unitedchargers.com`.

---

## 11. License

This documentation (text, tables, and examples) is released under
**[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)**.
You may share and adapt it, including commercially, as long as you give
appropriate credit and indicate if changes were made.

Any code samples in this document are additionally available under the
**MIT License**.

Trademarks and product names are the property of their respective owners; the
CC BY / MIT licenses cover this documentation only, not any manufacturer's
firmware, software, or trademarks.
