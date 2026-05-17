# Emporia Vue Local Extended

This is a more advanced and much more easily configured Emporia Vue ESPHome Template (plus a small external component).
This is really made for users at either tail end of the energy monitoring bell curve, as it's both easier to set up and gives you access to a lot more data by way of some simple calculations.

### Just copy over the example_config.yaml into your esphome dashboard and all of the required external components and sensor setups are pulled in and set up automatically.
## Overview
### Original Project
The original project (which provides the entirety of the underlying esphome component) here: 
- [Emporia Vue Local GitHub](https://github.com/emporia-vue-local)
- [Project Documentation](https://emporia-vue-local.github.io/docs/tutorial/intro/)

### Simplified Configuration!
  - Automatically adjusts for 1-3 phase installations.
  - Automatically adjusts calculations based on your configured wiring layout

### More Measurements!
We mostly take the provided onboard measurements off of the Atmega (which runs Emporia OEM code, even with the base configuration)
- Phase Voltage
- Phase Angle
- Apparent Current
- Real Power

And from those we calculate extended measurements using lambdas on the ESP32
- Real Current
- Apparent Power
- Power Factor
- Real Power (refined with phase to phase voltages configured by the user)

### More ESPHome Features!
- [Uses ESPHome Subdevices](https://esphome.io/components/esphome/#sub-devices) for simplified configuration and presentation of similar sensors
- Also uses lots of lambdas

## Configuration
Including the package is important.
```yaml
  emporia_vue_extended:
    url: https://github.com/Legot/emporia-vue-local-extended
    ref: main
    files: [vue2_base.yaml] # vue2_base.yaml is for the vue2 and vue3_base.yaml is for the vue3
    refresh: 0s  # Sets esphome to update this file at each compilatoin
```

The configuration relies mostly on substitutions. All options are needed, and if you mess something up it most likely will not compile.
```yaml
substitutions:
  name: em02
  friendly_name: EM02

  phase_a_voltage_calibration: 0.02164686998
  phase_b_voltage_calibration: 0.02276118
  phase_c_voltage_calibration: 0.022
  # 0.022 is used as the default as starting point but may need adjusted to ensure accuracy
  # To calculate new calibration value use the formula <in-use calibration value> * <accurate voltage> / <reporting voltage>

  # These set the wire colors the for voltage/phase measurements
  red_wire: phase_a
  black_wire: phase_b
  blue_wire: phase_c

  sensor_update_rate: 500ms  # minimum is 240ms (limited by the Atmega)
  energy_update_rate: 60s    # should be at least 2x the sensor_update_rate

  # Configuration for each CT clamp
  ct_a:  { disable: "false", backfeed: "false", phase: a  }
  ct_b:  { disable: "false", backfeed: "false", phase: b  }
  ct_c:  { disable:  "true", backfeed: "false", phase: c  }
  ct_1:  { disable: "false", backfeed: "false", phase: ab }
  ct_2:  { disable: "false", backfeed: "false", phase: a  }
  ct_3:  { disable: "false", backfeed: "false", phase: b  }
  ct_4:  { disable: "false", backfeed: "false", phase: a  }
  ct_5:  { disable: "false", backfeed: "false", phase: b  }
  ct_6:  { disable: "false", backfeed: "false", phase: a  }
  ct_7:  { disable: "false", backfeed: "false", phase: b  }
  ct_8:  { disable: "false", backfeed: "false", phase: a  }
  ct_9:  { disable: "false", backfeed: "false", phase: b  }
  ct_10: { disable: "false", backfeed: "false", phase: a  }
  ct_11: { disable: "false", backfeed: "false", phase: b  }
  ct_12: { disable: "false", backfeed: "false", phase: a  }
  ct_13: { disable: "false", backfeed: "false", phase: b  }
  ct_14: { disable: "false", backfeed: "false", phase: ab }
  ct_15: { disable: "false", backfeed: "false", phase: ab }
  ct_16: { disable: "false", backfeed: "false", phase: ab }
```
#### Current Clamp Configuration

Each Current Clamp has an entry in the substitutions section shown as single line yaml dictionaries.
```yaml
substitutions:
  ct_1:  { disable: "false", backfeed: "false", phase: ab }
```
Alternatively this can be represented as standard multiline yaml.
```yaml
substitutions:
  ct_1:
    disable: "false"
    backfeed: "false"
    phase: ab
```
Each clamp must be configured from the following options:

| Option     | Values                    | Description                                                                                      |
|------------|---------------------------|--------------------------------------------------------------------------------------------------|
| `disable`  | `"true"`, `"false"`       | Marks the sensor as internal so all related measurement and calculations are ignored (must be in quotes due to ESPHome parsing inconsistencies)          |
| `backfeed` | `"true"`, `"false"`       | Allows or clamps negative sensor values (also must be in quotes)                                 |
| `phase`    | `a`, `b`, `c`, `ab`/`ba`, `ac`/`ca`, `bc`/`cb`, `abc`/`acb`/`bac`/`bca`/`cab`/`cba` | Sets the voltage phase or crossphase (for split-phase circuts) for the measurement, accepts permutations of the phase ordering, correct phase ordering here improves accuracy. **The first letter in the phase must be which phase the clamp is installed on.**|

### The order of the letters in your "phase:" substitution matter!
- The first letter in "phase:" should be the phase that the current clamp is installed on.
- The second letter is only used for crossphase and 3-phase circuits, and should match the appropriate phase/leg.
- The third letter is only used for 3-phase circuits and should match the appropriate phase.

*This is the most complicated part of the configuration, it's important to get this right!*


## Some Math used for the sensors
#### Real Power
```math
 P_{\text{actual}} = \left(\frac{P_{\text{measured}}}{V_{\text{phase}}}\right)\times V_{\text{crossphase}}
```
#### Real Current
```math
I_{\text{real}} = \frac{P_{\text{actual}}}{V_{\text{phase}}}
```
#### Apparent Current
- Interesting note is that we don't use the Phase Angle in any calculations other than apparent current, because the Atmega does that already to calculate the
Real Power. We don't even have to use it for the Apparent Current, but it is slightly more accurate than root sum of squares
```math
I_{\text{apparent}} = \frac{P}{V \cdot \cos(\theta)}
```
#### Apparent Power
```math
\text{VA} = V_{\text{phase}} * I_{\text{apparent}}
```
#### Power Factor
```math
\text{PF} = \frac{W}{\text{VA}}
```
## Notes
The bottleneck on update_rate is always going to be your HomeAssistant host and database preformance. I've been running two of these devices doing all of these calculations at a 500ms update rate for over 2 years now with no issues.
- My Homeassistant server is a Proxmox VM running at <20% CPU utilization on 4 Intel i5 cores. YMMV

The default homeassistant database isn't designed for time series data (despite heavy optimization), but the recorder is. I highly recommend setting up a real Time Searies Database (either as a homeassistant add-on or standalone/VM)
- I use the InfluxDB integration to send data to a VictoriaMetrics Container and visualize using Grafana
- InfluxDB v2 also works well, but I moved away due to liscensing concerns with the free version.

NTP time isn't neccessary, because of the Homeassistant API, but in my experience the NTP timesource is far more reliable
- for some reason I was seeing the HA API time (which is derived from NTP) drift by up to 5 seconds per day. I think the API only syncs time on boot of the ESP
- The Chrony Homeassistant Add-on will let you set up an NTP server on your HA host, which allows the ESP to sync every 10-15 minutes.

Depending on what you're doing with your Daily Energy numbers, it may be useful to go as fast as a 1s update rate.
