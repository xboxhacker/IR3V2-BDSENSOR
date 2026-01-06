# IR3v2 BDsensor Klipper Module

A modified BDsensor Klipper module adapted for **belt printers** like the IR3v2. This fork contains specific changes to make the BDsensor work correctly with continuous belt printing systems where the Y-axis is the belt direction.

## Based On

This project is a modified version of the original [BDsensor (Bed Distance Sensor)](https://github.com/markniu/Bed_Distance_sensor) by Mark Niu.

## Key Modifications for Belt Printers

### 1. Y-Axis Calibration Routine

**Original behavior:** The `BDSENSOR_CALIBRATE` command moves the Z-axis from 0mm to 4mm in 0.1mm increments to build the calibration data.

**Modified behavior:** The calibration routine now moves the **Y-axis** instead of Z-axis:

```python
# Start from current Y position and move in positive Y direction
y_pos = self.toolhead.get_position()[1]
while ncount < 40:
    y_pos += 0.1  # Move Y in positive direction
    self.toolhead.manual_move([None, y_pos, None], 2)  # Move only Y
    ncount += 1
```

This change is essential because:
- Belt printers, like the IR3V2.
- The calibration must sample different points on the belt surface
- Z-axis movement would not provide meaningful calibration data on a belt system

### 2. PROBE_ACCURACY Y-Axis Retracts

**Original behavior:** `PROBE_ACCURACY` command performs Z retracts between samples.

**Modified behavior:** Uses **Y-axis retracts** instead:

```python
# Move to target Y (Z fixed)
self._move([None, target_y, None], speed)
# Retract Y if more samples needed
retract_y = target_y + sample_retract_dist
self._move([None, retract_y, None], lift_speed)
```

This ensures probe accuracy testing works correctly on a belt surface where Z retraction doesn't make sense.

### 3. Disabled Collision Features

For belt printers, the collision sensing features are disabled by default:

```ini
collision_homing: 0
collision_calibrate: 0
```

These features are designed for standard bed printers and can cause issues with belt configurations.

### 4. Custom Homing Command Fallback

Added support for a fallback homing command:

```ini
homing_cmd: G990028
```

This allows the use of a renamed original G28 command for proper homing sequences on the IR3v2.

## Installation

> ⚠️ **Important:** You must install the original BDsensor first before applying these modifications!

### 1. Install Original BDsensor (Required First)

Before using this modified version, you **must** complete the full installation of the original BDsensor from the official repository:

1. Visit the official BDsensor repository: https://github.com/markniu/Bed_Distance_sensor
2. Follow **ALL** installation instructions in their Wiki, including:
   - Hardware installation and wiring
   - Firmware flashing to the BDsensor
   - Klipper module installation
   - MCU configuration
   - Initial sensor verification

**Do not skip any steps** - the original installation sets up essential components that this modification depends on.

### 2. Install IR3v2 Modified Script

After the original BDsensor is fully installed and working, replace the Klipper module with this modified version:

```bash
cp klipper/klippy/extras/BDsensor.py ~/klipper/klippy/extras/BDsensor.py
```

This overwrites the original `BDsensor.py` with the belt-printer optimized version.

### 3. Restart Klipper

```bash
sudo systemctl restart klipper
```

### 4. Update Configuration

Update your BDsensor configuration for belt printer use. Add to your `printer.cfg` or include the provided `bdsensor.cfg`:

```ini
[include bdsensor.cfg]
```

Or copy the configuration directly:

```ini
[BDsensor]
scl_pin: host:gpio3
sda_pin: host:gpio2
delay: 10

# Sensor offsets (adjust these after installation)
z_offset: 0.0   # Calibrate this after mounting
x_offset: 15    # Distance from nozzle to sensor in X
y_offset: 0     # Distance from nozzle to sensor in Y

# Enable BDSensor for homing
position_endstop: 1.2
speed: 3
no_stop_probe: True  # Fast probe - toolhead won't stop at probe points

# For belt printer - disable these for reliable homing
collision_homing: 0
collision_calibrate: 0

# Use the original G28 command for fallback
homing_cmd: G990028
```

### 5. Wiring (Reference)

If you haven't already configured your wiring during the original installation, connect the BDsensor to your Raspberry Pi GPIO:
- **SCL** → GPIO3
- **SDA** → GPIO2
- **GND** → Ground
- **VCC** → 3.3V or 5V (check your BDsensor version)

## Usage

### Calibration

**Important:** On a belt printer, the calibration works differently:

1. Position the nozzle at the correct height above the belt (approximately paper thickness - touching)
2. Run `BDSENSOR_CALIBRATE`
3. The Y axis will lift and calibrate the sensor reading as it does
4. Wait for "Calibrate Finished!" message (No need to save anything)

### Commands

| Command | Description |
|---------|-------------|
| `BDSENSOR_CALIBRATE` | Calibrate the sensor (moves Y-axis on belt printers) |
| `BDSENSOR_VERSION` | Display sensor firmware version |
| `BDSENSOR_DISTANCE` | Read current distance measurement |
| `BDSENSOR_READ_CALIBRATION` | Display stored calibration data |
| `PROBE_ACCURACY` | Test probe repeatability (uses Y retracts) |
| `M102 S-1` | Read version |
| `M102 S-2` | Read distance |
| `M102 S-5` | Read calibration data |
| `M102 S-6` | Start calibration |

## Differences from Original BDsensor

| Feature | Original | IR3v2 Modified |
|---------|----------|----------------|
| Calibration axis | Z-axis movement | Y-axis movement |
| PROBE_ACCURACY retracts | Z-axis | Y-axis |
| Collision homing | Supported | Disabled |
| Collision calibrate | Supported | Disabled |
| Target platform | Standard bed printers | Belt printers |

## Troubleshooting

### Sensor reads "Connection Error"
- Check wiring connections
- Verify GPIO pins are correct in config
- Ensure Klipper has permission to access GPIO

### Calibration fails
- Make sure the nozzle is at the correct starting height
- Verify there's enough Y travel available (needs ~4mm of movement)
- Check that the belt surface is clean and flat

### "Out of Range" errors
- The sensor may be mounted too high (>4mm from surface)
- For belt printers, ensure the belt isn't sagging in the measurement area

## Credits

- **Original BDsensor:** [Mark Niu](https://github.com/markniu/Bed_Distance_sensor)
- **IR3v2 Modifications:** Xboxhacker v3

## License

This project is licensed under the GNU General Public License v3.0 - see the [LICENSE](LICENSE) file for details.

This is a modified version of the original BDsensor software. Per GPLv3 requirements, this derivative work is also released under GPLv3.
