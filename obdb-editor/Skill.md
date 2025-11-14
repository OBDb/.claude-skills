---
name: obdb-editor
description: Add and manage OBD-II extended PIDs in OBDb vehicle signalset JSON files by converting CSV formulas to proper mul/div/add format
license: Complete terms in LICENSE.txt
---

# OBD Parameter Management

You are an expert in working with OBD-II (On-Board Diagnostics) extended PIDs (Parameter IDs) for vehicles. Use this skill when asked to add OBD parameters from CSV files, convert parameter formulas, or manage vehicle signalset configurations for the OBDb.

## When to Use This Skill

Invoke this skill when:
- Adding OBD parameters from CSV files to signalset JSON
- Converting vehicle parameter equations to JSON format
- Managing extended PIDs for vehicle diagnostics
- Working with CAN bus headers and OBD-II protocols

## Core Concept: The Value Calculation Formula

**CRITICAL**: All OBD parameter values are calculated using this exact formula:

```
var value: Double
if sign {
  value = SignedBitReader(response, bix, len)
} else {
  value = UnsignedBitReader(response, bix, len)
}
value = (value * mul / div + add).clamped(to: min...max)
return Measurement(value: value, unit: unit)
```

**Default values when not specified:**
- `mul = 1`
- `div = 1`
- `add = 0`
- `bix = 0` (bit index offset)

## Two-Phase Process

### Phase 1: Analysis
1. **Read** the CSV files containing OBD PID mappings
2. **Compare** against existing signalset to identify missing parameters
3. **Group** parameters logically by system (battery, motor, charging, etc.)
4. **Plan** implementation phases

### Phase 2: Implementation
1. **Convert** CSV formulas to JSON format using conversion rules
2. **Add** parameters with correct headers, units, and model year filters
3. **Validate** that max values are mathematically achievable
4. **Test** the configuration

## Formula Conversion: Critical Rules

### Basic Conversions

**Raw value (no conversion):**
- CSV: `A`
- ✅ JSON: `{}`

**Multiply by 0.5:**
- CSV: `A*0.5`
- ✅ JSON: `{"div": 2}` (preferred)
- ❌ JSON: `{"mul": 0.5}` (avoid - use div instead)

**Multiply by 2:**
- CSV: `A*2`
- ✅ JSON: `{"mul": 2}`

**Multiply by 0.01:**
- CSV: `A*0.01`
- ✅ JSON: `{"div": 100}` (preferred)
- ❌ JSON: `{"mul": 0.01}` (avoid - use div instead)

**Subtract 40:**
- CSV: `A-40`
- ✅ JSON: `{"add": -40}`

**Complex formula (multiply by 0.05 then add 6):**
- CSV: `A*0.05+6`
- ✅ JSON: `{"div": 20, "add": 6}` (0.05 = 1/20, then add 6)

See "Integer Division vs Floating-Point Multiplication" section below for detailed guidance.

### Data Type Conversions

| CSV Pattern | JSON Format | Notes |
|-------------|-------------|-------|
| `INT16(A:B)` | `{"len": 16}` | 16-bit unsigned |
| `INT16(A:B)*0.01` | `{"len": 16, "div": 100}` | 16-bit ÷ 100 |
| `signed(A)*256+B` | `{"len": 16, "sign": true}` | 16-bit signed |
| `(signed(A)*256+B)*0.1` | `{"len": 16, "sign": true, "div": 10}` | Signed ÷ 10 |

### ⚠️ Unit Simplification Rule

**CRITICAL**: Use the simplest unit representation for each parameter. Infer the vehicle's native unit from the formula.

**Temperature Example:**
- CSV: `(A-40)*1.8+32`
- Analysis: Vehicle stores in Celsius, formula converts to Fahrenheit
- ❌ **Wrong**: `{"add": -40, "mul": 9, "div": 5, "add": 32, "unit": "fahrenheit"}` (over-complicated)
- ✅ **Correct**: `{"add": -40, "unit": "celsius"}` (simple - let app handle display conversion)

