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

> Firmware covered: `1PGRW001A-R3.02.7`, `-R3.02.9` and `-R3.05.5` (single-phase),
> `3PGRW001A-R3.05.6` (three-phase). The `/main` example below was captured on R3.02.9;
> the R3.02.9 → R3.05.4/R3.05.5 differences in §1.x were measured on the **same physical
> unit** before and after that update.
> Example values come from real captures with serial numbers, IPs and any secrets
> replaced by placeholders.

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
| `verFWStatus` | `1` | **Firmware-integrity flag**, not an API version: `0` makes the station's own web UI render both version strings in red. Any other value = OK |

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
| `systemTime` | `"HH:MM:SS"` string | unix epoch (integer) — **local time, not UTC**, see §4.3 |
| Charge state | `state` only (0–22) | `state` (0–7) **+** `subState` |
| Calibration | `voltKoef*/curKoef*/freqKoef/leakKoef` | `kV*/kC*/kCl` (in `/init`/`/debug`) |
| OCPP / tariffs / schedules | absent | present |
| Energy meters | `sessionEnergy/totalEnergy` | `+ IEM1/IEM2 (+_money)` |

> **Naming note.** "Legacy" / "current" are labels used *in this document* to
> group two observed API shapes. They are **not** vendor terms, and the device
> does not report them.

### There is no API-version field — key off the firmware version

The firmware exposes **no explicit API-version parameter**. (`verFWStatus` looks like
one but is a firmware-integrity flag — see §1.) So identify the generation by the
**firmware version** and/or by feature-probing the `/main` response:

- **By firmware:** anchor to `verFWWifi` / `verFWMain` (e.g. `1PGRW001A-R3.05.5`,
  `GRM070A-R3.05.4`). The current generation corresponds to the `?PGRW001A-R3.0x`
  Wi-Fi firmware family this document covers.
- **By runtime probe (more robust across future releases):**
  - `verFWWifi` present in `/main` → **current** generation; absent → legacy. (Equivalent
    probes: `subState` or `minCurrent` present → current.)
  - `systemTime` is a string (`"HH:MM:SS"`) → **legacy**; integer epoch → current.
  - `curMeas1` is a float → current; integer → legacy (scaled ×0.1).
  - Calibration coefficients appear directly in `/main` (`voltKoef*`, `curKoef*`) → legacy;
    on the current generation they live in `/init` / `/debug` instead.

Prefer the runtime probe: it survives firmware version-string changes and doesn't
require a lookup table of versions.

The rest of this document describes the **current generation** unless stated
otherwise; §7 covers the legacy differences.

### 1.x Observed changes across current-generation firmware

Measured on **one physical single-phase unit captured before and after an
R3.02.9 → R3.05.4/R3.05.5 update**, so the differences below are firmware changes, not
model differences.

| Area | R3.02.9 | R3.05.4 / R3.05.5 |
|---|---|---|
| `/main` field count | 95 | **101–102** (see §3.1 — it also varies with device state) |
| `/main` additions | — | `model`, `manufacturer`, `evseType`, `switchState`, `fixedMode`, `ocppVendor`, `aiAutoPercent`, `broadcastMode`, `displayOrientation` |
| `/main` removals | `typeEvse`, `adapter` | — |
| Hardware-type field | **`typeEvse`** | renamed to **`evseType`** |
| `/init` field count | 13 | **38** (the multi-network block) |
| `/debug` field count | 42 | 42 — unchanged |
| `/ocppData` | 20 fields, incl. `publicMode` | 19 — **`publicMode` removed** |
| `/config` body (Wi-Fi STA) | **single network** (`ssidName`, `ssidPassword`, `WifiMode`, `mac_bind`, `STA_MAC`) | **up to 3 networks** (`ssidName2/3`, static-IP & auto-connect per network — see §5.5) |
| `broadcastMode` | absent | present — soft-AP broadcast policy, see §5.1 |
| OCPP `connectorID` param | present | removed / superseded |

