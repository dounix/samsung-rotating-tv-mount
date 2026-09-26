## Samsung rotating TV mount control with home assistant via esphome

Tested with model VG-ARAB43WMT(400x300 VESA), but assume VG-ARAB22WMTZA(200x200 VESA mount) is the same
Control with esp32, or anything i2c

### Hardware 

Has an STM32410CB, and a BT module connected via FFC cable.
The BT seems to do some Samsung pairing/commands that were not investigated.

The STM32 has no readout protection, dumpable with a cheap st-link clone.
That dump and Ghidra should be enough for a clanker to figure out the i2c commands, but it wasn't.

Ripped and replaced the bluetooth module that connects to the i2c bus, but anything that can talk i2c will work.

Using the original firmware, it's only a curiosity that the stepper motor is a HEM-60S1401, this is driven by a DRV8886AT stepper driver  

## Summary
The mount's STM32 takes rotation commands over I2C from supported TVs via the Samsung Bluetooth module.  
I found a deal on this mount, and wanted to rotate a TV that is unsupported via homeassistant.
Didn't see these commands documented anywhere, hope this save someone a bit of time  

## Power budget

Not considered, no brownouts, but the ESP32 does use more power than any BT module.  
Risk weighed against being a paperweight.

## Physics still apply

Under the weight limit
Central VESA mount/center of gravity
Design that doesn't self destruct in portrait mode(cooling, etc).


| | |
|---|---|
| **Bus** | I²C1, 400 kHz, slave address **0x41** |
| **easy SDA/SCL access** | **CN302 pins 2/3** — an unpopulated header next to the opto encoder port |
| **Power** | Pick up vcc/gnd where ever you get your microcontroller supplies |
| **What works** | slow rotation(35s) this is homing speed, ramped "fast" rotation(10s) |

## Bus commands

All values **hex**. Message format is `[command][parameter]`, two bytes.

### parameters

Opcode **`0x11`** against 7-bit address **`0x41`** (`0x82` to write). The values below are
**operands** — message byte 1 — not opcodes.

| Param | Move | 90° |
|---|---|---|
| `0x01` | **portrait**, speed ramped | **~10 s** |
| `0x02` | **landscape**, speed ramped | **~10 s** |
| `0x05` | **portrait**, slow(homing speed) | ~35 s |
| `0x06` | **landscape**, slow(homing speed) | ~35 s |



## ESPHome configuration

```yaml
esphome:
  name: rotato
  friendly_name: rotato

esp32:
  variant: esp32

logger:

api:
  encryption:
    key: "REPLACE_WITH_YOUR_OWN_KEY"      # generate: openssl rand -base64 32

ota:
  - platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  ap:
    ssid: rotato Fallback Hotspot
    password: "REPLACE_WITH_YOUR_OWN_PASSWORD"

captive_portal:

i2c:
  id: rotato_i2c
  sda: GPIO16
  scl: GPIO17
  frequency: 400kHz
  timeout: 13ms          # clock-stretch limit; 13ms is the esp-idf maximum
  scan: true             # logs what answered at boot — expect 0x41

script:
  - id: send_command
    parameters:
      command: int
      parameter: int
    mode: queued
    then:
      - lambda: |-
          const uint8_t message[2] = {(uint8_t) command, (uint8_t) parameter};
          auto err = id(rotato_i2c)->write(0x41, message, sizeof(message));
          if (err != i2c::ERROR_OK) {
            ESP_LOGE("stand", "command 0x%02X param 0x%02X failed, i2c error %d",
                     (unsigned) command, (unsigned) parameter, (int) err);
          } else {
            ESP_LOGI("stand", "command 0x%02X param 0x%02X sent",
                     (unsigned) command, (unsigned) parameter);
          }

button:
  - platform: template
    name: "Stand Portrait"
    icon: mdi:phone-rotate-portrait
    on_press:
      - script.execute: {id: send_command, command: 0x11, parameter: 0x01}

  - platform: template
    name: "Stand Landscape"
    icon: mdi:phone-rotate-landscape
    on_press:
      - script.execute: {id: send_command, command: 0x11, parameter: 0x02}

  - platform: template
    name: "Stand Slow Portrait"
    icon: mdi:phone-rotate-portrait
    on_press:
      - script.execute: {id: send_command, command: 0x11, parameter: 0x05}

  - platform: template
    name: "Stand Slow Landscape"
    icon: mdi:phone-rotate-landscape
    on_press:
      - script.execute: {id: send_command, command: 0x11, parameter: 0x06}

  # Always sends quiet, even if the switch already reads off.
  - platform: template
    name: "Stand Force Quiet"
    icon: mdi:volume-off
    on_press:
      - script.execute: {id: send_command, command: 0x10, parameter: 0}


```