**Inference rules:**
- `(A-40)*1.8+32` or `(A-50)*9/5+32` → Vehicle uses Celsius, convert to `{"add": -40, "unit": "celsius"}`
- `A*0.621371` (km to miles) → Vehicle uses kilometers, keep as `{"mul": 621371, "div": 1000000, "unit": "kilometers"}`
- `A/2.55` → Simplify to `{"mul": 100, "div": 255}` (avoid decimals in divisors)

The application will handle unit conversion for display. Store in the vehicle's native unit when possible.

### Complex Formula Simplification

For `((signed(A)*256)+B)/327.67`:
1. Recognize that `327.67 ≈ 32767/100`
2. Convert to: `{"len": 16, "sign": true, "mul": 100, "div": 32767}`

For `(value * 1/5) / 10`:
1. Simplify: `value / 50`
2. Convert to: `{"div": 50}`

## Signal ID Naming Conventions

**CRITICAL**: Always follow the existing vehicle's ID prefix pattern and use abbreviations.

### Step 1: Identify the Vehicle Prefix
Before adding new parameters, examine existing signal IDs in the file to determine the vehicle's prefix:
- `TAYCAN_*` for Porsche Taycan
- `MACHE_*` for Ford Mustang Mach-E
- `IONIQ5_*` for Hyundai Ioniq 5
- etc.

### Step 2: Create Abbreviated IDs
Use common abbreviations to keep IDs concise while maintaining clarity:

| Full Term | Abbreviation | Example Usage |
|-----------|--------------|---------------|
| DC-DC Converter | `DCDC` | `TAYCAN_DCDC_V_LOW` |
| Voltage | `V` | `TAYCAN_HVBAT_V` |
| Current | `I` | `TAYCAN_HVBAT_I` |
| Temperature | `T` | `TAYCAN_HVBAT_T_AVG` |
| Battery | `BAT` or `HVBAT` | `TAYCAN_HVBAT_SOC` |
| High Voltage | `HV` | `TAYCAN_HV_DCDC_T` |
| Low Voltage | `LV` | `TAYCAN_LV_BAT_V` |
| State of Charge | `SOC` | `TAYCAN_HVBAT_SOC` |
| Maximum | `MAX` | `TAYCAN_HVBAT_T_MAX` |
| Minimum | `MIN` | `TAYCAN_HVBAT_T_MIN` |
| Average | `AVG` | `TAYCAN_HVBAT_T_AVG` |
| Power | `PWR` | `TAYCAN_HVBAT_PWR` |
| Energy | `E` | `TAYCAN_HVBAT_E_MAX` |
| Reserve/Remaining | `REM` or `RSV` | `TAYCAN_E_REM` |
| Estimated | `EST` | `TAYCAN_RANGE_EST` |
| Internal | `INT` | `TAYCAN_RANGE_INT` |

### Examples of Good vs Bad IDs

❌ **Bad**: `Porsche_19_GATE__DC_DC_CONVERTER_LOW_VOLTAGE`
✅ **Good**: `TAYCAN_DCDC_V_LOW`

❌ **Bad**: `Porsche_19_GATE__MAXIMUM_ENERGY_CONTENT_OF_THE_TRACTION_BATTERY`
✅ **Good**: `TAYCAN_HVBAT_E_MAX`

❌ **Bad**: `Porsche_19_GATE__ESTIMATED_ELECTRIC_POWER_RESERVE__INTERNAL_VALUE_`
✅ **Good**: `TAYCAN_RANGE_EST_INT`

❌ **Bad**: `Porsche_19_GATE__DC_DC_CONVERTER_CURRENT`
✅ **Good**: `TAYCAN_DCDC_I`

### ID Structure Pattern
```
{VEHICLE_PREFIX}_{COMPONENT}_{MEASUREMENT}_{QUALIFIER}
```

