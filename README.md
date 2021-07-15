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

## Pressure (Linear) advance

1. download the file from [here](https://www.klipper3d.org/prints/square_tower.stl "stl file to download")
1. setup the printer and generate g-code:
  1. use high printing speeds (eg 100 mm/s)
  1. set 0% infill
  1. layer height 75% of nozzle diameter (0.3 for 0.4 nozzle)
1. send the follwing command to printer `SET_VELOCITY_LIMIT SQUARE_CORNER_VELOCITY=1 ACCEL=500`
1. and set the pressure_advance start and increment factor by sending either of the following commands:
  1. for direct drive `TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.005`
  1. for bowden `TUNING_TOWER COMMAND=SET_PRESSURE_ADVANCE PARAMETER=ADVANCE START=0 FACTOR=.020`
1. with calipers measure the height from bottom till the layer that seems to produce best results (`measured_height`)
1. calculate the pressure advance by applying formula `pressure_advance = <START> + <measured_height> * <FACTOR>`
1. under `[extruder]` section, add the resulted value eg: `pressure_advance: 0.0614`
1. issue a `RESTART` command to restart the firmware

## More Info

<https://github.com/KevinOConnor/klipper/blob/master/docs/Overview.md>

<https://www.klipper3d.org/Pressure_Advance.html>

<https://github.com/KevinOConnor/klipper/blob/e7b0e7b43bbf20bf89f47444fbbfc0e10aca1ed1/docs/Slicers.md>

<https://github.com/KevinOConnor/klipper/blob/d36dbfebd17500f0af176abd88d8b258c7940e47/config/printer-lulzbot-taz6-dual-v3-2017.cfg#L216>