Practical takeaways for a client:
- Accept **both** `typeEvse` and `evseType` (read whichever is present).
- Don't assume `/config` echoes networks 2/3 on older firmware.
- Don't add `publicMode` — it exists only on the older generation.
- The core `/main` telemetry and the charge-control commands (`currentSet`,
  `evseEnabled`, `aiMode`, limits, schedules, tariffs) are stable across R3.02–R3.05.

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
| `POST /config` | Save Wi-Fi station config (up to **3 networks**) | form (see §5.5) | text |
| `POST /configAP` | Save access-point (hotspot) config | form | text |
| `POST /configHttp` | Save web login / password / auth key | form | text |
| `POST /scan` | Start a Wi-Fi scan | *(empty)* | text |
| `POST /scanResult` | Poll Wi-Fi scan results | *(empty)* | JSON array |
| `POST /debug` | Diagnostic snapshot | *(empty)* | JSON |
| `POST /get_logResult` | Completed-session log (see §10.1) | *(empty)* | JSON array |
| `POST /getLogData` | Event log (see §10.2) | *(empty)* | **CSV text** |
| `POST /cleanLogData` | Erase the event log | *(empty)* | text |
| `POST /ocppEvent` | Save OCPP settings / fire OCPP action | form | text |
| `POST /ocppData` | OCPP status/data | *(empty)* | JSON |

The firmware also registers OTA-update endpoints (`/update/*`, `/oldupdate/*`,
`/updateEvent`) and serves the UI pages themselves (`/`, `/service`, `/ocppconfig`,
`/log.html`). Those are outside the scope of this reference.

### 3.1 The `/main` key set is not a stable contract

Two things vary, so a client must tolerate missing keys rather than assume a fixed schema:

- **By firmware version** — see §1.x.
- **By device state** — `logReady` is present only while it is set: it appears while a
  session is running and is **absent entirely** (not `0`) when idle. That alone accounts for
  the 101-vs-102 field count on the same firmware.

The station's own web UI guards every single field with a presence check and treats a missing
optional flag as `0`. Do the same.

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
| `leakValue` | see note | Earth-leakage reading, **low** channel |
| `leakValueH` | see note | Earth-leakage reading, **high** channel — a second measurement channel, *not* a threshold. On observed devices it is the one that goes non-zero |
| `ground` | 0/1 | `1` = protective earth present (`0` is the fault condition) |
| `vBat` | V | RTC backup battery |
| `RSSI` | dBm | Wi-Fi signal (present in `/main`; the current UI reads it here) |
| `sessionTime` | s | Current session duration |
| `sessionEnergy` | kWh | Current session energy |
| `sessionMoney` | currency | Current session cost |
| `sessionStarted` | 0/1 | Session active flag |
| `totalEnergy` | kWh | Lifetime energy. On observed devices **numerically identical to `IEM2`** — the same counter, except `IEM2` can be reset |
| `IEM1` / `IEM2` | kWh | Two **user-resettable trip meters** over the same physical meter (`rstEM1` / `rstEM2`). They are *not* a split by energy source |
| `IEM1_money` / `IEM2_money` | currency | Cost accumulated per meter |
| `activeTarif` | enum | Which rate is in force **right now**: `0` = primary (`tarif`), `1` = rate A (`tarifAValue`), `2` = rate B (`tarifBValue`). Read-only — the device chooses; there is no command to set it |

> **Do not derive cost from energy or vice versa.** `sessionMoney` and `sessionEnergy` are
> accumulated independently and are not guaranteed mutually consistent within a single
> response; after a mid-session tariff switch their ratio matches neither rate.

**Configuration echoed in `/main`** (writable via `/pageEvent`, see §5.2)