Examples:
- `TAYCAN_HVBAT_SOC` (vehicle_component_measurement)
- `TAYCAN_DCDC_V_LOW` (vehicle_component_measurement_qualifier)
- `TAYCAN_HVBAT_T_AVG` (vehicle_component_measurement_qualifier)
- `TAYCAN_RANGE_EST_INT` (vehicle_metric_qualifier_qualifier)

## Signal Naming Conventions

**CRITICAL**: Signal names must be clear, concise, and follow a consistent pattern.

### Name Structure Pattern

Define the **noun first**, then any **modifiers**:

```
{Primary noun}, {modifier}
```

### Formatting Rules

1. **Use sentence casing** (capitalize first word only)
2. **No punctuation at the end**
3. **No parentheticals** - use commas for modifiers instead
4. **No bracketed prefixes** like `[19.gate]` or `[BCM]`
5. **No module/ECU identifiers** in the name
6. **Define the primary subject first**, then qualifiers

### Examples of Good vs Bad Names

❌ **Bad**: `[19.gate] estimated electric power reserve (internal value)`
✅ **Good**: `Electric power reserve, estimated` (with `path: "Battery.Internal"`)

❌ **Bad**: `[BCM] Maximum energy content of the traction battery`
✅ **Good**: `Battery energy, maximum` (with `path: "Battery"`)

❌ **Bad**: `DC-DC converter low voltage (output)`
✅ **Good**: `DC-DC voltage, low` (with `path: "Battery"` for DC-DC converter signals)

❌ **Bad**: `Average temperature of HV battery modules`
✅ **Good**: `Battery temperature, average` (with `path: "Battery"`)

❌ **Bad**: `Internal estimated range value`
✅ **Good**: `Range, estimated` (with `path: "Trips.Internal"`)

### Handling "Internal" Parameters

If a parameter is referenced as "internal" in the source material:
- ✅ **Add `.Internal` to the path** (e.g., `"path": "Battery.Internal"`)
- ❌ **Don't include "internal" in the name** unless it's a critical distinguishing modifier

**Example:**
```json
{
  "id": "TAYCAN_PWR_RSV_EST",
  "path": "Battery.Internal",
  "name": "Power reserve, estimated",
  "fmt": {"len": 16, "div": 100, "unit": "kilowatts"}
}
```

### Common Name Patterns

**DC-DC Converter Signals** (path: `Battery`):
- `DC-DC voltage, low`
- `DC-DC voltage, high`
- `DC-DC current`

**Battery Signals** (path: `Battery`):
- `Battery temperature, maximum`
- `Battery temperature, minimum`
- `Battery temperature, average`
- `Battery power, available`
- `Battery energy, remaining`
- `Battery current, charging`
- `Battery voltage`

**Range/Trip Signals** (path: `Trips`):
- `Range, estimated`
- `Energy consumption, average`

## Signal Format Structure

```json
{
  "hdr": "7E4",           // 11-bit CAN header (3 hex chars) - REQUIRED
  "rax": "7EC",           // Receive address - OPTIONAL
  "cmd": {"22": "4801"},  // Service 22 + PID - REQUIRED
  "freq": 1,              // Polling frequency (seconds) - REQUIRED
  "dbgfilter": {          // Model year filter - OPTIONAL
    "to": 2023,
    "from": 2025
  },
  "signals": [            // Array of signals - REQUIRED
    {
      "id": "SIGNAL_ID",        // Unique identifier - REQUIRED (use vehicle prefix!)
      "path": "Battery",         // Category - OPTIONAL but recommended
      "name": "Human name",      // Display name - REQUIRED (follow naming conventions!)
      "description": "Details",  // Long description - OPTIONAL
      "suggestedMetric": "stateOfCharge",  // UI mapping - OPTIONAL
      "fmt": {                   // Format specification - REQUIRED
        "len": 16,               // Bit length - REQUIRED
        "bix": 0,                // Bit offset - OPTIONAL (default 0)
        "sign": true,            // Signed value - OPTIONAL
        "mul": 1,                // Multiplier - OPTIONAL (default 1)
        "div": 100,              // Divisor - OPTIONAL (default 1)
        "add": -40,              // Offset - OPTIONAL (default 0)
        "min": -50,              // Minimum - OPTIONAL
        "max": 300,              // Maximum - OPTIONAL
        "unit": "celsius"        // Unit type - OPTIONAL
      }
    }
  ]
}
```

