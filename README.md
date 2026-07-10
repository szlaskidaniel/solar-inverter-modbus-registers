# Solar Inverter Modbus Registers

Community-maintained **Modbus TCP register maps for solar inverters**, as structured JSON.
Read live PV power, battery SOC, grid import/export, load power and daily energy counters **directly from your inverter over the local network** — no cloud account, no vendor API keys.

Currently covers **10 profiles across 5 brands**: Solis, Sofar Solar, SolaX Power, Growatt and GoodWe — including full **alarm / error-code bit maps** for Solis hybrid and string inverters.

## Why

Every inverter vendor ships a cloud app, and every cloud app polls slowly, goes down, or requires an account in someone else's datacenter — while the inverter itself sits on your LAN happily speaking Modbus. The register documentation, however, is scattered across PDF datasheets, forum posts and reverse-engineering threads, and differs per model, per generation and sometimes per firmware.

This repo consolidates that knowledge into one machine-readable file you can drop into any Modbus client — Home Assistant, Node-RED, ESPHome, a Python script, or a native app.

## Supported inverters

| Brand | Model family | Battery data | Alarms | Notes |
|---|---|---|---|---|
| Solis | RHI / S6 Hybrid | ✅ | ✅ (80+ codes) | FC 04, unit ID 1 |
| Solis | S5 / S6 String | — | ✅ | `addressOffset: -1` (see below) |
| Sofar Solar | HYD ES (legacy 1-phase) | ✅ | — | FC 03 |
| Sofar Solar | HYD KTL-3PH (3-phase) | ✅ | — | FC 03 |
| SolaX Power | X1 / X3 Hybrid Gen3/Gen4 | ✅ | — | FC 04 |
| SolaX Power | X1 Air/Boost/Mini, X3 MIC (string, Gen1) | — | — | FC 04 |
| Growatt | SPH TL-BH (hybrid, Gen3) | ✅ | — | FC 04, power scale `0.0001` |
| Growatt | MIN / MIC TL-X (string, Gen1) | — | — | FC 04 |
| GoodWe | ET / EH / BT / BH (hybrid) | ✅ | — | FC 03, **unit ID 247** |
| GoodWe | DT / NS / MS / XS (string) | — | — | FC 03, unit ID 247 |

> Your model missing? [Open an issue](../../issues/new?template=new-inverter.md) or send a PR — see [CONTRIBUTING.md](CONTRIBUTING.md).

## File format

Everything lives in [`profiles.json`](profiles.json). One *profile* = one inverter family with a consistent register layout:

```jsonc
{
  "brandId": "solis",
  "brandName": "Solis",
  "modelId": "rhi-s6-hybrid",
  "modelName": "Solis RHI / S6 Hybrid (battery)",
  "unitId": 1,                       // Modbus unit / slave ID to address
  "polling": {
    "maxBlockSize": 70,              // max registers per read request
    "gapTolerance": 35               // merge reads if gap between addrs ≤ this
  },
  "fields": {
    "powerKW": {
      "fc": 4,                       // Modbus function code (3 = holding, 4 = input)
      "addr": 33057,                 // register address
      "count": 2,                    // 1 = 16-bit, 2 = 32-bit (two registers, big-endian)
      "signed": true,                // two's complement?
      "scale": 0.001,                // multiply raw value by this
      "min": -200, "max": 200        // sanity range — discard readings outside it
    },
    "statusWord": { "presence": "unsupported" }   // field not available on this model
  },
  "alarms": [ /* bit-mapped fault registers, see below */ ]
}
```

### Field semantics

| Key | Meaning |
|---|---|
| `fc` | Modbus function code: `3` = Read Holding Registers, `4` = Read Input Registers |
| `addr` | Register address as sent on the wire (see `addressOffset` caveat) |
| `count` | Number of consecutive 16-bit registers. `2` means a 32-bit value, high word first (big-endian) |
| `signed` | Interpret as two's complement. Critical for power fields where negative = export / discharge |
| `scale` | Multiplier applied to the raw integer (e.g. raw `4213` × `0.001` = `4.213 kW`) |
| `min` / `max` | Plausibility bounds. Dataloggers occasionally return garbage (`0xFFFF`, mid-write reads) — reject values outside this range instead of showing a 6 553.5 kW spike |
| `presence` | `"unsupported"` = model doesn't expose it; `"computed"` = derive it client-side (e.g. `plantState` from power flows) |

