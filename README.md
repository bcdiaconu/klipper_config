# klipper configuration

## Hardware in use

* Frame: Ender 3 Pro (v1)
  * X axis
    * transmission: belt
    * rail: linear rail
  * Y axis
    * transmission: belt
    * rail: linear rail
  * Z axis
    * transmission: dual lead screw
    * rails: triple V-Wheels on V-Slots
* Main Board: BigTreeTech SKR v1.4 Turbo
* Stepper drivers: BigTreeTech TMC2209
* Display: Original Universal LCD 12864 Creality CR10
* Bed: Creality Glass
* Extruder:
  * Type: Direct
  * Hotend: Mellow BMG Aero Volcano
  * Gears: Mellow BMG Aero

## Install Klipper

Klipper can be easely installed by using KIAUH linux app.

## Install config

### Using SSH

Printer configuration is located at `~/printer_data/config`.

### Using web interface

Printer configuration can be modified in Klipper web by accessing the menu `Configuration` (keyboard shortcut `X`).

## Flash or update MCU firmware

1. `cd ~/klipper/ && make menuconfig`

    * Set the following:

    ```bash
    Micro-controller Architecture (LPC176x (Smoothieboard))
    Processor model (lpc1769 (120 MHz))
    ```

1. `make flash`

## Scripts

The following gcodes are valid for `OrcaSlicer`, `PrusaSlicer` or `SuperSlicer`.

### Start

```gcode
START_PRINT FIRST_LAYER_BED_TEMP=[first_layer_bed_temperature] FIRST_LAYER_NOZZLE_TEMP=[first_layer_temperature]
```

### End

```gcode
END_PRINT
```

## BLTouch offset

### X and Y axes calibration

X and Y axes calibration can be achieved by using the CaliFlower calibrator. [^califlower_calibrator]

### Z axis calibration

1. set a big `z_offset` value under `[bltouch]` section, in `printer.cfg` file
1. restart the firmware
1. make sure the nozzle is clean of plastics; also the the bed surface is flat and free of debree
1. home the machine with `G28`
1. issue a `PROBE_CALIBRATE` command
1. if the nozzle does not move to a position above the automatic probe point, then issue `ABORT` and perform the XY probe offset calibration
1. get the returned Z position and substract or add a distance dz by issuing `TESTZ Z=-dz`
1. check with the thickness of a paper the distance between the nozzle and the bed
1. repet substracting or adding by issuing `TESTZ` command until distance is met; by issuing `TESTZ Z=+` will add (or substract if `Z=-`) half the distance last used
1. when accuracy is met issue a `ACCEPT` command
1. save the setting by issuing `SAVE_CONFIG`
1. restart firmware by issuing `RESTART`

### Test probe accuracy

1. home the machine with `G28`
1. issue a `PROBE_ACCURACY` command and wait for the results

## Mesh

### Creating a mesh

1. make sure the the bed surface free of debree
1. home the machine with `G28`
1. issue `BED_MESH_CALIBRATE` and wait to finish
1. give the profile a name `BED_MESH_PROFILE SAVE=<name>`
1. save the setting by issuing `SAVE_CONFIG`

### Save a mesh

1. give the profile a name `BED_MESH_PROFILE SAVE=<name>`
1. save the setting by issuing `SAVE_CONFIG`

### Load a mesh

1. get a mesh by the profile name `BED_MESH_PROFILE LOAD=<name>`

### Delete a mesh

1. get a mesh by the profile name `BED_MESH_PROFILE REMOVE=<name>`

## PID

### PID for Extruder

Issue a `PID_CALIBRATE HEATER=extruder TARGET=200` command

### PID for Bed

Issue a `PID_CALIBRATE HEATER=heater_bed TARGET=60` command

## Printing temperature

OrcaSlicer provides built-in calibration model [^orca_calibration] for temperature. See calibration menu.

## Retraction settings

OrcaSlicer provides built-in calibration model [^orca_calibration] for retractions. See calibration menu.

Once the proper value was calculated according the described procedure in documentation, update Klipper settings and update the slicer:

1. update the tuned parameter inside `[firmware_retraction]` section
1. issue a `RESTART` command to restart the firmware
1. setup slicer:
    * set 0 retraction length (PrusaSlicer/SuperSlicer: `Printer Settings -> Extruder 1 -> Retraction -> Length`)
    * disable wipe (PrusaSlicer/SuperSlicer: `Printer Settings -> Extruder 1 -> Retraction -> Wipe while retracting`)
    * enable use firmware retractions (PrusaSlicer/SuperSlicer: `Printer Settings -> General -> Advanced -> Use firmware retractions`)

## Skew

Skew can be determined and corrected after determining the parameters for it.

A calibrator for Skew is the CaliFlower calibrator. [^califlower_calibrator]