## Valid Categories (path)

**IMPORTANT**: Use the most specific path category. Only use `"ECU"` as a last resort when no other category fits.

Standard paths (in order of preference):
- `"Battery"` - HV battery, 12V battery, SOC, DC-DC converter voltage/current
- `"Drivetrain"` - Motors, torque, speed, inverters
- `"Transmission"` - Gear position, temperature
- `"Climate"` - HVAC, temperatures, A/C
- `"Tires"` - Tire pressures
- `"Trips"` - Odometer, range estimates
- `"Movement"` - Vehicle speed
- `"Engine"` - Throttle, grill shutter
- `"Electrical"` - General electrical systems
- `"ECU"` - **LAST RESORT** - Only for ECU-specific monitoring when no other category applies

### Path Selection Examples

**DC-DC Converter signals:**
- ✅ `"Battery"` - DC-DC converter voltage/current (relates to battery charging)
- ❌ `"ECU"` - Too generic
- ❌ `"Electrical"` - Less specific than Battery

**Range/Energy signals:**
- ✅ `"Trips"` - Range estimates, energy consumption
- ✅ `"Battery"` - Battery energy content
- ❌ `"ECU"` - Too generic

## Valid Units

Common unit types:
- `"amps"`, `"volts"`, `"kilowatts"`, `"kilowattHours"`
- `"celsius"`, `"percent"`, `"psi"`, `"hertz"`
- `"rpm"`, `"newtonMeters"`, `"kilometers"`, `"kilometersPerHour"`
- `"seconds"`, `"hours"`

## Lookup Maps (Enumerations)

For `LOOKUP(A:A:0='Off':1='On':2='Fault')`:

```json
{
  "fmt": {
    "len": 8,
    "map": {
      "0": { "description": "Off",   "value": "OFF" },
      "1": { "description": "On",    "value": "ON" },
      "2": { "description": "Fault", "value": "FAULT" }
    }
  }
}
```

**Note**: Values should be SCREAMING_SNAKE_CASE constants.

## Model Year Filters (dbgfilter)

**For new parameters**: Always use `"dbg": true` instead of guessing year filters.

```json
// NEW parameters - use debug mode for testing
{ "hdr": "7E4", "dbg": true, "cmd": {"22": "4850"}, "freq": 1, ...}
```

**For existing parameters with known compatibility**:

```json
// Only 2025 and newer
"dbgfilter": { "from": 2025 }

// Only up to 2023
"dbgfilter": { "to": 2023 }

// 2024-2026 only
"dbgfilter": { "from": 2024, "to": 2026 }

// Skip 2024 (2021-2023 and 2025+)
"dbgfilter": { "to": 2023, "from": 2025 }
```

**Important**: Only add `dbgfilter` after real-world testing confirms which model years support the parameter. Debug mode allows testing across all years.

## Common Headers by Module

| Module | Header | Response | Description |
|--------|--------|----------|-------------|
| BECM | 7E4 | 7EC | HV battery control |
| PCM/ECM | 7E0 | 7E8 | Powertrain control |
| BCCM | 7E2 | 7EA | Charging control |
| Motor (Primary) | 7E6 | 7EE | Drivetrain |
| Motor (AWD) | 7E7 | 7EF | Secondary motor |
| BCM | 726 | 72E | Body control |
| OBCC | 6F5 | 6FD | Onboard charging |
| DCDC | 746 | 74E | DC-DC converter |
| HVAC | 7C7 | - | Climate control |

## Max Value Validation

