# Contributing

Adding an inverter = adding one object to the `profiles` array in [`profiles.json`](profiles.json). No code required.

## Adding a new inverter profile

1. **Copy the closest existing profile** (hybrid → copy a hybrid; string → copy a string profile).
2. Set `brandId` / `modelId` (lowercase, kebab-case) and human-readable `brandName` / `modelName`. Include the model _family_ and generation in the name — register maps are usually per-generation, not per-SKU.
3. Fill in the `fields` you have verified. For everything else, be explicit:
   - `{ "presence": "unsupported" }` - the model does not expose this value
   - `{ "presence": "computed" }` - derivable client-side from other fields
     Please don't just omit unknown fields; an explicit `unsupported` tells consumers "checked, not there" vs. "nobody looked yet".
4. Set `unitId` (test it — GoodWe uses 247, most others 1) and, if the vendor docs use 1-based register numbers, add `addressOffset: -1`.
5. Add `polling.maxBlockSize` / `polling.gapTolerance` if you know the datalogger's limits; omit if untested.

## Verification standard

State in your PR description **how the map was verified**, one of:

- ✅ **Hardware-verified** — you read these registers from a real inverter and the values matched the vendor app/display. Include model + firmware version if known.
- 📄 **Docs-derived** — taken from official vendor Modbus documentation (link or name the document + revision).
- 🔎 **Community-sourced** — from a reverse-engineering thread or another open project (link it).

Hardware-verified > docs-derived > community-sourced. Docs lie surprisingly often (wrong scales, off-by-one addresses), so if you can spot-check even two or three fields against real hardware, say so.

### Quick sanity checklist

- [ ] `powerKW` matches the inverter display within rounding
- [ ] `signed` is correct — force an export / battery discharge and confirm the sign
- [ ] 32-bit fields (`count: 2`) decode high-word-first; a value like `4 294 967.2 kWh` means you got the word order wrong
- [ ] `scale` verified against a known value, not assumed from a sibling model
- [ ] JSON is valid: `python3 -m json.tool profiles.json > /dev/null`

## Fixing an existing profile

PRs correcting registers are very welcome — please include which hardware/firmware you tested on, since vendors do change layouts between firmware revisions. If a layout genuinely differs by firmware, add a **new profile** with the generation in `modelId` rather than editing the old one.
I personally tested this against Solis Hybrid with Pylontech Battery setup.

## Scope

- **Read-only registers only** (FC 03 / 04). Write registers are out of scope for this repo — see the disclaimer in the README.
- Live/instantaneous values and daily counters first; exotic diagnostics second.
- Alarm/fault bit maps are highly welcome (see the Solis profiles for the format).

## Field naming

Reuse the existing canonical field names (`powerKW`, `batteryPercent`, `gridImportTodayKWh`, …) so consumers can switch profiles without remapping. If your inverter exposes something genuinely new, propose the name in the PR — convention: `camelCase`, unit suffix (`KW`, `KWh`, `Percent`), `Today`/`ThisMonth` for period counters.
