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

| CSV Formula | Convert To | Explanation |
|-------------|------------|-------------|
| `A` | `{}` | Raw value, no conversion |
| `A*0.5` | `{"div": 2}` | Multiply by 0.5 = divide by 2 |
| `A*2` | `{"mul": 2}` | Direct multiplication |
| `A*0.01` | `{"div": 100}` | 0.01 = 1/100 |
| `A-40` | `{"add": -40}` | Subtract 40 |
| `A*0.05+6` | `{"div": 20, "add": 6}` | Divide by 20, then add 6 |

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
      "name": "Human name",      // Display name - REQUIRED
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

Use these standard paths:
- `"Battery"` - HV battery, 12V battery, SOC
- `"Drivetrain"` - Motors, torque, speed, inverters
- `"Transmission"` - Gear position, temperature
- `"Climate"` - HVAC, temperatures, A/C
- `"Tires"` - Tire pressures
- `"Trips"` - Odometer
- `"Movement"` - Vehicle speed
- `"Engine"` - Throttle, grill shutter
- `"ECU"` - ECU monitoring
- `"Electrical"` - DC-DC converter

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