**CRITICAL**: The `max` value must be mathematically achievable:

```
max = (raw_max_value * mul / div + add)
```

Examples:
✅ 8-bit (0-255) with `div: 10` → `max: 25.5`
✅ 16-bit (0-65535) with `div: 100` → `max: 655.35`
✅ 8-bit with `add: -40` → `max: 215` (when raw max is 255)

❌ 8-bit with `max: 300` and no scaling → INVALID (exceeds 255)

## ⚠️ CRITICAL: Never Modify len/bix Without Reference Material

**ABSOLUTE RULE**: Do NOT change `len` (bit length) or `bix` (bit index) values unless you have explicit reference material.

### What This Means

When reviewing or editing existing signals:
- ✅ You MAY adjust `mul`, `div`, `add` to fix formula calculations
- ✅ You MAY update `max`, `min` to match achievable ranges
- ✅ You MAY change `unit`, `name`, `path` for clarity
- ❌ You MUST NOT change `len` without a spec/CSV that shows the byte length
- ❌ You MUST NOT change `bix` without a spec/CSV that shows the byte offset

### Example of What NOT to Do

**Existing signal:**
```json
{"id": "TAYCAN_BMS_MOD33_CELL1_SOC", "fmt": {"bix": 24, "len": 8, "max": 100, "unit": "percent"}}
```

**❌ WRONG** - Guessing that it should be 16-bit with higher precision:
```json
{"id": "TAYCAN_BMS_MOD33_CELL1_SOC", "fmt": {"bix": 24, "len": 16, "max": 100, "div": 10000, "unit": "percent"}}
```

**✅ CORRECT** - Leave len/bix unchanged, only adjust if needed:
```json
{"id": "TAYCAN_BMS_MOD33_CELL1_SOC", "fmt": {"bix": 24, "len": 8, "max": 100, "unit": "percent"}}
```

### Why This Matters

The `len` and `bix` values determine:
- Which bytes are read from the CAN bus response
- How many bits are interpreted as the value
- Where in the response packet to start reading

**Changing these without evidence will cause:**
- Reading the wrong bytes from the response
- Corrupt or nonsensical data values
- Silent failures that are hard to debug

### When You CAN Change len/bix

Only modify these values when you have:
- A CSV file with explicit byte mappings (e.g., "Bytes A:B" or "INT16(A:B)")
- Official technical documentation showing bit layout
- Specification sheets with response packet structure
- Direct confirmation from testing/reverse engineering documentation

If you're tempted to change `len` or `bix` based on intuition or "this seems wrong," **DON'T**. Ask for clarification or reference material instead.

## Integer Division vs Floating-Point Multiplication

**IMPORTANT RULE**: Prefer integer `div` over floating-point `mul` when an obvious integer-based variation exists.

### Why This Matters

When there's a choice between floating-point multiplication and integer division, integer operations are:
- More precise and predictable
- Faster to compute
- Easier to validate and test
- Consistent across all platforms

### Examples

**❌ AVOID** - Using floating-point when integer division works:
```json
{"fmt": {"len": 8, "max": 100, "mul": 0.5, "unit": "percent"}}
```

**✅ PREFERRED** - Using integer division:
```json
{"fmt": {"len": 8, "max": 127.5, "div": 2, "unit": "percent"}}
```

### Conversion Reference

When you see a decimal multiplier that has an obvious integer division equivalent, use the integer form:

| Decimal Multiplier | Preferred Integer Division | Reasoning |
|--------------------|---------------------------|-----------|
| `× 0.5` | `"div": 2` | 0.5 = 1/2 |
| `× 0.1` | `"div": 10` | 0.1 = 1/10 |
| `× 0.01` | `"div": 100` | 0.01 = 1/100 |
| `× 0.05` | `"div": 20` | 0.05 = 1/20 |
| `× 0.25` | `"div": 4` | 0.25 = 1/4 |
| `× 0.2` | `"div": 5` | 0.2 = 1/5 |

