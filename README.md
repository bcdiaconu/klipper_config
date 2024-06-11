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

`cd ~/klipper/ && make menuconfig`

Set the following:

```bash
Micro-controller Architecture (LPC176x (Smoothieboard))
Processor model (lpc1769 (120 MHz))
```

## Scripts

### Start

For `PrusaSlicer` or `SuperSlicer` the startup gcode would be similar to:

```gcode
START_PRINT FIRST_LAYER_BED_TEMP=[first_layer_bed_temperature] FIRST_LAYER_NOZZLE_TEMP=[first_layer_temperature]
```

### End

```gcode
END_PRINT
```

## BLTouch offset

### X and Y axes calibration

1. home the machine with `G28`
1. place peace of tape on the center of the bed
1. move the bltouch in a convenient postion just on top of the tape and such that when the probe is deployed it will touch the bed but still be almost fully extended
1. deploy the probe with `BLTOUCH_DEBUG COMMAND=pin_down`
1. mark the location where the probe touches the tape
1. retract the probe with `BLTOUCH_DEBUG COMMAND=reset`
1. start probing: `probe`
1. get current position by issuing `GET_POSITION`
1. note the `toolhead` coordinates
1. move the nozzle on top of the mark with `G1` command eg: `G1 F300 X118.5 Y116.5 Z1.8`; repeat move until the nozzle sits just on top of the mark made on tape
1. to be sure, get current position by issuing `GET_POSITION` and receiving
1. probe_offset_x = nozzle_x_position - probe_x_position; probe_offset_y = nozzle_y_position - probe_y_position
1. save configuration in `printer.cfg` under `[bltouch]` cathegory
1. restart firmware by issuing `RESTART`
1. remove the tape off the bed

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

1. get a good model to test on (eg: [Smart compact temperature calibration tower](https://www.thingiverse.com/thing:2729076 "Smart compact temperature calibration tower on thingiverse"))
1. set each section to appropriate temperature
1. slice the model
1. print the model
1. update the slicer configuration based on findings (!) this might be influenced by other factors like pressure advance and retractions

## Retraction settings

1. set inside `[firmware_retraction]` section, the initial values for parameters `retract_length`, `retract_speed`, `unretract_extra_length`, `unretract_speed`
1. restart firmware by issuing `RESTART`
1. issue a `TUNING_TOWER` command that tunes a specific `parameter` (eg: for retraction length: `TUNING_TOWER COMMAND=SET_RETRACTION PARAMETER=LENGTH START=0 FACTOR=.01`)
    * syntax: `TUNING_TOWER COMMAND=<command_to_issue> PARAMETER=<command_parameter> START=<initial_value> FACTOR=<increase_factor_per_layer>`
    * command_parameter for `SET_RETRACTION` are: `RETRACT_LENGTH`, `RETRACT_SPEED`, `UNRETRACT_EXTRA_LENGTH`, `UNRETRACT_SPEED`
    * for feedback reasons, can be used a custom macro named `SET_RETRACTIONLENGTH` issuing `TUNING_TOWER COMMAND=SET_RETRACTIONLENGTH PARAMETER=RETRACT_LENGTH START=0 FACTOR=.01`

    ```conf
    [gcode_macro SET_RETRACTIONLENGTH]
    gcode:
      SET_RETRACTION RETRACT_LENGTH={params.RETRACT_LENGTH|firmware_retraction.retract_length|float} RETRACT_SPEED={params.RETRACT_SPEED|firmware_retraction.retract_speed|float} UNRETRACT_EXTRA_LENGTH={params.UNRETRACT_EXTRA_LENGTH|firmware_retraction.unretract_extra_length|float} UNRETRACT_SPEED={params.UNRETRACT_SPEED|firmware_retraction.unretract_speed|float}
      GET_RETRACTION
    ```

1. setup slicer:
    * set 0 retraction length (PrusaSlicer/SuperSlicer: `Printer Settings -> Extruder 1 -> Retraction -> Length`)
    * disable wipe (PrusaSlicer/SuperSlicer: `Printer Settings -> Extruder 1 -> Retraction -> Wipe while retracting`)
    * enable use firmware retractions (PrusaSlicer/SuperSlicer: `Printer Settings -> General -> Advanced -> Use firmware retractions`)
1. slice the model to resulting `gcode` file
1. check the `gcode` file:
    * that it contains the `G10` and `G11` instructions
    * that it does not contain any negative extrusion (eg: `G1 X18 Y32 E-3.42`)
1. start printing the sliced object
1. with calipers measure the height from bottom till the layer that seems to produce best results (`measured_height`)
1. calculate the pressure advance by applying formula `<parameter> = <START> + <measured_height> * <FACTOR>`
1. update the tuned parameter inside `[firmware_retraction]` section
1. issue a `RESTART` command to restart the firmware

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
    ~/klippy-env/bin/pip install -v serial
    ```

1. Enable the SPI on Raspberry PI

    ```bash
    sudo raspi-config
    ```

#### Make the RPI a secondary MCU [^rpi_as_secondary_mcu]

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

## Pressure (Linear) advance

1. send the follwing command to printer `SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=1 ACCEL=500`
1. and set the pressure_advance start and increment factor by sending either of the following commands:
    * syntax: `TUNING_TOWER COMMAND=<command_to_issue> PARAMETER=<command_parameter> START=<initial_value> FACTOR=<increase_factor_per_layer>`
    1. for direct drive `TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.005`
    1. for bowden `TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.020`
1. download the file from [here](https://www.klipper3d.org/prints/square_tower.stl "stl file to download")
1. setup the printer:
    * use high printing speeds (eg 100 mm/s) (`Print Settings -> Speed -> Speed for print moves -> all with mm/s`)
    * infill 0% (`Print settings -> Infill -> Infill -> Sparse`)
    * layer height 75% of nozzle diameter (0.3 for 0.4 nozzle) (`Print Settings -> Slicing -> Layer height  -> Base layer height`)
1. check the layer speeds inside silicer (on the sliced object); if speeds are not the set ones:
    * increase the number of permiter walls (`Print Settings -> Permiters & Shell -> Vertical shells -> Perimteres`)
    * increase the infill (`Print settings -> Infill -> Infill -> Sparse`)
1. when everything checks out, slice and print
1. with calipers measure the height from bottom till the layer that seems to produce best results (`measured_height`)
1. calculate the pressure advance by applying formula `pressure_advance = <START> + <measured_height> * <FACTOR>`
1. under `[extruder]` section, add the resulted value eg: `pressure_advance: 0.0614`
1. issue a `RESTART` command to restart the firmware

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
