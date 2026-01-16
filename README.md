# ESPHome-LVGL-running-on-a-Guition-JC4827W543
### A proof of concept project to control a ESP32-C6 relay module from HA and/or an ESP32 based 4.3in Display Module (January 2026 - ESPHome LVGL version (2025.12.5) used)  

![cyd_relay](https://github.com/user-attachments/assets/f0d13519-b735-44eb-8d1f-94dbe338054e)

#### Intro  

This project explores the integration of Home Assistant (HA) with a cheap ESP32-C6 relay module and the Guition JC4827W543C 4.3" LCD Display module.  

My aim was to learn how to use ESPHome LVGL to run YAML on the display module to create a UI to control the ESP32 relay module both connected to my test Home Assistant setup on my home network. My knowledge of ESPHome is not extensive to be fair making me reliant on searching out example code that I can use and learn from. Furthermore - a quick examination of the ESPHome LVGL documentation https://esphome.io/components/lvgl/ made me realise that learning LVGL from scratch was going to be a steep learning curve especially as example code on the net seemed to be sparse!  

A quick search revealed that the Guition Display module IS supported in ESPHome LVGL https://devices.esphome.io/devices/guition-jc4827543c/ and I was able to copy/paste the short YAML supplied into my ESPHome environment that runs on my development docker computer. This at least compiled and uploaded to my module using ESPHome (running on my Linux docker system. This YAML became the base YAML for my proof of concept project presented here.    

The creation of the YAML for the ESP32-C6 Relay module is covered in a separate project  https://github.com/roadsnail/ESP32-C6-Relay  

Please check out that repo if you wish to follow along with this project as the UI created here may be used to control the physical relay on that module.




#### Initial Goals and design evolution with assistance from ChatGPT

The software evolved in stages with some input from ChatGPT but with a lot of sanity checking its proposed '100% solutions' from me! For some bizarre reason ChatGPT would offer solutions for older versions of ESPHome which is understandable. However, this happened many times even though I had provided the version of ESPHome LVGL version (2025.12.5)!  

This was my first time of using ChatGPT for code creation as I knew I would be encountering a steep learning curve and I was interested to see just how good it is. My conclusion is it creates mostly usable code VERY quickly and can really help with useful pointers for ESPHome LVGL 'newbies' like me, however, it cannot be trusted absolutely and it does seem to get stuck in what I can only describe as endless loops when encountering an issue. On at least one occasion I had to study the documentation then provide a link to that documentation as ChatGPT was obviously stuck for a solution. So humans still have a vital role to play (for now)!

### Goal  
To create an ESPHome-based LVGL touchscreen interface (UI) running on a Guition JC4827W543 display to switch a physical relay on a ESP32-C6 relay module controlled via Home Assistant.

Key design features: - The UI will use various LVGL widgets to carry out relay switching while reporting delay and physical relay status and integrating cleanly with Home Assistant.

* HA ↔ LVGL bi-directional sync (UI BUTTON presses, delay timer SLIDER changes synchronised with HA entities and vice-versa)

* Physical relay–driven state (not optimistic). The UI and HA will display the actual state of the physical relay  

* Delayed turn-off with countdown UI. When switching the physical relay OFF from the UI or HA, the relay module YAML switches the relay OFF after a programmable delay time (0 to 300s). That countdown will be displayed on the UI display while disabling the delay SLIDER and 'Toggle Relay' button. These controls will be re-enabled when the relay delay timer expires

* Antiburn + backlight timeout implimented on the display. The default is 60s before the backlight is switched OFF. Any touch or update from HA turns the backlight ON. This is best practice for LCD screens improving longevity

To create the YAML for the UI using the latest (at the time of writing this) ESPHome version (2025.12.5)  

#### State diagram showing UI (button) and/or HA Relay Control Actions

<img width="1652" height="328" alt="cyd_state" src="https://github.com/user-attachments/assets/14ee9f26-4bdc-412b-aefc-38407101f37c" />


###  Stage 1 - Create a UI Relay Switch

* Create a LVGL SWITCH widget which drives a template switch (id: relay_proxy). The template switch in turn drives the HA virtual relay switch which controls the relay module virtual relay switch.  

  The SWITCH widget was changed to a BUTTON (id: relay_button) widget as there was an issue updating its status from HA. 
  
###  Stage 2 - Create a label widget 

* To indicate a BUTTON press, a label WIDGET ( id: relay_state_label) was used to reflect the physical relay state (ON or OFF) via a binary sensor (id: ha_physical_relay).

###  Stage 3 - Backlight Control stayed on permanently

*   Added LEDC-controlled backlight
*   Created restartable script
*   Backlight turns off after 60 seconds of inactivity
*   Any touch or HA-triggered state change wakes display

###  Stage 4 - Implement anti burn-in measures

*  Implemented LVGL pause/resume with snow animation: - Activated via HAswitch - Prevents OLED/LCD burn-in - Touch interaction resumes normal UI

###  Stage 5 - Add a slider to control HA relay turn-off delay

*  LVGL slider (0--300s)
*  Bound to HA number entity: `number.relay_relay_turn_off_delay`

#### Behavior

*  Slider updates HA
*  HA updates slider (bidirectional sync)
*  Invalid transient HA values filtered

### Stage 6 - Implement Local Persistence of Slider Value

#### Problem

Slider reset on reboot.

#### Solution

*  Stored delay value in ESPHome global
*  `restore_value: true`
*  Slider initialized from local value on boot


### Stage 7. Implement Countdown Display & UI Lockout

#### Feature Added

When relay is turned off with a delay: - Countdown label appears -
Button and slider are disabled - UI shows: `Turning off in: X s`

Countdown driven by ESPHome `interval` timer.



## Stage 8. HA Reconnect Synchronization

On HA reconnect: - Slider value synced from HA - Labels refreshed -
Relay state label updated

Ensures clean recovery after Wi-Fi / HA restarts.


## Stage 9. Fix 'Countdown from 0' bug  

Fix a bug where the slider is set to 0, button is pressed, relay module switches on the physical relay. Press button (start countdown from 0) and relay module should switch the physical relay OFF, however this disables the button/slider effectively locking up the UI! Fix - Add checks to detect a count down value of zero and handle correctly.







##  Latest Display YAML for the - ESPHome with LVGL for the Guition JC4827W543C 4.3" LCD Display module  

Latest cyd display YAML version. Note that there may still be a few features (or should that be bugs) which require further investigation so use this with discretion. This whole exercise has been a ESPHome LVGL learning experience and a test of ChatGPT's expertise.

ENJOY!

```
esphome:
  #---------------------
  # version: 160126_1300
  #---------------------
  name: "cyd"
  platformio_options:
    board_build.flash_mode: dio
  on_boot:
    priority: -100
    then:
      - component.update: my_display
      - lvgl.slider.update:                                         #set display slider value (delay_slider) to global relay_turn_off_delay_local value
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
  - id: relay_turn_off_delay_local      # local value of the relay_off_delay timer. Make persistent (restore_value: true)
    type: int
    restore_value: true
    initial_value: '0'

  - id: countdown_active                # relay off countdown time active flag
    type: bool
    restore_value: no
    initial_value: 'false'

  - id: countdown_remaining             # relay delay time remaining
    type: int
    restore_value: no
    initial_value: '0'


interval:
  - interval: 1s
    then:
      - if:                                             # if countdown_active is true, then...
          condition:
            lambda: 'return id(countdown_active);'
          then:                                         # if the relay delay is active, then every second reduce countdown_remaining by 
            - lambda: |-
                if (id(countdown_remaining) > 0) {
                  id(countdown_remaining)--;
                }
            - lvgl.widget.disable:                      # disable delay progress slider, then...
                id: delay_prog_slider
            - lvgl.slider.update:                       # update progress slider using countdown_remaining
                id: delay_prog_slider
                hidden: false
                value: !lambda return id(countdown_remaining);
            - lvgl.label.update:                        # then update the UI countdown_prog_label...define a buffer for the UI label, then put the countdown remaining value into the buffer
                id: delay_prog_label
                hidden: false
                text: !lambda |-
                  static char buf[36];
                  snprintf(buf, sizeof(buf), "Physical Relay Turning off in: %d s", id(countdown_remaining));
                  return buf;

# -------------------------------
# I2C + Touchscreen hardware setup
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
        - script.execute: restart_backlight_timer                   # touching the screen runs script 
    on_release:
      then:
        - if:
            condition: lvgl.is_paused                               # and (if antiburn is running), switches it off and restores UI
            then:
              - lvgl.resume
              - lvgl.widget.redraw
              - logger.log: "Antiburn stopped by touch"

# -------------------------------
# Display... spi hardware setup. 
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
# H/W Backlight Setup
# -------------------------------
output:
  - platform: ledc
    pin: GPIO1
    id: gpio_backlight_pwm
    frequency: 1000Hz

#-----------------------------------------------------------
# light component. Available as entity in HA Cyd Display Backlight
#-----------------------------------------------------------
light:
  - platform: monochromatic
    output: gpio_backlight_pwm
    id: display_backlight
    name: Display Backlight
    restore_mode: ALWAYS_OFF


# -------------------------------
# scripts - self documenting
#--------------------------------
script:
  - id: restart_backlight_timer
    mode: restart
    then:
      - switch.turn_off: switch_antiburn
      - light.turn_on: display_backlight
      # screen timeout delay is 60s unless relay off countdown is active, then 60s is added to the delay 
      - delay: !lambda "if (id(countdown_active)) return ((id(relay_turn_off_delay_local) + 60)* 1000); else return 60000;"
      - light.turn_off: display_backlight
      - delay: 1s
      - switch.turn_on: switch_antiburn

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

#-----------------------------------------------
# Antiburn - available as a switch on HA
#-----------------------------------------------
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
      then:                                                     # if the delay entity (from HA) updates then...sanity check it
        - lambda: |-
            if (x < 0 || x > 300) {
              // Ignore any transient / invalid HA values
              return;
            }
            id(relay_turn_off_delay_local) = (int)x;
        - lvgl.slider.update:                                   # if valid, then set the local variable to be the same and update UI slider and its label
            id: delay_slider
            value: !lambda 'return (int)x;'
        - lvgl.label.update:
            id: delay_label
            text: !lambda |-
              char buf[40];
              sprintf(buf, "Off Delay: %d s", (int)x);
              return std::string(buf);


# -------------------------------
# Import states of entities from HA and act upon any changes
# -------------------------------
binary_sensor:
  - platform: homeassistant
    id: ha_physical_relay
    entity_id: switch.relay_main_relay_physical
    on_state:
      then:
        - if:
            condition:
              binary_sensor.is_on: ha_physical_relay        # if physical relay state (of the relay module) goes on... then
            then:
              - lvgl.label.update:
                  id: relay_state_label
                  text: "Physical Relay State: ON"          # update UI label to indicate the physical relay is ON
            else:                                           # physical relay has switched OFF, so... enable UI 'Toggle Relay' button reset countdown flag and make counter zero
              - script.execute: enable_relay_button
              - lambda: |-
                  id(countdown_active) = false;
                  id(countdown_remaining) = 0;
              - lvgl.slider.update:                         # hide progress slider
                  id: delay_prog_slider
                  hidden: true
              - lvgl.label.update:                          # hide progress_label
                  id: delay_prog_label
                  hidden: true
              - lvgl.label.update:
                  id: relay_state_label
                  text: "Physical Relay State: OFF"         # update the UI physical relay state to OFF


  - platform: homeassistant
    id: ha_virtual_relay                                    # binary_sensor state is set from HA entity switch.relay_relay_control
    entity_id: switch.relay_relay_control
    on_state:
      then:
        - script.execute: restart_backlight_timer
        - if:
            condition:
              binary_sensor.is_off: ha_virtual_relay
            then:
              - if:
                  condition:
                    lambda: 'return id(relay_turn_off_delay_local) > 0;'
                  then:                                                                 # countdown is active (as relay_turn_off_delay_local is > 0)
                    # Start countdown UI ONLY if relay_turn_off_delay_local > 0
                    - lambda: |-
                        id(countdown_active) = true;
                        id(countdown_remaining) = id(relay_turn_off_delay_local);
                    - script.execute: disable_relay_button                              # disable the UI 'Toggle relay' button (and slider) during countdown

# -------------------------------
# LVGL UI
# -------------------------------
lvgl:
  pages:
    - id: main_page
      scrollable: false
      widgets:              # define widgets (on main_page) below. button,labels and slider
        #-----------------------------------------------------------
        # button widget (relay_button) - Contains text 'Toggle Relay' - used to switch local template switch (relay_proxy)
        # turn on backlight and restart backlight timer...may also start the 'countdown' 
        # if the physical relay state is reported to be ON (from HA)
        #------------------------------------------------------------
        - button:
            bg_color: orange
            id: relay_button
            x: 60
            y: 20
            width: 140
            height: 50
            text: "\uF021 Toggle Relay"
            on_click:
              then:
                - script.execute: restart_backlight_timer   #always restart the backlight (timeout) timer if clicked..screen comes ON
                - if:
                    condition:
                      binary_sensor.is_on: ha_physical_relay
                    then:
                      - if:
                          condition:
                            lambda: 'return id(relay_turn_off_delay_local) > 0;'
                          then:                                                             # set countdown_active flag if local variable > 0, write countdown to UI and disable the Toggle Relay button (as we are counting down)
                            - lambda: |-
                                id(countdown_active) = true;
                                id(countdown_remaining) = id(relay_turn_off_delay_local);
                            - script.execute: disable_relay_button
                      - switch.turn_off: relay_proxy                                        # turn on or off the local relay_proxy switch 
                    else:                                                                   # which drives the HA entity switch.relay_relay_control
                      - switch.turn_on: relay_proxy                                         # thus switching the relay board virtual switch

        - label:
            id: relay_state_label                                                           # UI label reports state of physical relay on the relay module
            x: 220
            y: 40            
            text: "Physical Relay State: OFF"

#
# define a slider widget to display the relay off timer (derived from HA entity number.relay_relay_turn_off_delay)
#
        - slider:
            id: delay_slider
            x: 60
            y: 100
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
            y: 120
            text: "Off Delay: 0 s"


        - slider:
            id: delay_prog_slider
            x: 60
            y: 180
            hidden: true
            width: 300
            min_value: 0
            max_value: 300

        - label:
            id: delay_prog_label
            text: ""
            x: 60
            y: 200
            hidden: true


# -------------------------------
# HA reconnect sync various things from HA...
# restarts the backlight timer... writes the local ha_relay_delay value to the UI... checks the current state of
# the physical relay (reported via HA driven binary_sensor) state is written to the UI
# -------------------------------
api:
  on_client_connected:
    then:
      - script.execute: restart_backlight_timer
      - switch.turn_off: switch_antiburn
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