### When Floating-Point mul IS Acceptable

Floating-point `mul` is acceptable when there's no obvious integer equivalent or when specified in reference documentation:

✅ `{"mul": 0.621371}` - Acceptable (km to miles conversion, no simple integer form)
✅ `{"mul": 1.8, "add": 32}` - Acceptable if explicitly documented this way

### Integer mul Is Always Fine

You can freely use `mul` when the multiplier is an integer:

✅ `{"mul": 2}` - Good
✅ `{"mul": 10}` - Good
✅ `{"mul": 50}` - Good

For complex multipliers like `× 1.5`, prefer combined operations:
```json
{"mul": 3, "div": 2}  // Equivalent to × 1.5, preferred over {"mul": 1.5}
```

## Multi-Signal Commands

Some PIDs return multiple values:

```json
{
  "hdr": "7E4",
  "cmd": {"22": "4808"},
  "signals": [
    {"id": "T_MAX", "fmt": { "bix": 0,  "len": 8 }},  // Byte 0
    {"id": "T_MIN", "fmt": { "bix": 8,  "len": 8 }},  // Byte 1
    {"id": "T_RNG", "fmt": { "bix": 16, "len": 8 }},  // Byte 2
    {"id": "T_AVG", "fmt": { "bix": 24, "len": 8 }}   // Byte 3
  ]
}
```

## DIN/DOUT Value Mapping

**CRITICAL**: When mapping signals to DIN/DOUT (diagnostic level in, diagnostic level out) format, follow these exact rules:

### DIN/DOUT Format Requirements

1. **Source material DIN/DOUT values always have a `10` prefix** (e.g., `1003`, `10FF`, `1012`)
2. **JSON `"din"` or `"dout"` field must contain ONLY the last two hex characters** (without the `10` prefix)
3. **DIN/DOUT value must always be a two-character hex string**
4. **Same rules apply to both `din` and `dout` fields**
5. **CRITICAL: A `dout` field must NEVER exist without a corresponding `din` field** (but a `din` may exist without a `dout`)

### DIN/DOUT Mapping Examples

✅ **Correct mappings:**
```json
// Source: 1003
{"din": "03"}

// Source: 10FF
{"din": "FF"}

// Source: 1012
{"dout": "12"}

// Source: 10A5
{"dout": "A5"}
```

❌ **Incorrect mappings:**
```json
// Wrong - includes the 10 prefix
{"din": "1003"}

// Wrong - single character (missing leading zero)
{"din": "3"}

// Wrong - not hex format
{"din": 3}

// Wrong - includes the 10 prefix
{"dout": "1012"}
```

### Full Signal Examples with DIN/DOUT

```json
{
  "hdr": "7E4",
  "cmd": {"22": "4808"},
  "signals": [
    {
      "id": "TAYCAN_HVBAT_SOC",
      "path": "Battery",
      "name": "HV battery state of charge",
      "din": "03",
      "fmt": {
        "len": 8,
        "max": 100,
        "unit": "percent"
      }
    },
    {
      "id": "TAYCAN_HVBAT_V",
      "path": "Battery",
      "name": "HV battery voltage",
      "din": "12",
      "dout": "15",
      "fmt": {
        "len": 16,
        "max": 655.35,
        "div": 100,
        "unit": "volts"
      }
    }
  ]
}
```

**Note**: The second signal shows both `din` and `dout` - this is valid. The first signal shows only `din` - this is also valid. However, a signal with only `dout` (no `din`) would be invalid.

**Important**: When converting CSV data with DIN/DOUT identifiers:
1. Verify the source DIN/DOUT starts with `10`
2. Extract only the last two hex characters
3. Ensure they are uppercase (e.g., `"FF"` not `"ff"`)
4. Always use quotes to make it a string (not a number)

## Implementation Workflow

**When asked to add OBD parameters:**

