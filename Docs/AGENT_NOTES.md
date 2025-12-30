# Agent Notes for AXe

This document captures learnings and best practices discovered while using AXe for simulator automation. It's intended to help future AI agents (and humans) avoid re-learning these lessons.

## Quick Start

```bash
# Build
swift build -c release

# List simulators (find booted ones)
.build/release/axe list-simulators | grep Booted

# Describe UI hierarchy
axe describe-ui --udid <SIMULATOR_UDID>

# Tap by coordinates
axe tap -x <X> -y <Y> --udid <SIMULATOR_UDID>

# Tap by accessibility label (preferred)
axe tap --label "Scan Devices" --udid <SIMULATOR_UDID>
```

## Tap Command (Reliable as of 2024-12-29)

The `tap` command now uses a reliable touch down/delay/touch up pattern internally:

```bash
# Tap by coordinates
axe tap -x 883 -y 56 --udid <UDID>

# Tap by accessibility label (preferred - finds element and taps center)
axe tap --label "Scan Devices" --udid <UDID>
axe tap --id "myAccessibilityIdentifier" --udid <UDID>
```

### Implementation Details
- Uses `touchDownAt` → 100ms delay → `touchUpAt` pattern
- This replaced the atomic `tapAt()` which was unreliable (~30% success)
- The `touch` command is still available for advanced use cases

### Optional Timing Controls
```bash
axe tap --label "Button" --pre-delay 0.5 --udid <UDID>   # Wait before tap
axe tap --label "Button" --post-delay 0.5 --udid <UDID>  # Wait after tap
```

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
| 2024-12-29 | iPad Pro 13-inch (M5) | iOS 26.1 | `tap` ✅, `tap --label` ✅, `touch` ✅ |
| 2024-12-29 | Apple Vision Pro | visionOS | `tap` ❌ (broken frames), `screenshot` ✅ |

## Tips for Future Agents

1. **Use `tap --label` when possible** - Finds element by accessibility label, taps center
2. **Check simulator is booted** - Commands will fail on shutdown simulators
3. **Use delays for sequences** - Add `--pre-delay` or `--post-delay` when chaining commands
4. **Parse JSON carefully** - Output includes build logs; filter them out with `2>/dev/null`
5. **visionOS has broken tap** - Use manual testing for Pfizer/GMP apps