| Field | Unit | Notes |
|---|---|---|
| `currentSet` | A | Charge current setpoint. **Absolute value, applied immediately — there is no ramp/slew primitive in the API** |
| `curDesign` | A | Hardware max current — the upper bound for `currentSet` |
| `minCurrent` | A | Station minimum — the lower bound for `currentSet`. Read it rather than assuming a constant: it differed (7 → 6) between firmware releases on the same unit |
| `gridRange` | enum | Grid voltage domain: `0` = 230 V family, `1` = 110 V family. ⚠️ On `1` the station **clamps all currents to 12 A** and the voltage ranges shift |
| `minVoltage` | V | Under-voltage cutoff. Valid values are a closed set that depends on `gridRange` |
| `aiStatus` | enum | Adaptive mode currently active, see §6.3. ⚠️ Written under a **different name**, `aiMode` |
| `aiVoltage` | V | Adaptive mode 1 under-voltage **threshold** — stored configuration chosen by the user, not a measurement. Range: min = `minVoltage + 10`, max = 220 (or 110 when `gridRange == 1`) |
| `aiModecurrent` | A | The current the adaptive algorithm is **actually allowing** right now; the UI shows it as "allowed / requested" next to `currentSet`. Only meaningful while `aiStatus != 0` |
| `aiVoltageStart` | V | Voltage captured at charge start, used as the adaptive-Auto reference (`0` when idle) |
| `aiVoltageDrop` | % | Adaptive-Auto drop from `aiVoltageStart` (`0` when idle) |
| `aiPowerDrop` | W | Adaptive-Power measured power drop |
| `aiAutoPercent` | % | Stored adaptive-Auto set-point. Present from R3.05.x; no command writes it |
| `groundCtrl` | 0/1 | Protective-earth **monitoring** enable (`1` = PE control active). ⚠️ A service-level setting — see the warning in §5.2 |
| `switchState` | int | Read-back of a physical current-limit selector on the housing. Present from R3.05.x; the mapping from position to amperes is not established |
| `timeMsg` | 0/1 | `1` = **the device clock is not usable** (typically a flat RTC backup battery, cf. `vBat`). When set, ignore `systemTime` |
| `scanComplete` | 0/1 | `1` = a Wi-Fi scan has finished; fetch `/scanResult` once |
| `logReady` | 0/1 | `1` = session-log data is available (`/get_logResult`). **Key may be absent entirely** — see §3.1 |
| `displayOrientation` | 0/1 | Rotate the on-device display 180°. R3.05.x+ |
| `broadcastMode` | 0/1/2 | Soft-AP broadcast policy, see §5.1. R3.05.x+ |
| `model` / `manufacturer` | string | Device-reported names, e.g. `"EVEUS Pro 1P 2024"` / `"EVEUS"`. R3.05.x+ — absent on older firmware |
| `fixedMode` | int | ⚠️ **Not configuration.** Observed changing between polls minutes apart with no reboot, and mirrored in `/init`. Treat as an internal/rolling value and do not surface it. R3.05.x+ |
| `ocppVendor` | int | CSMS vendor quirk profile (`0` = default). R3.05.x+ |
| `sessionStarted` | 0/1 | A session is open. This is what distinguishes live session counters from leftover ones — they keep their values after unplug until the next session begins |
| `STA_IP_Addres` | string | Station IP on the LAN. **Spelling is the firmware's** (one `s`) |
| `serialNumCPU` | string | Legacy controller serial; empty when not provisioned |
| `SNflag` | enum | Status of the last serial-number registration attempt against the vendor cloud (`0` = never attempted) |
| `rfData` | int | Revision counter for the identity block — when it changes, re-read identity fields. Not RFID |
| `timeLimitS` / `energyLimitS` / `moneyLimitS` | 0/1 | Limit **enable flags** — the `S` suffix is a *state/switch*, not a setpoint |
| `timeLimit` / `energyLimit` / `moneyLimit` | s / kWh / currency | Limit **values**. ⚠️ Units and sentinels are not uniform — see §5.4 before writing these |
| `delayedLimit` | 0/1 | "Special limit signal" — changes how a limit/schedule stop is signalled; not a limit itself |
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

⚠️ **While OCPP is connected and driving a charging profile, a local `currentSet` write is
ignored.** Since the station answers with HTTP 200 either way (§5), these three fields are the
only way a client can tell that its setpoint is being overridden. The matching `subState` is
`9` (§6.2).

### 4.3 `systemTime` is a **local**-time epoch, not UTC

This is the single easiest thing to get wrong, and the direction differs per operation:

