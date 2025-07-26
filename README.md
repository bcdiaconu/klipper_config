# Klipper Configuration

## Table of Contents

1. [Hardware Overview](#hardware-overview)
1. [Software Installation](#software-installation)
1. [Configuration Setup](#configuration-setup)
1. [File Structure](#file-structure)
1. [Calibration Procedures](#calibration-procedures)
1. [Slicer Integration](#slicer-integration)
1. [Maintenance & Troubleshooting](#maintenance--troubleshooting)
1. [References](#references)

## Hardware Overview

### 3D Printer Specifications

* **Frame**: Ender 3 Pro (v1)
  * **X axis**: Belt transmission with linear rail
  * **Y axis**: Belt transmission with dual linear rail
  * **Z axis**: Dual lead screw transmission with triple V-Wheels on V-Slots, synchronized by belt
* **Main Board**: BigTreeTech SKR v1.4 Turbo (LPC1769 MCU)
* **Stepper Drivers**: BigTreeTech TMC2209 (silent, sensorless homing capable)
* **Display**: Original Universal LCD 12864 Creality CR10
* **Bed**: Creality Glass bed with heated aluminum plate
* **Extruder**: Direct drive system
  * **Hotend**: Creality Sprite Pro (all-metal, up to 300°C)
  * **Gears**: Creality Sprite Pro (3:1 gear ratio)
* **Auto-leveling**: 3DTouch v3.0 (BLTouch clone) probe for bed mesh compensation

## Software Installation

### KIAUH Installation

Klipper can be easily installed using KIAUH[^kiauh_repo] - a comprehensive installation assistant for Raspberry Pi.

```bash
cd ~ && git clone https://github.com/dw-0/kiauh.git
./kiauh/kiauh.sh
```

### Required Applications

#### Core Components

* **Klipper**: Advanced 3D printer firmware that runs on the host computer
* **Moonraker**: API web server for Klipper (enables web interface communication)

#### Web Interfaces (choose one)

* **Fluidd**: Modern, responsive web interface for Klipper
* **Mainsail**: Alternative web interface with different UI/UX approach

#### Optional Components

* **KlipperScreen**: Touch screen interface for direct printer control
* **Mobileraker**: Mobile companion app for remote printer monitoring
* **Spoolman**: Filament spool management system for tracking usage
* **Crowsnest**: Camera streaming service for print monitoring

## Configuration Setup

### Installation Methods

#### Using SSH

```bash
# Navigate to configuration directory
cd ~/printer_data/config

# Clone or copy configuration files
git clone <this-repository> .
```

#### Using Web Interface

1. Access Fluidd web interface
2. Navigate to `Configuration` menu (keyboard shortcut `X`)
3. Upload configuration files directly through the interface

### MCU Firmware Flashing

#### Automatic Flashing (KIAUH)

1. Run KIAUH interface
2. Choose `Advanced` menu
3. Select `Build + Flash`
4. Follow prompts for LPC1769 configuration

#### Manual Flashing

```bash
cd ~/klipper/ && make menuconfig
```

Set configuration:

* **Micro-controller Architecture**: LPC176x (Smoothieboard)
* **Processor model**: lpc1769 (120 MHz)

```bash
make flash
```

#### Alternative: FlashMagic (Windows)

The `flashmagic/` directory contains FlashMagic project files for flashing firmware using the Windows-based FlashMagic utility. This is useful for initial firmware installation or recovery.

## File Structure

```md
klipper_config/
├── printer.cfg              # Main configuration file (includes all others)
├── frame.cfg                # Printer kinematics and motion settings
├── creality_sprite_pro.cfg  # Extruder and hotend configuration
├── skr_1.4_turbo.cfg        # Motherboard pin definitions and settings
├── creality_lcd.cfg         # Display configuration
├── bl_touch.cfg             # BLTouch probe settings
├── accelerometers.cfg       # Input shaping sensor configuration
├── macros.cfg               # Custom G-code macros
├── fluidd.cfg               # Fluidd web interface settings
├── crowsnest.cfg            # Camera streaming configuration
├── moonraker.conf           # Moonraker API server configuration
├── orca_slicer/             # Slicer profiles directory
│   └── *.json               # OrcaSlicer printer profiles
├── flashmagic/              # FlashMagic project files
│   └── *.fmx                # MCU flashing configurations
└── README.md                # This documentation
```

### Configuration Files Explained

* **printer.cfg**: Main entry point that includes all other configuration modules
* **frame.cfg**: Defines printer geometry, speeds, accelerations, and bed settings
* **skr_1.4_turbo.cfg**: Pin assignments and hardware-specific settings for the motherboard
* **creality_sprite_pro.cfg**: Extruder motor settings, thermistor configuration, and retraction
* **bl_touch.cfg**: Probe offsets, homing behavior, and mesh settings
* **macros.cfg**: Custom commands for start/end print sequences and utilities

## Calibration Procedures

### Initial Setup Sequence

#### 1. BLTouch Probe Calibration

##### XY Offset Calibration

Use CaliFlower[^califlower_calibrator] or CaliLantern[^calilantern_calibrator] calibrators for precise probe positioning.

##### Z Offset Calibration

```gcode
G28                    # Home all axes
PROBE_CALIBRATE        # Start calibration
TESTZ Z=-0.1          # Adjust nozzle height
ACCEPT                # Save when paper drag is correct
SAVE_CONFIG           # Persist settings
```

#### 2. PID Tuning

```gcode
# Extruder PID (200°C target)
PID_CALIBRATE HEATER=extruder TARGET=200

# Bed PID (60°C target)
PID_CALIBRATE HEATER=heater_bed TARGET=60
```

#### 3. E-Steps Calibration

1. Mark filament 120mm from extruder entry
2. Heat extruder to printing temperature
3. Extrude 100mm: `G1 E100 F60`
4. Measure actual extrusion
5. Calculate new rotation_distance:

   ```md
   new_rotation_distance = current_value × (commanded_mm / actual_mm)
   ```

#### 4. Bed Mesh Creation

```gcode
G28                           # Home all axes
BED_MESH_CALIBRATE           # Generate mesh
BED_MESH_PROFILE SAVE=default # Save profile
SAVE_CONFIG                  # Persist to disk
```

#### 5. Resonance Compensation

##### Prerequisites

```bash
# Install dependencies
sudo apt update
sudo apt install -y python3-numpy python3-matplotlib libatlas-base-dev

# Configure Raspberry Pi as secondary MCU
cd ~/klipper/
sudo cp ./scripts/klipper-mcu.service /etc/systemd/system/
sudo systemctl enable klipper-mcu.service
make menuconfig  # Set to "Linux process"
make flash
```

##### Frequency Testing

```gcode
# Test X axis resonance
TEST_RESONANCES AXIS=X

# Test Y axis resonance  
TEST_RESONANCES AXIS=Y

# Analyze results
~/klipper/scripts/calibrate_shaper.py /tmp/resonances_x_*.csv -o /tmp/shaper_x.png
~/klipper/scripts/calibrate_shaper.py /tmp/resonances_y_*.csv -o /tmp/shaper_y.png
```

#### 6. Skew Correction

```gcode
# Set skew values from calibration print
SET_SKEW XY=99.79,100.31,70.6
SKEW_PROFILE SAVE=CaliFlower
SAVE_CONFIG
```

### Advanced Calibration

#### Pressure Advance Tuning

Use OrcaSlicer's built-in calibration models[^orca_calibration] or print a calibration tower. Typical values for direct drive: 0.02-0.08.

#### Retraction Tuning

Configure firmware retraction for optimal stringing control:

```ini
[firmware_retraction]
retract_length: 0.8      # Distance to retract
retract_speed: 40        # Retraction speed
unretract_speed: 40      # Prime speed
```

## Slicer Integration

### OrcaSlicer Configuration

Pre-configured profiles are available in `orca_slicer/` directory:

* **Creality Ender-3 0.4 nozzle Klipper.json**: Optimized profile for this printer
* Includes remote printing setup (`print_host: 3dprinter001.iveronsoft.ro`)
* Firmware retraction enabled (set slicer retraction to 0)

### Start/End G-code

```gcode
# Start G-code
START_PRINT FIRST_LAYER_BED_TEMP=[first_layer_bed_temperature] FIRST_LAYER_NOZZLE_TEMP=[first_layer_temperature]

# End G-code
END_PRINT
```

### Remote Printing Setup

1. Ensure `print_host` in slicer profile matches your printer's IP
2. Configure Moonraker to accept API requests
3. Set API key if authentication is enabled

## Maintenance & Troubleshooting

### Regular Maintenance

* **Bed Mesh**: Re-calibrate monthly or after bed changes
* **PID Values**: Re-tune after hardware changes
* **Belt Tension**: Check and adjust periodically
* **Linear Rails**: Clean and lubricate as needed

### Common Issues

* **Layer shifting**: Check belt tension, verify TMC2209 current settings
* **First layer problems**: Re-calibrate Z-offset, check bed mesh
* **Temperature fluctuations**: Re-run PID tuning
* **Stringing**: Adjust retraction settings, verify hotend temperature

### Firmware Updates

1. Update Klipper: Use KIAUH's update function
2. Update MCU firmware: Flash new firmware when Klipper updates require it
3. Backup configurations before major updates

## Camera Setup

### Crowsnest Installation

1. Install via KIAUH (select `Y` to update Moonraker configuration)
2. Copy `crowsnest.cfg` to printer configuration directory
3. Add camera in Fluidd/Mainsail settings
4. Restart Klipper host service

## Performance Notes

* **Max Travel Acceleration**: 7000 mm/s² (may cause Y-axis skipping above this value)
* **Max Print Acceleration**: 4500 mm/s² (limited by Y-axis resonance)
* **Recommended Print Speeds**: 60-80 mm/s for quality, up to 120 mm/s for speed

## References

[^kiauh_repo]: [KIAUH Repository](https://github.com/dw-0/kiauh)
[^orca_calibration]: [OrcaSlicer Calibration](https://github.com/SoftFever/OrcaSlicer/wiki/Calibration)
[^califlower_calibrator]: [CaliFlower Calibrator](https://vector3d.shop/products/califlower-calibration)
[^calilantern_calibrator]:
