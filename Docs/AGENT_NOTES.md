# Agent Notes for AXe

This document captures learnings and best practices discovered while using AXe for simulator automation. It's intended to help future AI agents (and humans) avoid re-learning these lessons.

## Quick Start

```bash
# Build
swift build

# List simulators (find booted ones)
swift run axe list-simulators | grep Booted

# Describe UI hierarchy
swift run axe describe-ui --udid <SIMULATOR_UDID>

# Interact with elements
swift run axe tap -x <X> -y <Y> --udid <SIMULATOR_UDID>
swift run axe touch -x <X> -y <Y> --down --up --delay 0.1 --udid <SIMULATOR_UDID>
```

## Critical Finding: `tap` vs `touch` Commands

### Problem
The `tap` command may report success but fail to actually trigger UI interactions (especially toggles/switches) on some simulators.

### Solution
Use the `touch` command with explicit `--down --up` flags and a small delay:

```bash
# This may NOT work reliably:
swift run axe tap -x 883 -y 56 --udid <UDID>

# This WORKS reliably:
swift run axe touch -x 883 -y 56 --down --up --delay 0.1 --udid <UDID>
```

### Why This Matters
- The `tap` command uses `FBSimulatorHIDEvent.tapAt(x:y:)` which combines down+up in one event
- The `touch` command with `--delay` provides explicit timing control between touch down and touch up
- Some UI elements (especially SwiftUI Toggle/Switch) seem to require this explicit timing

### Recommended Approach
1. **First try**: Use `tap` for simple interactions
2. **If `tap` fails**: Switch to `touch --down --up --delay 0.1`
3. **For critical automation**: Default to `touch` with delay for reliability

## Calculating Tap Coordinates

UI elements report their frame as `{{x, y}, {width, height}}`. To tap the center:

```
center_x = x + (width / 2)
center_y = y + (height / 2)
```

Example for frame `{{852, 42}, {61, 28}}`:
- Center X: 852 + 30.5 = 882.5
- Center Y: 42 + 14 = 56

## Workflow for Interacting with UI Elements

1. **Get UI hierarchy**:
   ```bash
   swift run axe describe-ui --udid <UDID> 2>&1 | tail -n +10 > ui.json
   ```

2. **Find target element** - look for:
   - `AXLabel` - accessibility label (e.g., "Scan Devices")
   - `role` / `subrole` - element type (e.g., "AXCheckBox", "AXSwitch")
   - `frame` - coordinates for tapping
   - `AXValue` - current state (e.g., "0" for off, "1" for on)

3. **Calculate center coordinates** from the `frame`

4. **Interact using `touch`**:
   ```bash
   swift run axe touch -x <center_x> -y <center_y> --down --up --delay 0.1 --udid <UDID>
   ```

5. **Verify the change**:
   ```bash
   swift run axe describe-ui --udid <UDID> 2>&1 | grep -E 'AXValue|AXSwitch'
   ```

## Common Element Types

| UI Element | role | subrole | AXValue |
|------------|------|---------|---------|
| Toggle/Switch | AXCheckBox | AXSwitch | "0" or "1" |
| Button | AXButton | null | null |
| Text | AXStaticText | null | null |
| Image | AXImage | null | null |
| Text Field | AXTextField | null | current text |

## Filtering Output

The commands output build info and warnings to stderr. To get clean JSON:

```bash
# Redirect stderr, keep stdout
swift run axe describe-ui --udid <UDID> 2>/dev/null

# Or skip first N lines of output
swift run axe describe-ui --udid <UDID> 2>&1 | tail -n +10
```

## Known Warnings (Safe to Ignore)

```
objc[XXXX]: Class FBProcess is implemented in both .../FrontBoard.framework and .../FBControlCore.framework
```

This warning about duplicate class implementations is cosmetic and doesn't affect functionality.

## Environment Variables

- `AXE_HID_STABILIZATION_MS` - Stabilization delay after HID events (default: 25ms, max: 1000ms)

## Tested Configurations

| Date | Simulator | iOS Version | Working Commands |
|------|-----------|-------------|------------------|
| 2024-12-14 | iPad Pro 13-inch (M5) | iOS 26.1 | `touch --down --up --delay 0.1` ✅, `tap` ❌ (unreliable) |

## Tips for Future Agents

1. **Always verify interactions** - Run `describe-ui` before and after to confirm state changes
2. **Prefer `touch` over `tap`** - More reliable for critical interactions
3. **Check simulator is booted** - Commands will fail on shutdown simulators
4. **Use delays for sequences** - Add `--pre-delay` or `--post-delay` when chaining commands
5. **Parse JSON carefully** - Output includes build logs; filter them out before parsing