- **Reading** `/main.systemTime` gives `unix_epoch + timeZone*3600` — i.e. an epoch whose UTC
  rendering equals the station's **local wall clock**. Decoding it as UTC and comparing it
  against real UTC yields a constant error equal to the configured offset.
- **Writing** `systemTime` expects a **true UTC epoch**; the station applies `timeZone`
  itself. (The station's own web UI writes `Date.now()/1000` and does so on *every* page load,
  so opening the web UI re-syncs the clock from the browser.)
- `timeZone` is a separate signed integer in **whole hours**, accepted in the range −12…+12.
- If `timeMsg == 1`, the clock is not usable at all and `systemTime` should be ignored.

On the legacy generation `systemTime` is a `"HH:MM:SS"` wall-clock string with no date and no
timezone.

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

**The `pageEvent` header is optional.** The parameter name is taken from the **body**; the
header merely repeats it. Requests without it are accepted, and the station's own bulk writes
omit it. Several parameters can be sent at once as an `&`-joined body, e.g.
`sh1Start=1380&sh1Stop=360&sh1Enabled=1`.

### 5.0 Response handling — HTTP 200 does not mean success

There is **no JSON error envelope and no error-code field**. Application-level rejections come
back as **HTTP 200 with a plain-text body**, so a rejected write is indistinguishable from an
accepted one unless you read the body:

| Body | Meaning |
|---|---|
| `OK` | accepted — confirmed on a live device (`/pageEvent`, R3.05.4/5); treat this as the success body, not the string below |
| `ILLEGAL_CMD` | unknown parameter name, or the controller rejected the command |
| `Failed to post control value` | the POST body itself could not be read — e.g. the connection was cut mid-request (see the one-connection-at-a-time note below). This is a transport-level failure, not a rejection of the value by the charge controller; retrying is the appropriate response |
| `content too long` | request body too large (see limits below) |
| `Error: already started` | start command while a session is already running |

`mainPost successfully` also exists in the firmware binary and was previously documented here as
the success body, based on static string analysis. A live test showed the actual response is the
bare string `OK` instead — the binary contains both strings, but only tracing execution (not
string extraction) can tell which endpoint emits which. Treat any success-body claim not backed by
a live capture as a hypothesis.

The only status codes the vendor code sets itself are `200`, `302` (UI route redirects) and
`401`. Everything else (`400`, `404`, `405`, `500`, …) comes from the underlying HTTP server.

**Practical limits and behaviour, worth designing around:**

- **Request body limit is 511 bytes** on `/pageEvent` and the other form endpoints; exceeding
  it returns `content too long`.
- **The station serves one HTTP connection at a time**, and a new connection causes the
  existing session to be closed rather than queued. Concurrent requests therefore abort each
  other — serialise all requests to the device, and do not rely on keep-alive sockets staying
  usable while anything else (for example an open web UI, which polls once per second) is
  talking to the charger.
- A write reaches the charge controller asynchronously. Allow roughly **two poll cycles**
  before reading a value back — the station's own UI discards the next 2 `/main` responses
  after every write. Do not treat an immediate read-back as authoritative.
- Reads of `/main` are served from cached values and do not themselves query the charge
  controller, so a successful response proves the comms module is alive, not that the data is
  fresh. `systemTime` failing to advance between polls is a cheap staleness check.

### 5.1 Common controls

| Command | Values | Effect |
|---|---|---|
| `currentSet` | `minCurrent`…`curDesign` (A) | Charge current. UI sends 2-digit zero-padded (`08`) |
| `evseEnabled` | 0/1 | **`1` = stop charging, `0` = charging permitted** on the current generation. The station's own UI labels this switch "Stop charging". Legacy generation is inverted |
| `oneCharge` | 0/1 | Bypass the session limits for one charge. Write the camelCase name |
| `chargeNow` | `0` | **Clears every limit and schedule enable flag** — a reset command rather than a "start" command. The station's own UI always sends `0` |
| `aiMode` | 0/1/2/3 | Adaptive mode, see §6.3. Read back as `aiStatus` |
| `broadcastMode` | 0/1/2 | Soft-AP broadcast policy: `0` = always on, `1` = off once connected, `2` = always off. Other values are rejected with `Wrong Mode. Use 0, 1 or 2.` |
| `rstEM1` / `rstEM2` | *(no value)* | Reset trip meter A / B. The station's own UI literally posts `rstEM1=undefined` / `rstEM2=undefined` (a vendor client bug — the value argument is omitted, not empty), and the firmware acts on the parameter name alone. Sending `rstEM1=1` is expected to work too but has not been confirmed against a device. ⚠️ `rstEM2` also zeroes `totalEnergy` on observed devices |