### Common field names

`powerKW` (PV / active power), `batteryPercent` (SOC), `batteryPowerKW` (signed: + charge / − discharge, vendor-dependent), `familyLoadPowerKW` (house load), `dayEnergyKWh` (today's generation), `gridImportTodayKWh` / `gridExportTodayKWh`, `homeLoadTodayKWh`, `batteryDischargeTodayKWh`, `gridConnectedFlag`.

The same logical field name maps to a different register on every profile — that's the entire point of this repo.

### Alarms

Solis profiles include bit-mapped fault registers. Read `count` consecutive registers starting at `addr`, concatenate them into one bit field (register order = bit order, 16 bits per register), and look up set bits:

```jsonc
"alarms": [{
  "fc": 4, "addr": 33116, "count": 5,
  "bits": {
    "0":  { "code": "1015", "level": "3", "message": "NO-Grid" },
    "33": { "code": "1053", "level": "3", "message": "OV-Vbatt" }
    // bit index → official Solis error code + severity (3 = fault, 2 = warning)
  }
}]
```

This doubles as a plain **Solis error code reference**: `1010 OV-G-V` (grid overvoltage), `1015 NO-Grid`, `1031 INI-FAULT`, `1041 ARC-FAULT`, and so on — see [`profiles.json`](profiles.json) for the full list.

## Gotchas we learned the hard way

- **Off-by-one addressing.** Some vendor docs list *register numbers* (1-based), some list *protocol addresses* (0-based). Solis string docs are 1-based, so that profile carries `addressOffset: -1` — subtract 1 from every `addr` before putting it on the wire. Profiles without the key need no adjustment.
- **GoodWe wants unit ID 247**, not 1. Sending unit ID 1 to a GoodWe datalogger gets you silence.
- **Block reads, politely.** Dataloggers are slow, single-client and easily offended. Batch registers into as few read requests as possible (respect `maxBlockSize` / `gapTolerance`), keep one TCP connection, and don't poll faster than every few seconds.
- **Scales differ wildly** — Growatt reports power at `0.0001` kW resolution while Solis uses `0.001`. Never assume; always apply the profile's `scale`.
- **Validate everything.** Mid-write reads and comms glitches produce plausible-looking garbage. That's what `min`/`max` are for.

## Quick start (Python)

```python
# pip install pymodbus
import json
from pymodbus.client import ModbusTcpClient

profile = next(p for p in json.load(open("profiles.json"))["profiles"]
               if p["modelId"] == "rhi-s6-hybrid")

client = ModbusTcpClient("192.168.1.50", port=502)
f = profile["fields"]["batteryPercent"]
rr = client.read_input_registers(f["addr"], count=f["count"], slave=profile["unitId"])
soc = rr.registers[0] * f["scale"]
print(f"Battery: {soc:.0f}%")
```

## Sources & verification

Register maps come from vendor Modbus documentation where available, cross-checked against community reverse-engineering work and **verified against real hardware in production** — these exact profiles power [Glance for PV](https://danielszlaski.com/glance-for-solis.html), an iOS/watchOS widget app reading inverters over local Modbus TCP.

## Contributing

Adding a model is one JSON object — see [CONTRIBUTING.md](CONTRIBUTING.md). Most-wanted: Huawei SUN2000, Deye/Sunsynk, Fronius (sunspec), SMA, Fox ESS.

## Disclaimer

All registers here are **read-only** (FC 03/04). This repo intentionally does not document write registers — misconfigured writes can damage equipment or violate grid codes. Use at your own risk; inverter firmware updates can change register layouts.

## License

[MIT](LICENSE)

---

*Maintained by [Daniel Szlaski](https://danielszlaski.com), author of Glance for PV.*
