Projects Overview

Smart Water Flow Meter
This YAML configuration is for a water flow meter that tracks tank capacity, inflow, and outflow. The system automatically calculates and adjusts the total water percentage in your tank based on real-time flow data. Features include:

Tank capacity adjustment: Set the total tank volume via a slider in Home Assistant.
Flow control: Track water in and out, with adjustments also available via sliders.
Automatic calculation: Updates the tank percentage dynamically based on flows.

Prerequisites
ESPHome installed on your local machine or accessible via Home Assistant.
Compatible ESP devices:
ESP32-based board for the water flow meter.
water flow sensor.
Basic knowledge of YAML configuration and Home Assistant integration.

Setup Instructions
Copy the desired YAML configuration(s) to your ESPHome directory.
Modify the YAML file to match your hardware setup:
For the Water Flow Meter, configure GPIO pins, tank capacity, and initial settings.
Upload the configuration to your ESP device using ESPHome.

# esp-home-water-flow-sensor
Water Tank Level

esphome:
  name: YOUR ESP NAME
  friendly_name: water flow meter

esp8266:
  board: esp01_1m

# Enable logging
logger:

web_server:
  port: 80 
  version: 3

# Enable Home Assistant API
api:
  encryption:
    key: ""

ota:
  - platform: esphome
    password: ""

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: ""
    password: ""

captive_portal:

# Global Variables for Restoration
globals:
  - id: stored_water_in
    type: float
    restore_value: true
    initial_value: "0.0"
  - id: stored_water_out
    type: float
    restore_value: true
    initial_value: "0.0"
  - id: tank_capacity
    type: float
    restore_value: true
    initial_value: "130"  # Default tank capacity in liters (ADJUST TO YOUR MAX TANK SIZE)

# Restoration Values and Settings
number:
  - platform: template
    name: "Stored Water In"
    id: restore_water_in
    optimistic: true
    min_value: 0
    max_value: 130
    step: 0.1
    initial_value: 0
    unit_of_measurement: "L"
    on_value:
      then:
        - lambda: |-
            id(stored_water_in) = x;

  - platform: template
    name: "Stored Water Out"
    id: restore_water_out
    optimistic: true
    min_value: 0
    max_value: 130
    step: 0.1
    initial_value: 0
    unit_of_measurement: "L"
    on_value:
      then:
        - lambda: |-
            id(stored_water_out) = x;

  - platform: template
    name: "Tank Capacity"
    id: tank_capacity_setting
    optimistic: true
    min_value: 1
    max_value: 130. # ADJUST TO YOUR MAX TANK SIZE
    step: 0.1
    initial_value: 0.0  # Default capacity
    unit_of_measurement: "L"
    on_value:
      then:
        - lambda: |-
            id(tank_capacity) = x;

# Water Flow Sensors
sensor:
  # Input Water Sensor
  - platform: pulse_meter
    pin: GPIO3    # CHANGE TO YOUR ESP PINOUT
    name: "Water Flow In"
    id: sensor_water_in
    internal_filter: 100ms
    timeout: 30s
    unit_of_measurement: "L/min"
    accuracy_decimals: 1
    filters:
      - multiply: 0.010  # Convert pulses/min to L/min (0.010L/pulse)
    total:
      name: "Total Water In"
      id: total_water_in
      unit_of_measurement: "L"
      accuracy_decimals: 1
      state_class: total_increasing
      filters:
        - multiply: 0.010  # 0.010L per pulse

  # Output Water Sensor
  - platform: pulse_meter
    pin: GPIO1   # CHANGE TO YOUR ESP PINOUT
    name: "Water Flow Out"
    id: sensor_water_out
    internal_filter: 100ms
    timeout: 30s
    unit_of_measurement: "L/min"
    accuracy_decimals: 1
    filters:
      - multiply: 0.10  # Convert pulses/min to L/min (0.010L/pulse)
    total:
      name: "Total Water Out"
      id: total_water_out
      unit_of_measurement: "L"
      accuracy_decimals: 1
      state_class: total_increasing
      filters:
        - multiply: 0.010  # 0.010L per pulse

  # Calculated Sensors
  - platform: template
    name: "Remaining Water"
    id: remaining_water
    unit_of_measurement: "L"
    device_class: water
    state_class: measurement
    accuracy_decimals: 1
    lambda: |-
      float water_in = id(total_water_in).state + id(stored_water_in);
      float water_out = id(total_water_out).state + id(stored_water_out);
      float remaining = water_in - water_out;
      return (remaining < 0) ? 0 : remaining;
    update_interval: 2s

  - platform: template
    name: "Tank Percentage"
    id: tank_percentage
    unit_of_measurement: "%"
    device_class: water
    state_class: measurement
    accuracy_decimals: 1
    lambda: |-
      float water_in = id(total_water_in).state + id(stored_water_in);
      float water_out = id(total_water_out).state + id(stored_water_out);
      float remaining = water_in - water_out;
      if (remaining < 0) remaining = 0;
      float percentage = (remaining / id(tank_capacity)) * 100.0;
      return (percentage > 100.0) ? 100.0 : percentage;
    update_interval: 2s

  - platform: template
    name: "Total Water Usage"
    unit_of_measurement: "L"
    device_class: water
    state_class: total_increasing
    accuracy_decimals: 1
    lambda: |-
      return id(total_water_out).state + id(stored_water_out);
    update_interval: 2s

# Periodic Value Storage
interval:
  - interval: 5min
    then:
      - lambda: |-
          id(restore_water_in).publish_state(id(total_water_in).state + id(stored_water_in));
          id(restore_water_out).publish_state(id(total_water_out).state + id(stored_water_out));

# Reset Buttons
button:
  - platform: template
    name: "Reset Total Water In"
    on_press:
      - pulse_meter.set_total_pulses:
          id: sensor_water_in
          value: 0
      - lambda: |-
          id(stored_water_in) = 0;
          id(restore_water_in).publish_state(0);
      - logger.log: 
          format: "Total Water In reset to 0L"
          level: INFO

  - platform: template
    name: "Reset Total Water Out"
    on_press:
      - pulse_meter.set_total_pulses:
          id: sensor_water_out
          value: 0
      - lambda: |-
          id(stored_water_out) = 0;
          id(restore_water_out).publish_state(0);
      - logger.log:
          format: "Total Water Out reset to 0L"
          level: INFO

  - platform: template
    name: "Reset All Values"
    on_press:
      - pulse_meter.set_total_pulses:
          id: sensor_water_in
          value: 0
      - pulse_meter.set_total_pulses:
          id: sensor_water_out
          value: 0
      - lambda: |-
          id(stored_water_in) = 0;
          id(stored_water_out) = 0;
          id(restore_water_in).publish_state(0);
          id(restore_water_out).publish_state(0);
      - logger.log:
          format: "All water values reset to 0L"
          level: INFO

# Water Level Monitoring
binary_sensor:
  - platform: template
    name: "Low Water Level Alert"
    device_class: problem
    lambda: |-
      return id(tank_percentage).state < 10.0;
    filters:
      - delayed_on: 5s
      - delayed_off: 5s
    on_state:
      then:
        if:
          condition:
            lambda: 'return id(tank_percentage).state < 10.0;'
          then:
            - logger.log:
                format: "Warning: Water level is below 10%% (%.1fL remaining)!"
                args: ['id(remaining_water).state']
                level: WARN