### 5.2 Full command list (49 commands, identical on 1P & 3P)

Charging / current: `currentSet`, `minCurrent`, `curDesign`, `evseEnabled`,
`oneCharge`, `chargeNow`, `aiMode`, `broadcastMode`

Session limits — **enable flags**: `timeLimitS`, `energyLimitS`, `moneyLimitS`;
**values**: `timeLimit`, `energyLimit`, `moneyLimit` (units differ — see §5.4);
bypasses: `suspendLimits`, `oneCharge`; other: `delayedLimit`, `suspendErrors`

Tariffs: `tarif`, `tarifAEnable`, `tarifAValue`, `tarifAStart`, `tarifAStop`,
`tarifBEnable`, `tarifBValue`, `tarifBStart`, `tarifBStop`

Adaptive mode: `aiMode`, `aiVoltage`

Display: `displayOrientation`

Schedule 1: `sh1Enabled`, `sh1Start`, `sh1Stop`, `sh1CurrentEnable`,
`sh1CurrentValue`, `sh1EnergyEnable`, `sh1EnergyValue`

Schedule 2: `sh2Enabled`, `sh2Start`, `sh2Stop`, `sh2CurrentEnable`,
`sh2CurrentValue`, `sh2EnergyEnable`, `sh2EnergyValue`

Meters / reset: `rstEM1`, `rstEM2` (reset internal meters), `factoryReset`

Clock / locale: `systemTime` (unix epoch on current generation), `timeZone`, `timerType`, `lang`

UI/nav: `pageOpen` (sent once as `pageOpen=1` after `/init`; optional telemetry, not a
handshake — no endpoint requires it first)

⚠️ **Hardware / calibration / service — do not send unless you know exactly what you are
doing; wrong values can damage metering, defeat protections, or create a hazard:**
`kC1`, `kC2`, `kC3`, `kCl`, `kV1`, `kV2`, `kV3`, `evseType`, `typeRelay`, `minVoltage`,
`curDesign`, `minCurrent`, `Cmax`, `suspendErrors`, `factoryReset`.

⚠️ `groundCtrl` belongs in that list too: it enables/disables **protective-earth
monitoring**, and on the station it is reachable only from the service page. It is safe to
*read*; do not offer it as a normal user control.

### 5.3 Schedule fields

Each of the two schedules (`sh1*`, `sh2*`):

| Suffix | Meaning |
|---|---|
| `Enabled` | schedule on/off |
| `Start` / `Stop` | window start/stop (**minutes since midnight**, 0…1439) |
| `CurrentEnable` + `CurrentValue` | cap current during window (A) — same bounds as `currentSet` |
| `EnergyEnable` + `EnergyValue` | stop at energy limit (**kWh**, both directions) |

- Windows **wrap around midnight** (`Stop < Start` is the shipped default), and
  `Start == Stop` behaves as "always active".
- Resolution is one minute; there is no seconds component.
- An active schedule window can block charging **even when `suspendLimits` is set** — that
  bypass applies only to the time/energy/cost limits.

> `sh*EnergyValue`/`sh*EnergyEnable` exist since at least R3.02.9; R3.05.x fixed
> their behaviour (per vendor release notes). The fields themselves are not new.

### 5.4 Limit units and sentinel values ⚠️

The limits do **not** share a unit convention, and two of them use magic read-back values that
mean "not set". Getting this wrong silently produces the wrong limit.

