# Eveus / United Chargers — Local HTTP API

A community-maintained reference for the **local HTTP API** of Eveus / United
Chargers EV charging stations, so third-party tools (Home Assistant, dashboards,
scripts, energy managers) can read status and control the charger over the LAN.

> ⚠️ **Unofficial.** Not affiliated with, authorised by, or endorsed by the
> manufacturer. Provided as-is, no warranty. An EV charger switches mains voltage —
> **use at your own risk** and read the [disclaimer](API.md#11-disclaimer--use-at-your-own-risk)
> before sending any command.

## 📖 The reference

➡️ **[API.md](API.md)** — full documentation:

- All 12 endpoints (`/main`, `/pageEvent`, `/init`, `/config`, `/scan`, OCPP, …)
- Complete `/main` field reference with units and example values
- Full `/pageEvent` command list (charging, limits, schedules, tariffs)
- `state` / `subState` / `aiStatus` enumerations
- Single-phase vs three-phase differences
- Legacy vs current API generations, and changes across firmware versions

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
| older units | legacy generation — see [API.md §7](API.md#7-legacy-generation-differences) |

## Contributing

Corrections, additional fields, and captures from other firmware versions are
welcome — open an issue or PR. Please **redact serial numbers, IPs, MACs,
passwords, and OCPP keys** from any captured examples.

## License

- Documentation: **[CC BY 4.0](LICENSE)**
- Code samples: **MIT**

Trademarks and product names belong to their respective owners and are used only
to identify the hardware described.
