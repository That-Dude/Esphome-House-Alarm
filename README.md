# Esphome-House-Alarm
Inexpensive self contained alarm system based on Esphome with optional Home Assistant integration.

# Background
I live in a property with 4 well spaced out buildings that I would like to protect and and monitor, a house, two barns and a static caravan.

My goal is to build an alarm system for each building around the following objectives:

- [x] Built around the well supported ESP32 and Esphome
- [x] Off the shelf parts
- [x] Supports multiple wired PIR sensors and magnetic doors sensors
- [x] Supports NFC tags to arm and disarm
- [x] Inexpensive to build
- [x] Easy to extend
- [x] Easy to program (Esphome)
- [x] Works independently with it's own web page but  also has native Home Assistant integration
- [x] Fully wired

To do:

- [ ] Battery backup - I'm not sure how to approach this atm, maybe a 12v battery pack and inline mains sensor?
- [ ] Ethernet with PoE - Take advantage of the UPS on my network switch and reliability of Ethernet over WiFi
- [ ] 3D Printed enclosure or adapt existing project box?
- [ ] Learn Kicad and have a custom PCB produced by PCBway of JLpcb

## Why not use WiFi / Zigbee / Z-Wave sensors?

I have experimented with all 3 of these wireless systems but I've struggled with reliability, namely:

- Connections dropping / Interference
- Battery issues
- Home Assistant / Z2M / Wifi issues
- Power loss
- Updates to devices and Home Assistant
- Reboots

# Prototype setup

![alt text](/prototype.jpg)

The first step was to put together a working prototype. After lots of trial and error these are the components that worked for me.

### Bill of Materials (prototype) all parts from AliExpress:

- Esp32Wroom £2.30
- Mains to 12vdc 1A transformer (LED driver) £1.71
- 12vdc PIR Motion Sensor Wired Alarm Dual Infrared Detector Pet Immune (2 pack) £10.13
- Magnetic door sensor (10 pack) £11.19
- 12vdc Flashing Siren, extremely loud £2.65
- RDM6300 NFC reader 125Khz RFID tag reader £1.52
- RFID tags (10 pack) £2.17
- MOSFET driver module to trigger the siren £1.10 - I had this laying around, you could use a logic level transistor which would cost £0.10
- DC-DC Buck Power Supply Module (12v to 5v) £0.62 (I'm using something similar that I had laying around)
- Wago 5 way connectors £0.9 each
- 100m Roll of 6 core alarm cable £10.00 (inc delivery) for CPC Farnell in the UK

Total build cost for Prototype: £42.47 or $53 if that's your thing :-)

You could make this cheaper, I've opted for PIR sensors with pet avoidance which cost £5.06 each, I saw PIR's for less than £1.00 that would probably work in a pinch.

NB: You can also buy a fully working alarm kit from AliExpress for a lower cost, which is way easier! But I wanted the niceties of an open source system with bespoke parts to suit my needs. Plus the fun of developing it myself.

#Esphome code

````
esphome:
  name: "house-alarm"
  friendly_name: "house-alarm"

############ Boiler plate ############
esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:
  level: debug

# Enable Web.
web_server:
  port: 80
  include_internal: true
  local: true  
  auth:
    username: admin
    password: !secret web_server_password

# Enable Home Assistant API
api:
  encryption:
    key: !secret alarm_api_key
  reboot_timeout: 0s

ota:
  - platform: esphome
    password: !secret alarm_ota_key

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  reboot_timeout: 0s

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Siren Fallback Hotspot"
    password: !secret alarm_fallback_hotspot_pass

captive_portal:

############ Siren ############
# alarm siren is on pin 23. We activate the mosfet gate allowing 12v to pass
# A relay or trasister would do this just fine, I just had a mosfet module to hand
output:
  - platform: gpio
    pin: GPIO23
    id: "out_1"
light:
  - platform: binary
    name: Siren
    id: siren
    output: out_1

# This switch tracks the state of the alarm, and can be used to manually activate it
switch:
  - platform: template
    name: "Alarm state"
    id: alarm_state
    optimistic: true
    lambda: return id(alarm_state).state;

button:
  # We want the sirent to 'chirp' X times when the RFID is used to activate it
  # Note; I plan to change this to a small piezo buzzer as the alarm is too loud
  # for general notifications
  - platform: template
    id: "AlarmGracePeriod"
    name: "Alarm Grace Period"
    on_press:
      - repeat:
          count: 2
          then:
            - output.turn_on: out_1
            - delay: "100ms"
            - output.turn_off: out_1
            - delay: "300ms"

  # Trigger the alarm. This is the full blown alarm sound, default 60 seconds
  - platform: template
    id: "AlarmTriggered"
    name: "Alarm Triggered"
    on_press:
      - repeat:
          count: 1
          then:
            - output.turn_on: out_1
            - delay: "60s"
            - output.turn_off: out_1

# Setup the UART for the RDM6300 RFID reader
uart:
  rx_pin: GPIO13
  baud_rate: 9600

# Enable the RFID reader
rdm6300:

binary_sensor:
# PIR's trgger pins connected directly to GPIO4
# this requires the esp32 internal pullup resister to be disabled
  - platform: gpio
    id: PIR01
    name: "PIR 01"
    device_class: motion
    filters:
      - delayed_off: 1000ms # the PIR sends a lot of activation traffic, this debouces it a single trigger
    pin:  
        number: GPIO4
        mode:
            input: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: alarm_state
            then:
              - button.press: AlarmTriggered

# Magnetic door sensor
  - platform: gpio
    id: MAGNET01
    name: "Door 01"
    device_class: door
    pin:  
        number: GPIO22
        mode:
            input: true
            pullup: true
        inverted: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: alarm_state
            then:
              - button.press: AlarmTriggered

# NFC tags arming and disarming the alarm
  - platform: rdm6300
    uid: !secret red_nfc_tag
    name: "NFC tag: RED"
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debouces it a single activation
      - delayed_off: 500ms
    on_press:
      then:
      - button.press: AlarmGracePeriod
      - switch.toggle: alarm_state

  - platform: rdm6300
    uid: !secret blue_nfc_tag
    name: "NFC tag: BLUE"
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debouces it a single activation
      - delayed_off: 500ms
    on_press:
      then:
      - button.press: AlarmGracePeriod
      - switch.toggle: alarm_state

  - platform: rdm6300
    uid: !secret green_nfc_tag
    name: "NFC tag: GREEN"
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debouces it a single activation
      - delayed_off: 500ms
    on_press:
      then:
      - button.press: AlarmGracePeriod
      - switch.toggle: alarm_state
````
