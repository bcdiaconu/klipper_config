# klipper configuration

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