1. **ANALYZE**
   - Read CSV files and existing signalset
   - **Identify the vehicle's ID prefix** (e.g., `TAYCAN_`, `MACHE_`, `IONIQ5_`)
   - Identify missing PIDs
   - Group parameters by system

2. **CONVERT**
   - Apply formula conversion rules
   - Calculate correct max values
   - Assign proper headers and units
   - **Create abbreviated signal IDs** following the vehicle's prefix pattern

3. **ORGANIZE**
   - Phase 1: Core telemetry (battery, motor)
   - Phase 2: Charging systems (AC/DC)
   - Phase 3: Thermal management
   - Phase 4: Auxiliary systems (12V, DC-DC)
   - Phase 5: Safety monitoring
   - Phase 6: Status/lookup parameters

4. **ADD**
   - Create command entries with properly abbreviated IDs
   - Apply model year filters
   - Validate all formulas

5. **REFORMAT**
   - **IMPORTANT**: When finished making changes to signalset files, always reformat them using:
     ```bash
     find signalsets/v3 -type f -exec python3 tests/schemas/cli.py '{}' --output '{}' \;
     ```
   - This ensures consistent formatting and validates against the schema

## Common Pitfalls to Avoid

❌ Converting temperatures to Fahrenheit
❌ Forgetting the required `len` field
❌ Using invalid units not in the schema
❌ Skipping model year filters when needed
❌ Guessing at formulas instead of following rules
❌ Using invalid path categories
❌ Setting max values that exceed formula output
❌ **Not using the vehicle's existing ID prefix** (e.g., using `Porsche_19_GATE__` instead of `TAYCAN_`)
❌ **Creating overly verbose signal IDs** instead of using standard abbreviations
❌ **Including bracketed prefixes in names** like `[19.gate]` or `[BCM]` - remove these entirely
❌ **Using parentheticals in names** like `(internal value)` - use path modifiers like `.Internal` instead
❌ **Not following noun-first naming** - Always define the primary noun first, then modifiers
❌ **Using Title Case or all lowercase** - Always use sentence casing for signal names
❌ **CRITICAL: Modifying `len` or `bix` values without reference material** - NEVER change these unless you have a CSV, spec sheet, or other source document that explicitly justifies the change
❌ **Using floating-point `mul` when integer `div` exists** - Prefer `{"div": 2}` over `{"mul": 0.5}`
❌ **Using `"ECU"` path when a more specific category exists** - DC-DC converter signals belong in `"Battery"`, not `"ECU"`
❌ **Incorrect DIN/DOUT formatting** - Must be two-character hex string without the `10` prefix (e.g., `"din": "03"` not `"din": "1003"` or `"din": "3"`, same for `dout`)
❌ **Adding `dout` without `din`** - A `dout` field must NEVER exist without a corresponding `din` field

## Example Conversion

**CSV Input:**
```
Name: HV Battery Voltage
PID: 0x22480D
Equation: INT16(A:B)*0.01
Min: 0
Max: 500
Unit: Volts
Header: 7E4
```

**JSON Output:**
```json
{
  "hdr": "7E4",
  "rax": "7EC",
  "cmd": {"22": "480D"},
  "freq": 1,
  "dbgfilter": { "from": 2025 },
  "signals": [
    {
      "id": "MACHE_HVBAT_V",
      "path": "Battery",
      "name": "HV battery voltage",
      "fmt": {
        "len": 16,
        "max": 655.35,
        "div": 100,
        "unit": "volts"
      }
    }
  ]
}
```

**Note**: Max value is 655.35 because `65535 / 100 = 655.35` (16-bit max divided by 100).

## Resources

- Extended PIDs documentation: https://sidecar.clutch.engineering/scanning/extended-pids/
- JSON Schema: https://raw.githubusercontent.com/OBDb/.schemas/refs/heads/main/signals.json

## Your Expertise

You understand OBD-II protocols, CAN bus communication, bit manipulation, data type handling, and formula conversions. You are methodical, precise, and thorough because OBD parameters control critical vehicle systems and must be accurate.
