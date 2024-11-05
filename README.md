# Enabling split heatbed for Elegoo Neptune 4 (Pro) in OrcaSlicer
This tutorial is meant for OrcaSlicer but will work with any slicer providing the same info adapting the variables.

The split heatbed of the Elegoo Neptune 4 (Pro) does not work with OrcaSlicer default configuration - or better to say the klipper configuration for it is already incomplete.
The root cause is that the M140 and M190 Gcodes will always heat up both heating elements at once. 

**Please consider before applying:** These changes will enable the split heatbed but will have further effects.
According to tests I performed a full heatbed heat up from 23C ambient to 60C will take about 2min 22sec, then continue to use on the inner heater about 10%/10W  and the outer about 33%/50W to keep this temperature stable (so 60W overall for keeping the temperature). 
Using only the inner heating element will take about 4min 48seconds to get from 23C ambient to 60C. This is significantly longer. The reason being that you will still heat up the same thermal mass/printbed - it is not split internally. The outer temperature bed probe will still monitor 45C created by the inner heating element only. To keep this temperature stable the inner element will further be used about 40%/40W.
During heat up the heating elements are used 100%, therefore drawing 100W on the inner heater and 150W on the outer. So the inner bed only will use 8Wh and the full bed 9,86Wh to heat up from 23C to 60C.
A one hour print as example will be 1:02:22h using 69,86Wh using both heating zones, while 1:04:48h using 48Wh using only the inner heating element (pre-print movement ignored as it is assumed to be identical).
So applying these changes will give lower power consumption but increase the time for heat up significantly. The proposed start print will try to compensate further shortcoming from this.

The change the heating behaviour changes both in the klipper macro config and the slicer start gcode have to be made. Only the klipper macros can ensure the outer element to stay off when not needed, while only the start print gcode can determine whether a print will need both heating zones or only the inner one.

## klipper macros
Open the printer.cfg and change the M104 and M190 macros as follow - please ensure the make a backup before:

    [gcode_macro M140]
    rename_existing: M99140
    gcode:
        {% if 'S' in params %}
            {% if (params.S|float) > 110 %}
            M118 The hot bed setting temperature is out of limit and has been automatically adjusted to Limit the temperature.
            {% endif %}
            {% set TARGET = (params.S|float, 110)|min %}
            {% set HEATBED_CURRENT_TARGET_TEMP = printer['heater_bed'].target %}
            {% set HEATBED1_CURRENT_TARGET_TEMP = printer['heater_generic heater_bed1'].target %}
                {% if params.S is defined and s > 0 %}
                SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET={s|int}
                ; Set temperature for the center "heater_bed" (120x120mm)
                {% if HEATBED1_CURRENT_TARGET_TEMP > 0 or HEATBED_CURRENT_TARGET_TEMP == 0 %}
                    SET_HEATER_TEMPERATURE HEATER=heater_bed1 TARGET={s|int} ; set outer bed temp
                {% endif %}
            {% else %}
                SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=0
                SET_HEATER_TEMPERATURE HEATER=heater_bed1 TARGET=0
            {% endif %} 
        {% endif %}

    [gcode_macro M190]
    rename_existing: M99190
    gcode:
        {% if 'S' in params %}
            {% if (params.S|float) > 110 %}
                M118 The hot bed setting temperature is out of limit and has been automatically adjusted to Limit the temperature.
            {% endif %}
            {% set s = (params.S|float,110)|min %}
            {% set HEATBED_CURRENT_TARGET_TEMP = printer['heater_bed'].target %}
            {% set HEATBED1_CURRENT_TARGET_TEMP = printer['heater_generic heater_bed1'].target %}
            {% if params.S is defined %}
                SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET={s|int} ; Set temperature for the center "heater_bed" (120x120mm)
                {% if HEATBED1_CURRENT_TARGET_TEMP > 0 or HEATBED_CURRENT_TARGET_TEMP == 0 %}
                SET_HEATER_TEMPERATURE HEATER=heater_bed1 TARGET={s|int} ; set outer bed temp
                TEMPERATURE_WAIT SENSOR='heater_generic heater_bed1' MINIMUM={s-1} MAXIMUM={s+10} ; wait for bed temp to stabilize
                {% endif %}
                {% if printer.heater_bed.temperature <= s-2 %}
                    TEMPERATURE_WAIT SENSOR=heater_bed MINIMUM={s-1} MAXIMUM={s+10}
                {% endif %}   
            {% else %}
                M140 S0
            {% endif %}
        {% endif %}

The new heating behaviour will be as follows:
M140/M190 will heat up both zones if the inner is currently disabled or the outer is already active. You will not be able to heat up only the outer area. Disabling will always disable both.
It will also ensure that the M190 command will wait for both elements to get to temperature, if both are used - this was not the case with stock configuration.

To heat up only the inner part yout will need to use the SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET=tempC command, which brings us to the neccessary changes for the start print gcode.

In OrcaSlicer open your printer preferences and go to "Machine G-code", "Machine start G-code". You will need to replace the initial call M140 with code checking if both zones need to be used or only the inner 120x120mm one.

    SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET={bed_temperature_initial_layer_single-5} ; set temporary bed temp
    { if (first_layer_print_min[0] < 55) or (first_layer_print_max[0] > 175) or (first_layer_print_min[1] < 55) or (first_layer_print_max[1] > 175) }
    SET_HEATER_TEMPERATURE HEATER=heater_bed1 TARGET={bed_temperature_initial_layer_single-5} ; set temporary outer bed temp
    { endif }

As the heat up will take significantly longer when only the inner heating element is used, you might want the change the entire start G-code to ensure the nozzle will not fully heat up before the bed is ready. Otherwise the filament will start oozing from the nozzle.
The following entire start G-code is a proposal to do so:

    M220 S100 ;Set the feed speed to 100%
    M221 S100 ;Set the flow rate to 100%
    G90 ; use absolute coordinates
    M83 ; extruder relative mode
    M104 S140 ; set temporary nozzle temp to prevent oozing during homing, auto bed leveling and heatbed warmup
    SET_HEATER_TEMPERATURE HEATER=heater_bed TARGET={bed_temperature_initial_layer_single-5} ; set temporary bed temp
    { if (first_layer_print_min[0] < 55) or (first_layer_print_max[0] > 175) or (first_layer_print_min[1] < 55) or (first_layer_print_max[1] > 175) }
    SET_HEATER_TEMPERATURE HEATER=heater_bed1 TARGET={bed_temperature_initial_layer_single-5} ; set temporary outer bed temp
    { endif }
    G28 ; home all axis
    BED_MESH_PROFILE LOAD=11 ; load 11x11 mesh profile
    G1 Z50 F240
    G1 X2 Y10 F3000
    M190 S{bed_temperature_initial_layer_single-5} ; wait for bed temp to stabilize at temporary level
    M104 S[nozzle_temperature_initial_layer] ; set final nozzle temp
    M190 S[bed_temperature_initial_layer_single] ; wait for bed temp to stabilize at final temp
    M109 S[nozzle_temperature_initial_layer] ; wait for nozzle temp to stabilize at final temp
    ; continue from here with your purge line

**Warnings about M141 / M191 / chamber heating:** 
- These macros are intended for chamber heating but not controlling the second heating zone as often suggested.
- The Elegoo configuration for these will also control the heatbet heating elements - do not use this feature in your slicer! It will interfere with the split heatbeat.
