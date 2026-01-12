# ESPHome-LVGL-running-on-a-Guition-JC4827W543
### A proof of concept project to control a ESP32-C6 relay module from HA and/or an ESP32 based 4.3in Display Module (January 2026)  

![cyd_relay](https://github.com/user-attachments/assets/f0d13519-b735-44eb-8d1f-94dbe338054e)

#### Intro  

This project explores the integration of Home Assistant (HA) with a cheap ESP32-C6 relay module and the Guition JC4827W543C 4.3" LCD Display module.  

My aim was to learn how to use ESPHome LVGL to run YAML on the display module to control the relay module both connected to my test Home Assistant
setup on my home network. My grip of ESPHome is not extensive to be fair making me reliant on searching out example code that I can use and
learn from. Furthermore - a quick examination of the ESPHome LVGL documentation https://esphome.io/components/lvgl/ made me realise that learning LVGL from 
scratch was going to be steep learning curve especially as example code seemed to be sparse!  

A quick search revealed that the Gution Display module is supported https://devices.esphome.io/devices/guition-jc4827543c/ and I was able
to copy/paste a short YAML into my ESPHome environment that runs on my development docker computer. This at least worked and formed the base YAML for my proof of concept project.    

The creation of the YAML for the ESP32-C6 Relay module is covered in a separate project  https://github.com/roadsnail/ESP32-C6-Relay  

This project builds upon that allowing me to control it from HA or from the display module

###  Display YAML
```
esphome:
  name: "cyd"
  platformio_options:
    board_build.flash_mode: dio
  on_boot:
    priority: -100
    then:
      - component.update: my_display
      - lvgl.slider.update:
          id: delay_slider
          value: !lambda "return id(relay_turn_off_delay_local);"


esp32:
  board: esp32-s3-devkitc-1
  variant: esp32s3
  flash_size: 4MB
  framework:
    type: esp-idf

psram:
  mode: octal
  speed: 80MHz

logger:

web_server:
  port: 80

ota:
  - platform: esphome

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password


globals:
  - id: relay_turn_off_delay_local
    type: int
    restore_value: true
    initial_value: '0'

  - id: countdown_active
    type: bool
    restore_value: no
    initial_value: 'false'

  - id: countdown_remaining
    type: int
    restore_value: no
    initial_value: '0'


interval:
  - interval: 1s
    then:
      - if:
          condition:
            lambda: 'return id(countdown_active);'
          then:
            - lambda: |-
                if (id(countdown_remaining) > 0) {
                  id(countdown_remaining)--;
                }
            - lvgl.label.update:
                id: countdown_label
                text: !lambda |-
                  static char buf[32];
                  snprintf(buf, sizeof(buf), "Turning off in: %d s", id(countdown_remaining));
                  return buf;

# -------------------------------
# I2C + Touch
# -------------------------------
i2c:
  - id: bus_a
    sda: GPIO8
    scl: GPIO4
    scan: true

touchscreen:
  - platform: gt911
    id: my_touchscreen
    i2c_id: bus_a
    interrupt_pin:
      number: 3
      ignore_strapping_warning: true
    reset_pin: GPIO38
    on_touch:
      then:
        - script.execute: restart_backlight_timer
    on_release:
      then:
        - if:
            condition: lvgl.is_paused
            then:
              - lvgl.resume
              - lvgl.widget.redraw
              - logger.log: "Antiburn stopped by touch"

# -------------------------------
# Display
# -------------------------------
spi:
  id: quad_spi
  type: quad
  clk_pin: GPIO47
  data_pins: [21,48,40,39]

display:
  - platform: qspi_dbi
    id: my_display
    update_interval: never
    auto_clear_enabled: false
    model: CUSTOM
    data_rate: 20MHz
    dimensions:
      width: 480
      height: 272
    cs_pin:
      number: 45
      ignore_strapping_warning: true
    invert_colors: true
    rotation: 180
    init_sequence:
      - [0xff,0xa5]
      - [0x36,0xc0]
      - [0x3A,0x01]
      - [0x41,0x03]
      - [0x44,0x15]
      - [0x45,0x15]
      - [0x7d,0x03]
      - [0xc1,0xbb]
      - [0xc2,0x05]
      - [0xc3,0x10]
      - [0xc6,0x3e]
      - [0xc7,0x25]
      - [0xc8,0x11]
      - [0x7a,0x5f]
      - [0x6f,0x44]
      - [0x78,0x70]
      - [0xc9,0x00]
      - [0x67,0x21]
      - [0x51,0x0a]
      - [0x52,0x76]
      - [0x53,0x0a]
      - [0x54,0x76]
      - [0x46,0x0a]
      - [0x47,0x2a]
      - [0x48,0x0a]
      - [0x49,0x1a]
      - [0x56,0x43]
      - [0x57,0x42]
      - [0x58,0x3c]
      - [0x59,0x64]
      - [0x5a,0x41]
      - [0x5b,0x3c]
      - [0x5c,0x02]
      - [0x5d,0x3c]
      - [0x5e,0x1f]
      - [0x60,0x80]
      - [0x61,0x3f]
      - [0x62,0x21]
      - [0x63,0x07]
      - [0x64,0xe0]
      - [0x65,0x02]
      - [0xca,0x20]
      - [0xcb,0x52]
      - [0xcc,0x10]
      - [0xcd,0x42]
      - [0xd0,0x20]
      - [0xd1,0x52]
      - [0xd2,0x10]
      - [0xd3,0x42]
      - [0xd4,0x0a]
      - [0xd5,0x32]
      - [0x80,0x00]
      - [0xa0,0x00]
      - [0x81,0x07]
      - [0xa1,0x06]
      - [0x82,0x02]
      - [0xa2,0x01]
      - [0x86,0x11]
      - [0xa6,0x10]
      - [0x87,0x27]
      - [0xa7,0x27]
      - [0x83,0x37]
      - [0xa3,0x37]
      - [0x84,0x35]
      - [0xa4,0x35]
      - [0x85,0x3f]
      - [0xa5,0x3f]
      - [0x88,0x0b]
      - [0xa8,0x0b]
      - [0x89,0x14]
      - [0xa9,0x14]
      - [0x8a,0x1a]
      - [0xaa,0x1a]
      - [0x8b,0x0a]
      - [0xab,0x0a]
      - [0x8c,0x14]
      - [0xac,0x08]
      - [0x8d,0x17]
      - [0xad,0x07]
      - [0x8e,0x16]
      - [0xae,0x06]
      - [0x8f,0x1B]
      - [0xaf,0x07]
      - [0x90,0x04]
      - [0xb0,0x04]
      - [0x91,0x0a]
      - [0xb1,0x0a]
      - [0x92,0x16]
      - [0xb2,0x15]
      - [0xff,0x00]
      - [0x11,0x00]
      - [0x29,0x00]

font:
  - file: "gfonts://Roboto"
    id: roboto_20
    size: 24

# -------------------------------
# Backlight
# -------------------------------
output:
  - platform: ledc
    pin: GPIO1
    id: gpio_backlight_pwm
    frequency: 1000Hz

light:
  - platform: monochromatic
    output: gpio_backlight_pwm
    id: display_backlight
    name: Display Backlight
    restore_mode: ALWAYS_OFF


# -------------------------------
script:
  - id: restart_backlight_timer
    mode: restart
    then:
      - light.turn_on: display_backlight
      - delay: 60s
      - light.turn_off: display_backlight

  - id: disable_relay_button
    then:
      - lvgl.widget.disable:
          id: relay_button
      - lvgl.widget.disable:
          id: delay_slider

  - id: enable_relay_button
    then:
      - lvgl.widget.enable:
          id: relay_button
      - lvgl.widget.enable:
          id: delay_slider



switch:
# ------------------------------------------------------------------------------------------------------------------
# Relay module control (proxy → HA) - local template switch used to control the relay module relay_control (virtual switch)
# ------------------------------------------------------------------------------------------------------------------
  - platform: template
    id: relay_proxy
    optimistic: true
    turn_on_action:
      - homeassistant.service:
          service: switch.turn_on
          data:
            entity_id: switch.relay_relay_control
    turn_off_action:
      - homeassistant.service:
          service: switch.turn_off
          data:
            entity_id: switch.relay_relay_control

  - platform: template
    id: switch_antiburn
    name: Antiburn
    optimistic: true
    icon: mdi:television-shimmer
    turn_on_action:
      - logger.log: "Starting Antiburn"
      - lvgl.pause:
          show_snow: true
    turn_off_action:
      - logger.log: "Stopping Antiburn"
      - lvgl.resume
      - lvgl.widget.redraw



number:
  - platform: homeassistant
    id: ha_relay_delay
    entity_id: number.relay_relay_turn_off_delay
    on_value:
      then:
        - lambda: |-
            if (x < 0 || x > 300) {
              // Ignore any transient / invalid HA values
              return;
            }
            id(relay_turn_off_delay_local) = (int)x;
        - lvgl.slider.update:
            id: delay_slider
            value: !lambda 'return (int)x;'
        - lvgl.label.update:
            id: delay_label
            text: !lambda |-
              char buf[40];
              sprintf(buf, "Off Delay: %d s", (int)x);
              return std::string(buf);



# -------------------------------
# HA state imports
# -------------------------------
binary_sensor:
  - platform: homeassistant
    id: ha_physical_relay
    entity_id: switch.relay_main_relay_physical
    on_state:
      then:
        - script.execute: restart_backlight_timer
        - if:
            condition:
              binary_sensor.is_on: ha_physical_relay
            then:
              - lvgl.label.update:
                  id: relay_state_label
                  text: "Physical Relay State: ON"
            else:
              - script.execute: enable_relay_button
              - lambda: |-
                  id(countdown_active) = false;
              - lvgl.label.update:
                  id: countdown_label
                  hidden: true
              - lvgl.label.update:
                  id: relay_state_label
                  text: "Physical Relay State: OFF"


  - platform: homeassistant
    id: ha_virtual_relay
    entity_id: switch.relay_relay_control
    on_state:
      then:
        - script.execute: restart_backlight_timer
        - if:
            condition:
              binary_sensor.is_off: ha_virtual_relay
            then:
              # Virtual relay turned OFF from HA → start countdown UI
              - lambda: |-
                  id(countdown_active) = true;
                  id(countdown_remaining) = id(relay_turn_off_delay_local);
              - lvgl.label.update:
                  id: countdown_label
                  hidden: false
                  text: !lambda |-
                    static char buf[32];
                    snprintf(buf, sizeof(buf), "Turning off in: %d s", id(countdown_remaining));
                    return buf;
              - script.execute: disable_relay_button



# -------------------------------
# LVGL UI
# -------------------------------
lvgl:
  pages:
    - id: main_page
      scrollable: false
      widgets:
#-----------------------------------------------------------
# button widget - Contains text 'Toggle Relay' - used to switch template switch relay_proxy
# may also start the 'countdown' (until physical relay on module is switched OFF)
# if the physical relay state is reported to be ON (from HA)
#------------------------------------------------------------
        - button:
            id: relay_button
            x: 60
            y: 60
            width: 140
            height: 50
            text: "Toggle Relay"
            on_click:
              then:
                - script.execute: restart_backlight_timer   #always restart the backlight (timeout) timer if clicked..screen comes ON
                - if:
                    condition:
                      binary_sensor.is_on: ha_physical_relay
                    then:
                      - lambda: |-
                          id(countdown_active) = true;
                          id(countdown_remaining) = id(relay_turn_off_delay_local);
                      - lvgl.label.update:
                          id: countdown_label
                          hidden: false
                          text: !lambda |-
                            static char buf[32];
                            snprintf(buf, sizeof(buf), "Turning off in: %d s", id(countdown_remaining));
                            return buf;
                      - script.execute: disable_relay_button
                      - switch.turn_off: relay_proxy
                    else:
                      - switch.turn_on: relay_proxy
        - label:
            id: relay_state_label
            x: 220
            y: 70
            text: "Physical Relay State: OFF"


        - label:
            id: countdown_label
            text: ""
            x: 220
            y: 110
            hidden: true

#
# define a slider widget
#
        - slider:
            id: delay_slider
            x: 60
            y: 140
            width: 300
            min_value: 0
            max_value: 300
            on_value:
              then:
                - lambda: |-
                    id(relay_turn_off_delay_local) = (int)x;
                - homeassistant.service:
                    service: number.set_value
                    data:
                      entity_id: number.relay_relay_turn_off_delay
                      value: !lambda "return x;"
        - label:
            id: delay_label
            x: 60
            y: 180
            text: "Off Delay: 0 s"


# -------------------------------
# HA reconnect sync various things from HA
# -------------------------------
api:
  on_client_connected:
    then:
      - script.execute: restart_backlight_timer
      - lvgl.slider.update:
          id: delay_slider
          value: !lambda 'return (int)id(ha_relay_delay).state;'
      - lvgl.label.update:
          id: delay_label
          text: !lambda |-
            char buf[40];
            sprintf(buf, "Off Delay: %d s", (int)id(ha_relay_delay).state);
            return std::string(buf);
      - if:
          condition:
            binary_sensor.is_on: ha_physical_relay
          then:
            - lvgl.label.update:
                id: relay_state_label
                text: "Physical Relay State: ON"
          else:
            - lvgl.label.update:
                id: relay_state_label
                text: "Physical Relay State: OFF"
```
