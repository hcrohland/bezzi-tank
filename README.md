# Beschreibung
Bezzera (Duo und andere) Espresso Maschinen stoppen den Bezug abrupt, wenn der Tank leer wird. Sie haben auch keinen Anzeige für den Wasserstand im Tank.

Ich habe einen einfachen Sensor aus einem [M5Atom Lite](https://shop.m5stack.com/products/atom-lite-esp32-development-kit) und dem dazu gehörigen [Ultraschall-Abstandssensor RCWL-9620](https://shop.m5stack.com/products/ultrasonic-distance-unit-i2c-rcwl-9620) programmiert, der bei niedrigem Stand eine LED anmacht. Damit wird man früh genug daran erinnert, das Wasser nachzufüllen.
Der Sensor kann in Home Assistant integriert werden. Der Button des M5Atom wird auch exportiert und ich nutze ihn, um die Steckdose der Bezzi zu schalten.

<img alt="Niedriger Wasserstand" src="https://github.com/user-attachments/assets/bb8827ff-993e-43df-bb2d-2bc531819f50" width="300"> 
<img alt="Sensor unter Tankdeckel" src="https://github.com/user-attachments/assets/2b51076e-bb9e-4e2d-8aba-b29bfde32b14" width="300"> 



# Programmiert ist es mit [ESPHome](https://esphome.io/):

```yaml
esphome:
  friendly_name: Bezzi
  name: "bezzi"
  # Workaround for Button (GPIO39) randomly triggering
  # Increases power usage slightly (~1mA)
  # https://docs.espressif.com/projects/esp-idf/en/v4.4/esp32/api-reference/peripherals/adc.html#_CPPv412adc1_get_raw14adc1_channel_t
  includes:
    - m5stack-atom-lite.h
  on_boot:
    then:
      - lambda: "adc_power_acquire();"

esp32:
  board: m5stack-core-esp32
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: !secret api_key

ota:
  - platform: esphome
    password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: "Bezzi Config Hotspot"
    password: "Bezzi"

captive_portal:

web_server:
  version: 3
  port: 80
  local: True
  ota: False
  log: False

## Device-specific

light:
  - platform: neopixelbus
    type: GRB
    variant: SK6812
    pin: 27
    num_leds: 1
    id: status_led
    internal: true

binary_sensor:
  - platform: gpio
    pin:
      number: 39
      inverted: true
    name: Button
    icon: "mdi:gesture-tap-button"
    web_server:
      sorting_weight: 50
  - platform: template
    name: "Kaffeewasser nachfüllen"
    id: nachfuellen
    icon: "mdi:kettle-alert-outline"
    web_server:
      sorting_weight: 10
    on_state:
      then:
        - if: 
            condition: 
              binary_sensor.is_on: nachfuellen
            then:
              - light.turn_on: 
                  id: status_led
                  brightness: 100%
                  red: 100%
                  green: 0%
                  blue: 0%
        - if:
            condition:
              binary_sensor.is_off: nachfuellen
            then:
              - light.turn_off: status_led
  - platform: template
    name: "Light"
    icon: "mdi:alarm-light"
    web_server:
      sorting_weight: 40
    condition:
      light.is_on:
        id: status_led


external_components:
  - source:
      type: git
      url: https://github.com/chill-Division/M5Stack-ESPHome/
      ref: main
    components: sonic_i2c

i2c:
  sda: 26
  scl: 32
  scan: true
  id: bus_1

sensor:
  - platform: sonic_i2c
    i2c_id: bus_1
    address: 0x57
    name: "Wasserstand"
    id: ultrasonic1
    unit_of_measurement: mm
    update_interval: 5s
    accuracy_decimals: 0
    icon: mdi:car-coolant-level
    web_server:
      sorting_weight: 1
    filters:
      - round: 0
      - min:
          window_size: 6
          send_every: 1
      - lambda: 'return id(max_distance).state - x;'
    on_value:
        then:
        - binary_sensor.template.publish:
            id: nachfuellen
            state: !lambda 'if (x < 0.0 ) return true ; if (x > 10.0) return false; return {};'

number:
  - platform: template
    name: "Kaffeewasser max distance"
    icon: "mdi:arrow-expand-vertical"
    id: max_distance
    unit_of_measurement: mm
    optimistic: true
    min_value: 20
    max_value: 200
    initial_value: 130
    step: 1
    restore_value: true
    web_server:
      sorting_weight: 2
```

# Konfiguration
Die Tiefe des Tanks kann es entweder via Home Assistant oder über das eingebaute Webinterface konfiguriert werden:
<img width=300 alt="Home Assistant Interface" src="https://github.com/user-attachments/assets/37e3ec14-b189-455e-99a7-bf0b4970fa03">
<img width=600 alt="Webinterface" src="https://github.com/user-attachments/assets/a5f59165-630b-496a-85e1-0ba85a8c4e2d">
