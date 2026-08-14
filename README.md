# Eveus / United Chargers — Local HTTP API

A community-maintained reference for the **local HTTP API** of Eveus / United
Chargers EV charging stations, so third-party tools (Home Assistant, dashboards,
scripts, energy managers) can read status and control the charger over the LAN.

> ⚠️ **Unofficial.** Not affiliated with, authorised by, or endorsed by the
> manufacturer. Provided as-is, no warranty. An EV charger switches mains voltage —
> **use at your own risk** and read the [disclaimer](API.md#disclaimer--use-at-your-own-risk)
> before sending any command.

## 📖 The reference

➡️ **[API.md](API.md)** — full documentation:

- All 12 endpoints (`/main`, `/pageEvent`, `/init`, `/config`, `/scan`, OCPP, …)
- Complete `/main` field reference with units and example values
- Full `/pageEvent` command list (charging, limits, schedules, tariffs)
- `state` / `subState` / `aiStatus` enumerations
- Single-phase vs three-phase differences
- Legacy vs current API generations, and changes across firmware versions

➡️ **[openapi.yaml](openapi.yaml)** — machine-readable OpenAPI 3.1 spec for the
**current generation** (all 12 endpoints, `MainResponse` schema, examples). Import it
into Swagger UI, Postman, Insomnia, or generate clients with `openapi-generator`.

➡️ **[openapi-legacy.yaml](openapi-legacy.yaml)** — a **separate** OpenAPI 3.1 spec for
the **legacy** generation (the pre-rebrand "EnergyStar" firmware). Deliberately a second
file rather than branches inside the first: legacy is not an older revision of the same
API but a different codebase — different `state` enum, inverted `timerType`, different
limit units, different write-response contract, smaller endpoint set. Conflating the two
has already caused real bugs in client integrations.

> Note: the API predates OpenAPI conventions — every operation is `POST` (even
> reads), so tools will show reads as POST operations. That is expected.

### ⚠️ Errata — legacy `state` enum corrected 2026-08-14

Every version of this reference published before **2026-08-14** contained an **incorrect
`state` enum for the legacy generation**. It was inherited from a third-party
current-generation config and applied to the wrong firmware; it matches the real enum on
only two codes.

The practical effect is inverted states, not merely wrong labels — a car sitting plugged
in reports `9`, which the old table called `no_ground`, and an unplugged station reports
`12`, which it called `overcurrent`. **If you copied that table, please re-check against
API.md §7.1.**

## Quick start

The API is plain HTTP on port 80, LAN only, HTTP Basic auth. Every request is a
`POST` — even reads.

```bash
# Read live status
curl -s -u USER:PASS -X POST http://CHARGER_IP/main | jq .

# Set charging current to 16 A
curl -s -u USER:PASS -X POST \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data 'currentSet=16' \
  http://CHARGER_IP/pageEvent
```

## Compatibility

| Firmware family | Notes |
|---|---|
| `1PGRW001A-R3.0x` | single-phase, current generation |
| `3PGRW001A-R3.0x` | three-phase, current generation (identical API) |
| older units ("EnergyStar") | legacy generation — a different API: see [API.md §7](API.md#7-legacy-generation-differences) and [openapi-legacy.yaml](openapi-legacy.yaml) |

## Manufacturer & official resources

For the product itself — purchase, warranty, official firmware, and support —
refer to the manufacturer. This project is **not** a source of firmware or official
support.

- **Official site:** https://eveus.ua/

If you need a firmware update or have a hardware/warranty question, use the
manufacturer's official channels, not this repository.

## Contributing

Corrections, additional fields, and captures from other firmware versions are
welcome — open an issue or PR. Please **redact serial numbers, IPs, MACs,
passwords, and OCPP keys** from any captured examples.

## Disclaimer

⚠️ **Unofficial, community project — use at your own risk.**

- Not affiliated with, authorised by, or endorsed by Eveus, United Chargers, or any
  manufacturer or distributor. All product names and trademarks belong to their
  respective owners and are used only to identify the hardware described.
- Provided **"as is", without warranty of any kind**. The information may be
  incomplete, inaccurate, or outdated, and may not match your firmware.
- An EV charger switches **mains voltage and high current**. Sending commands you
  do not fully understand — especially calibration parameters, `factoryReset`, or
  hardware-type/relay settings — **can permanently damage the device, corrupt its
  metering or safety calibration, or create a safety hazard.**
- Use only against hardware you own or are explicitly authorised to operate, on your
  own network. Nothing here is electrical or safety advice.

You are solely responsible for anything you do with this information; to the maximum
extent permitted by law the authors and contributors accept **no liability**. Full
text: [API.md → Disclaimer](API.md#disclaimer--use-at-your-own-risk).

## License

- Documentation: **[CC BY 4.0](LICENSE)**
- Code samples: **MIT**

Trademarks and product names belong to their respective owners and are used only
to identify the hardware described.