| Parameter | Unit on **write** | Unit on **read** | "Not set" sentinel |
|---|---|---|---|
| `timeLimit` | seconds (0…86399) | seconds | read-back **≥ 500000** → treat as unset/0 |
| `energyLimit` | **watt-hours** — multiply kWh by 1000 | **kilowatt-hours**, clamped to 200 | none (clamped, not sentinelled) |
| `moneyLimit` | whole currency units (integer; fractional limits are not possible) | currency | read-back **> 20000** → treat as unset |
| `sh1EnergyValue` / `sh2EnergyValue` | **kilowatt-hours** | kilowatt-hours | none |

So `energyLimit` and `sh*EnergyValue` express the same physical quantity in **different units**
within the same API: `energyLimit=5000` sets 5 kWh and reads back as `5`, while
`sh1EnergyValue=5` sets 5 kWh directly.

Also note:

- The **enable flag and the value are two separate writes** — there is no atomic "set and
  enable".
- `suspendLimits=1` bypasses the three limits above; `oneCharge=1` does so for a single
  session. Neither bypasses a schedule.
- Tariff values (`tarif`, `tarifAValue`, `tarifBValue`) are **hundredths of a currency unit per
  kWh** — `513` means 5.13/kWh. Tariff window times use the same minutes-since-midnight
  encoding as schedules.

### 5.5 Wi-Fi station config — `POST /config` (up to 3 networks)

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

### 5.6 Access point — `POST /configAP`

```
ssidNameAP, ssidPasswordAP, ssidPasswordAPConf
```

