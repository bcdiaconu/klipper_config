# klipper configuration

## Hardware in use

* Frame: Ender 3 Pro (v1)
* Mainboard: BigTreeTech SKR v1.4 Turbo
* Stepper drivers: BigTreeTech TMC2209
* Display: Original Universal LCD 12864 Creality CR10
* Bed: Creality Glass
* Extruder:
  * Type: Direct
  * Hotend: Mellow BMG Aero Volcano
  * Gears: Mellow BMG Aero

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

## Resonance compensation

### Finding the frequency

1. download the `ringing_tower` model from [here](https://www.klipper3d.org/prints/ringing_tower.stl)
1. slice the model with settings:
    * layer height 0.2
    * enable vase mode (`Print Settings -> Permiteres&Shell -> Vertical shells -> Spiral vase`) or
        * perimeters 1 - 2 (`Print Settings -> Permiteres&Shell -> Vertical shells -> Perimteres`)
        * top layer 0 (`Print Settings -> Permiteres&Shell -> Horizontal shells -> Solid layers`)
        * infill 0% (`Print settings -> Infill -> Infill -> Sparse`)
    * bottom layer 2mm (`Print Settings -> Permiteres&Shell -> Horizontal shells -> Minimum shell thickness -> Bottom`)
    * external perimtere speed 100mm/s (`Print Settings -> Speed -> Speed for print moves -> Perimter speed`)
    * minimum layer time at most 3s (`Filament Settings -> Cooling -> Very short layer time -> Layer time goal`)
    * disable dynamic acceleration control (``)
1. setup Klipper:
    1. `SET_VELOCITY_LIMIT VELOCITY=500 ACCEL=7000 ACCEL_TO_DECEL=7000 SQUARE_CORNER_VELOCITY=5.0`
    1. `SET_PRESSURE_ADVANCE ADVANCE=0`
    1. `SET_INPUT_SHAPER SHAPER_FREQ_X=0 SHAPER_FREQ_Y=0`
    1. `TUNING_TOWER COMMAND=SET_VELOCITY_LIMIT PARAMETER=ACCEL START=1250 FACTOR=100 BAND=5`; this command will increase the acceleration every 5 mm starting from 1250 mm/sec^2 and rising with `5*100` each step
1. print model
1. measure the distance between several oscillations on both axes (`=>` `D_x`, `D_y`)
1. count the number of oscillations between measurements (starting from 0) (`=>` `N_x`, `N_y`)
1. compute the ringing frequency (`F_k=V_k*N_k/D_k` `[Hz]`); `V` is the printing velocity of outer shells
1. add the `[input_shaper]` section in printer config file and the calculated values for frequency  `shaper_freq_x: <value>` and `shaper_freq_y: <value>`
1. issues a `FIRMWARE_RESTART` command to restart the firmware

### Fine tuning

1. setup Klipper
    1. set velocity and acceleration to a high value `SET_VELOCITY_LIMIT VELOCITY=500 ACCEL=7000 ACCEL_TO_DECEL=7000 SQUARE_CORNER_VELOCITY=5.0`
    1. disable pressure advance `SET_PRESSURE_ADVANCE ADVANCE=0`
    1. set the algorithm to MZV `SET_INPUT_SHAPER SHAPER_TYPE=ZV`
    1. start tuning for acceleration on an axis `TUNING_TOWER COMMAND=SET_INPUT_SHAPER PARAMETER=SHAPER_FREQ_X START=<calculated_start_val> FACTOR=<calculated_factor> BAND=5` where:
        * `calculated_start_val = crt_shaper_val * 83 / 132`
        * `factor = crt_shaper_val / 66`
        eg on x: `TUNING_TOWER COMMAND=SET_INPUT_SHAPER PARAMETER=SHAPER_FREQ_X START=20.4 FACTOR=0.4914 BAND=5`
        eg on y: `TUNING_TOWER COMMAND=SET_INPUT_SHAPER PARAMETER=SHAPER_FREQ_Y START=23.95 FACTOR=0.5772 BAND=5`
1. re-print the `ringing_tower` model
1. count the number of bands till best result (`=>` `band_number`)
1. calculate the new frequency `new_shaper_frequency = crt_shaper_val * (39 + 5 * band_number) / 66`

### Selecting algorithm

1. setup Klipper:
    1. set velocity and acceleration to a high value `SET_VELOCITY_LIMIT VELOCITY=500 ACCEL=7000 ACCEL_TO_DECEL=7000 SQUARE_CORNER_VELOCITY=5.0`
    1. disable pressure advance `SET_PRESSURE_ADVANCE ADVANCE=0`
    1. set the algorithm to MZV `SET_INPUT_SHAPER SHAPER_TYPE=MZV`
    1. start tuning for acceleration `TUNING_TOWER COMMAND=SET_VELOCITY_LIMIT PARAMETER=ACCEL START=1250 FACTOR=100 BAND=5`
1. re-print the `ringing_tower` model
1. check the printed model
1. change the algorithm to `EI` by issuing `SET_INPUT_SHAPER SHAPER_TYPE=EI`
1. re-print one again the `ringing_tower` model
1. check for best results:
    1. if the print still shows ringing
    1. the gaps are 0.15 mm so there should not be a big gap; if there is no gap, most likely it can be fixed with pressure advance setting

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

## More Info

[Klipper documentation overview](https://github.com/KevinOConnor/klipper/blob/master/docs/Overview.md)

[Pressure advance](https://www.klipper3d.org/Pressure_Advance.html)

[Slicers macros](https://github.com/KevinOConnor/klipper/blob/e7b0e7b43bbf20bf89f47444fbbfc0e10aca1ed1/docs/Slicers.md)

[Start print macro sample](https://github.com/KevinOConnor/klipper/blob/d36dbfebd17500f0af176abd88d8b258c7940e47/config/printer-lulzbot-taz6-dual-v3-2017.cfg#L216)

[Retraction Test Object](https://www.thingiverse.com/thing:4532977)

[Firmware Retraction](https://www.klipper3d.org/Config_Reference.html#firmware_retraction)

[Resonance compensation](https://www.klipper3d.org/Resonance_Compensation.html)