Commands to add skew data with determined values:

```klipper
SET_SKEW XY=99.79,100.31,70.6
SKEW_PROFILE SAVE=CaliFlower
SAVE_CONFIG
SKEW_PROFILE LOAD=CaliFlower
```

## Resonance compensation [^resonance_measurement]

### Depenedencies

#### Prepare environment

1. Install needed libs

    ```bash
    sudo apt update
    sudo apt install python3-numpy python3-matplotlib libatlas-base-dev libopenblas-dev
    ```

1. Install python libs

    ```bash
    ~/klippy-env/bin/pip install -v numpy
    /usr/bin/python3 -m pip install pyserial --break-system-packages
    ```

1. Enable the SPI on Raspberry PI

    ```bash
    sudo raspi-config
    ```

#### Make the RPI a secondary MCU [^rpi_as_secondary_mcu]

To be able to use the SPI, I2C or any other protocol from Raspberry PI, the RPI board needs to be made a secondary MCU.

Steps below describe how to make Raspberry PI a secondary MCU.

1. Enable the klipper-mcu service

    ```bash
    cd ~/klipper/
    sudo cp ./scripts/klipper-mcu.service /etc/systemd/system/
    sudo systemctl enable klipper-mcu.service
    ```

1. Change configuration to build for the Raspberry PI

    ```bash
    cd ~/klipper/
    sudo cp ./scripts/klipper-mcu.service /etc/systemd/system/
    sudo systemctl enable klipper-mcu.service
    ```

    Set "Microcontroller Architecture" to "Linux process," then save and exit.

1. Build and install the firmware on the Raspberry PI

    ```bash
    sudo service klipper stop
    make flash
    sudo service klipper start
    ```

1. (Optional) If issues with permission add current user `pi` to `tty` group

    ```bash
    sudo usermod -a -G tty pi
    ```

### Test communication

In klipper console type:

`ACCELEROMETER_QUERY CHIP=adxl345`

### Finding the frequency

1. connect the accelerator board to the Raspberry PI [^adxl345_hardware_connection]
1. Find resonance for axis X `TEST_RESONANCES AXIS=X` this will generate a `.csv` file in `/tmp/`
1. Find resonance for axis Y `TEST_RESONANCES AXIS=Y` this will generate a `.csv` file in `/tmp/`
1. Analyze `.csv` files for each of the axis by using the `calibrate_shaper.py` tool

    ```bash
    ~/klipper/scripts/calibrate_shaper.py /tmp/resonances_x_*.csv -o /tmp/shaper_calibrate_x.png
    ~/klipper/scripts/calibrate_shaper.py /tmp/resonances_y_*.csv -o /tmp/shaper_calibrate_y.png
    ```

1. The output of last step:
    1. the console output contains the final result with recommended shaper and the resonance frequency
    1. are two `.png` files which contain details about the analysis results

## Slicer settings

Travel Acceleration:

* 7000 does not skip on X but skips on Y axis

## More Info

[^1]: [KIAUH repository](https://github.com/dw-0/kiauh)

[2]: [Klipper documentation overview](https://github.com/KevinOConnor/klipper/blob/master/docs/Overview.md)

[3]: [Build and Flash the micro controller](https://www.klipper3d.org/Installation.html#building-and-flashing-the-micro-controller)

[4]: [Pressure advance](https://www.klipper3d.org/Pressure_Advance.html)

[5]: [Slicers macros](https://github.com/KevinOConnor/klipper/blob/e7b0e7b43bbf20bf89f47444fbbfc0e10aca1ed1/docs/Slicers.md)

[6]: [Start print macro sample](https://github.com/KevinOConnor/klipper/blob/d36dbfebd17500f0af176abd88d8b258c7940e47/config/printer-lulzbot-taz6-dual-v3-2017.cfg#L216)

[7]: [Retraction Test Object](https://www.thingiverse.com/thing:4532977)

[8]: [Firmware Retraction](https://www.klipper3d.org/Config_Reference.html#firmware_retraction)

[^resonance_measurement]: [Measuring resonances](https://www.klipper3d.org/Measuring_Resonances.html)

[10]: [Resonance compensation](https://www.klipper3d.org/Resonance_Compensation.html)

[^rpi_as_secondary_mcu]: [RPi microcontroller as a secondary MCU](https://www.klipper3d.org/RPi_microcontroller.html)

[^adxl345_hardware_connection]: [ADXL345 SPI hardware connection](https://www.klipper3d.org/Measuring_Resonances.html#direct-to-raspberry-pi)

[^orca_calibration]: [OrcaSlicer calibration](https://github.com/SoftFever/OrcaSlicer/wiki/Calibration)

[^califlower_calibrator]: [CaliFlower Calibrator] (https://vector3d.shop/products/califlower-calibration)