### 5.7 Web credentials — `POST /configHttp`

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
| 6 | disabled — entered when `evseEnabled` is set to stop (the firmware's own label is "Disabled") |
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
| 3 | time_limit | | 9 | **external_limit** |
| 4 | cost_limit | | 10 | paused_by_adaptive_mode |
| 5 | schedule1_limit | | | |

> **Code 9 (`external_limit`)** means current is being commanded from outside — OCPP or a
> vendor app. The station's own display renders it as *"App control is active"*. Note that some
> firmware translations render this string as "waiting for activation", which reads as the
> opposite of what it means. Code 10 means the adaptive algorithm is throttling — adaptive mode
> reduces current but never stops charging.

### 6.3 `aiStatus` / `aiMode`

Read the active mode from `aiStatus`; **write it as `aiMode`** — the read and write names differ.

| # | Firmware label | Meaning |
|---|---|---|
| 0 | — | off |
| 1 | Voltage | reduce current when grid voltage drops below the `aiVoltage` threshold |
| 2 | **Auto** | reduce current by percentage as voltage sags relative to `aiVoltageStart`. The station documents the ladder as: −6 % → current −20 %, −8 % → −30 %, −10 % → current = `minCurrent`. (Sometimes labelled "Tesla auto" by third-party clients; the firmware calls it simply "Auto" and nothing about it is vehicle-specific.) |
| 3 | Power | reduce current on measured power drop (`aiPowerDrop`, compared against a fixed 200 W) |

Adaptive mode only ever **lowers** the current; it never stops charging. The effective value it
allows is reported in `aiModecurrent`.

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
| `systemTime` | `"HH:MM:SS"` string | unix epoch, **local time** — read `+ timeZone*3600`, write true UTC (§4.3) |
| Temperatures | integer °C | integer °C. **Any reading below −50 means "sensor absent"** (`-60` is the common value) — applies to both generations |
| `sessionTime` | seconds | seconds |
| `timeLimit` | seconds | seconds (sentinel ≥ 500000 = unset) |
| `energyLimit` | — | **Wh on write, kWh on read** (§5.4) |
| `sh*EnergyValue` | — | kWh both directions (§5.4) |
| Schedule / tariff times | — | minutes since midnight, 0…1439 |
| `tarif`, `tarifAValue`, `tarifBValue` | — | hundredths of a currency unit per kWh (`513` = 5.13) |

---

## 9. Single-phase vs three-phase — summary

**The HTTP API is identical.** Same endpoints, same `/pageEvent` command set, same
`/main` JSON schema. Firmware `1PGRW*` and `3PGRW*` differ only in:

1. **Runtime values** — on 3-phase units `curMeas2/3` and `voltMeas2/3` carry real
   readings; on single-phase units they are always `0`. The same applies to the per-phase
   entries in `/debug`.
2. **Calibration** — the per-phase coefficients (`kV1..kV3`, `kC1..kC3`, `kCl`) are exposed via
   `/debug` on **both** variants, including single-phase units, where the phase-2/3 entries are
   simply unused. They never appear in `/main` on the current generation.

A consumer written against single-phase will work unchanged on three-phase; it
only needs to start reading `curMeas2/3` / `voltMeas2/3` to expose the extra phases.

---

## 10. Other endpoints (brief)

- **`POST /init`** — JSON superset of `/main` config, including `httpUsername`,
  Wi-Fi SSIDs, OCPP URL (`urlUrl`/`urlPort`/`urlPath`/`urlProtocol`),
  `authenticationKey`. Used by the UI to populate settings pages.
- **`POST /scan`** then **`POST /scanResult`** — start a Wi-Fi scan, then poll.
  `/scanResult` returns an array: `[{ "name": "<ssid>", "rssi": -62, "mac": "aa:bb:…" }, …]`.
- **`POST /debug`** — diagnostic JSON: raw ADC readings, calibration coefficients (`k*`),
  protection trip thresholds (`tr_*`), and interlock flags (`a_*`). The `tr_*` values are the
  actual cut-out points behind the error codes in §6.2 — for example under/over-voltage,
  over-current, over-temperature and the two leakage channels. ⚠️ The `a_*` flags **disable**
  individual protections (GFCI self-test, relay test, diode control, the housing current-limit
  selector); read them if useful, but do not write them.

### 10.1 `POST /get_logResult` — completed-session log

Returns a **JSON array, newest first**, of per-session summary records:

| Field | Meaning |
|---|---|
| `log_mnth`, `log_dd`, `log_hh`, `log_mm`, `log_sec` | session **start** timestamp, by the device clock |
| `s_hh`, `s_mm`, `s_sec` | session duration |
| `s_enrg` | energy, kWh |
| `s_cost` | cost, currency |

Four properties matter to a consumer, all observed on real devices:

1. **The ring holds 4 records** — a fifth session evicts the oldest.
2. **The newest record is the session in progress and updates live** — its duration, energy and
   cost track the corresponding `/main` values. It only becomes final once the next session
   pushes it down the list. Do not read record 0 as "the last completed session".
3. **The timestamp is the session start**, frozen at whatever the device clock read at that
   moment and **never corrected afterwards**. If the clock is subsequently adjusted (via SNTP,
   or by anyone opening the web UI), records stop being monotonic — a record can even carry a
   timestamp ahead of the current clock.
4. **There is no year field.** The consumer must infer it, which is ambiguous across a New Year
   boundary.

Records persist across reboots and firmware updates. There is no parameter to page, filter or
delete them.

### 10.2 `POST /getLogData` — event log (CSV)

Returns **CSV text**, not JSON, with this header:

```
TS,Src,Evt,cId,Data0,Data1,Data2,v1,v2,v3,c1,c2,c3,power,energy
```

- `TS` is `%Y-%m-%dT%H:%M:%S` — **this log does carry the year**, unlike `/get_logResult`.
  `0000-00-00T00:00:00` appears when the clock is invalid.
- `Src` is a subsystem tag (`EVSE`, `OCPPC`, `SERVE`, `DISPL`, `STM`, `NVS`, `OTA`, …).
- `Evt` is an event name (`Power on`, `Restart`, `Connected`, `DisConnected`, `StateUpdate`,
  `StartTransaction`, `StopTransaction`, `MeterValues`, `SetChargingProfile`, `OfflineMode`, …).

This is an **event** log with instantaneous power/energy columns — it has no per-session
summary, so reconstructing a session means pairing start/stop events yourself. It is also what
OCPP `GetDiagnostics` uploads.

**`POST /cleanLogData`** erases this event log (replies `clan logs successfully` / `clan logs
error` — the typo is the firmware's). It takes no parameters and no confirmation token.

There is **no pagination** on any of the log endpoints; all three are parameterless POSTs.
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
