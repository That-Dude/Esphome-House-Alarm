# Esphome-House-Alarm
Inexpensive self contained alarm system based on Esphome with optional Home Assistant integration.

Note: I'm sure there are more complete projects on github for building your own alarm system. I specifically wanted to develop my own as it gives me the opertunity to learn more about software / hardware design and development, while also being a fun and practical.

I'm very open to critique and feedback to improve all aspects of my project, please 

![Esphome webserver](/esphome-webserver-screenshot.png)

# Background
I live in a property with 4 well spaced out buildings that I would like to protect and and monitor, a house, two barns and a static caravan.

My goal is to build an alarm system for each building around the following objectives:

- [x] Fully self contained and works locally with no Internet required
- [x] When Internet is available sends updates to mobile devices using PushOver (could also use telegram)
- [x] Built around the well supported ESP32 and Esphome software / hardware combo
- [x] Off the shelf parts
- [x] Supports multiple wired PIR sensors and magnetic doors sensors
- [x] Supports NFC tags to arm and disarm
- [x] Inexpensive to build
- [x] Easy to extend
- [x] Easy to program (Esphome)
- [x] Has it's own password protected web management page and native Home Assistant integration
- [x] Fully wired
- [x] Supports NFC tags to arm and disarm

## Todo:

- [ ] Battery backup - I'm not sure how to approach this atm, maybe a 12v battery pack and inline mains sensor? Ideally an off the shelf part exists. I might resolve this using PoE as my switch has a large UPS already.
- [ ] Ethernet option
- [ ] 3D Printed enclosure or adapt off the shelf project box?
- [ ] Learn Kicad and produce a custom PCB via PCBway or JLpcb (this looks like a fun learning experience)
- [ ] Consider integrating 433Mhz receiver to interact with remote 433Mhz PIR and magnet sensors

## Why not use WiFi / Zigbee / Z-Wave sensors?

I have experimented with all 3 of these wireless options in combination with Home Assistant but I've struggled with reliability, namely:

- Connections dropping / Interference
- Battery issues on sensors
- Home Assistant / Z2M / Wifi issues
- Power loss
- Updates to devices and Home Assistant
- Reboots

When it works it's great! But it's simply not reliable enough to meet my needs.

Shutout to Ararmo on Home Assistant - which I love! it's an excellent option if you choose to go that route. Note, this project is actually compatible with Alarmo, all of the PIR and magnet sensors show up in Home Assistant.

# Prototype setup

![Ptototype image](/prototype.jpg)

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

# Esphome code

````yaml
esphome:
  name: "house-alarm"
  friendly_name: "House Alarm"
  on_boot:
    priority: 250
    then:
      - http_request.post:
          url: https://api.pushover.net/1/messages.json
          headers:
            Content-Type: application/json
          json:
            token: !secret pushover_api_token
            user: !secret pushover_user_key
            message: "Device just booted..."
            title: "House Alarm"

http_request:
  verify_ssl: false

############ Boiler plate ############
esp32:
  board: esp32dev
  framework:
    type: arduino

# Enable logging
logger:
  level: DEBUG

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


################################################################################################ start ################################################################################################

############ Siren output  ############
# alarm siren is on pin 23. We activate the mosfet gate allowing 12v to pass
# A relay or trasister would do this just fine, I just had a mosfet module to hand
switch:
  - platform: gpio
    pin: 
      number: GPIO23
      mode: output
    id: siren

############ Status status_led  ############
  - platform: gpio
    pin: 
      number: 4
      mode: output
    id: status_led

############ Alarm activation state  ############
# This switch tracks the state of the alarm, and can be used to manually activate it
  - platform: template
    name: "Armed State"
    id: armed_state
    optimistic: true
    on_turn_on: 
      then:
        - switch.turn_on: status_led
    on_turn_off: 
      then:
        - switch.turn_off: status_led
        - switch.turn_off: siren

button:
  - platform: template
    id: "rfid_detected"
    name: "RFID Detected"
    on_press:
      then:
        - if:
            condition:
              switch.is_off: armed_state
            then: # arm the alarm
              - logger.log: "*** INFO: Alaarm is off, we are arming it..."
              - http_request.post:
                  url: https://api.pushover.net/1/messages.json
                  headers:
                    Content-Type: application/json
                  json:
                    token: !secret pushover_api_token
                    user: !secret pushover_user_key
                    message: "Arming alarm..."
                    title: "House Alarm"
              - switch.turn_off: siren # just in case
              - logger.log: "*** INFO: starting 10s delay..."
              - repeat:
                  count: 20 # 20 * 500ms = 10s 
                  then:
                    - switch.turn_on: status_led
                    - delay: "250ms"
                    - switch.turn_off: status_led
                    - delay: "250ms"
              - logger.log: "*** INFO: Ending 10s delay..."
                # You should be out of the house at this point with the doors closed. Let's verify that....
              - if: 
                  condition:
                    binary_sensor.is_on: PIR02
                  then:
                    - logger.log: "*** ERROR: PIR02 prevented Arming"
                    - http_request.post:
                        url: https://api.pushover.net/1/messages.json
                        headers:
                          Content-Type: application/json
                        json:
                          token: !secret pushover_api_token
                          user: !secret pushover_user_key
                          message: "ERROR: PIR01 prevented Arming"
                          title: "House Alarm"
              - if:
                  condition:
                    binary_sensor.is_on: MAGNET01
                  then:
                    - logger.log: "*** ERROR: MAGNET01 prevented Arming"
                    - http_request.post:
                        url: https://api.pushover.net/1/messages.json
                        headers:
                          Content-Type: application/json
                        json:
                          token: !secret pushover_api_token
                          user: !secret pushover_user_key
                          message: "ERROR: MAGNET01 prevented Arming"
                          title: "House Alarm"
                  else:
                    - switch.turn_on: armed_state
                    - logger.log: "*** INFO: Alarm Armed"
                    - http_request.post:
                        url: https://api.pushover.net/1/messages.json
                        headers:
                          Content-Type: application/json
                        json:
                          token: !secret pushover_api_token
                          user: !secret pushover_user_key
                          message: "INFO: Alarm Armed"
                          title: "House Alarm"
            else:
              # disarm the alarm
              - switch.turn_off: armed_state
              - switch.turn_off: siren
              - logger.log: "*** INFO: Alarm is armed, disarming with RFID Tag"
              - http_request.post:
                  url: https://api.pushover.net/1/messages.json
                  headers:
                    Content-Type: application/json
                  json:
                    token: !secret pushover_api_token
                    user: !secret pushover_user_key
                    message: "INFO: Alarm is armed, disarming with RFID Tag"
                    title: "House Alarm"
              

############ Trigger the alarm  ############
 # . This is the full blown alarm sound, default 60 seconds
  - platform: template
    id: "AlarmTriggered"
    name: "Alarm Triggered"
    on_press:
      then:
      # Sends a high priority alert notifcation to my pushover devices (phone will sound even during quiet hours and you have to acknowstatus_ledge the message to stop it)
        - http_request.post:
            url: https://api.pushover.net/1/messages.json
            headers:
              Content-Type: application/json
            json:
              token: !secret pushover_api_token
              user: !secret pushover_user_key
              message: "The alarm has triggered"
              title: "Alarm Triggered"
              sound: "siren" # optional
              priority: "2" # -2 to 2
              retry: "30" # retry is mandatory with prio 2
              expire: "60" # expire is mandatory with prio 2
      # Tuen on the siren for X seconds
        - switch.turn_on: siren
        - delay: "300s" # siren sounds for 5 minures
        - switch.turn_off: siren

# Setup the UART for the RDM6300 RFID reader
uart:
  rx_pin: GPIO13
  baud_rate: 9600

# Enable the RFID reader
rdm6300:

binary_sensor:

################################################################################################ PIR Sensors ################################################################################################

############# PIR setup pins 34,35,36,39 ############

# These require the internal pullup resister to be disabstatus_led
  # - platform: gpio
  #   id: PIR01
  #   name: "PIR 01"
  #   device_class: motion
  #   filters:
  #     - delayed_off: 1000ms # the PIR sends a lot of activation traffic, this debouces it a single trigger
  #   pin:  
  #       number: GPIO39
  #       mode:
  #           input: true
  #   on_press:
  #     then:
  #       - if:
  #           condition:
  #              switch.is_on: armed_state
  #           then:
  #             - button.press: AlarmTriggered


  - platform: gpio
    id: PIR02
    name: "PIR 02"
    device_class: motion
    filters:
      - delayed_off: 1000ms # the PIR sends a lot of activation traffic, this debouces it a single trigger
    pin:  
        number: GPIO36
        mode:
            input: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: armed_state
            then:
              - button.press: AlarmTriggered
              - logger.log: "*** ALERT: Alarm Tripped - PIR02"

  # - platform: gpio
  #   id: PIR03
  #   name: "PIR 03"
  #   device_class: motion
  #   filters:
  #     - delayed_off: 1000ms # the PIR sends a lot of activation traffic, this debouces it a single trigger
  #   pin:  
  #       number: GPIO35
  #       mode:
  #           input: true
  #   on_press:
  #     then:
  #       - if:
  #           condition:
  #              switch.is_on: armed_state
  #           then:
  #             - button.press: AlarmTriggered

  # - platform: gpio
  #   id: PIR04
  #   name: "PIR 04"
  #   device_class: motion
  #   filters:
  #     - delayed_off: 1000ms # the PIR sends a lot of activation traffic, this debouces it a single trigger
  #   pin:  
  #       number: GPIO34
  #       mode:
  #           input: true
  #   on_press:
  #     then:
  #       - if:
  #           condition:
  #              switch.is_on: armed_state
  #           then:
  #             - button.press: AlarmTriggered

################################################################################################ Magnet Sensors ################################################################################################

# MAGNET01 is the MAIN DOOR and allows a 10s delay for the user to disarm the alarm before triggering
  - platform: gpio
    id: MAGNET01
    name: "MAGNET 01"
    device_class: door
    pin:  
        number: GPIO32
        mode:
            input: true
            pullup: true
        inverted: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: armed_state
            then:
              - logger.log: "*** INFO: Main Door opened, starting entry grace period..."
              - repeat:
                  count: 20 # 20 * 500ms = 10s 
                  then:
                    - switch.turn_on: status_led
                    - delay: "250ms"
                    - switch.turn_off: status_led
                    - delay: "250ms"
              - if:
                  condition:
                    switch.is_off: armed_state
                  then:
                    - logger.log: "*** INFO: Disarmed during entry grace period"
                  else:
                    - button.press: AlarmTriggered
                    - logger.log: "*** ALERT: Alarm Tripped - MAGNET01 - user did NOT disable during grace period"
          

  - platform: gpio
    id: MAGNET02
    name: "MAGNET 02"
    device_class: door
    pin:  
        number: GPIO33
        mode:
            input: true
            pullup: true
        inverted: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: armed_state
            then:
              - button.press: AlarmTriggered
              - logger.log: "*** ALERT: Alarm Tripped - MAGNET02"

  - platform: gpio
    id: MAGNET03
    name: "MAGNET 03"
    device_class: door
    pin:  
        number: GPIO25
        mode:
            input: true
            pullup: true
        inverted: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: armed_state
            then:
              - button.press: AlarmTriggered
              - logger.log: "*** ALERT: Alarm Tripped - MAGNET03"

  - platform: gpio
    id: MAGNET04
    name: "MAGNET 04"
    device_class: door
    pin:  
        number: GPIO26
        mode:
            input: true
            pullup: true
        inverted: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: armed_state
            then:
              - button.press: AlarmTriggered
              - logger.log: "*** ALERT: Alarm Tripped - MAGNET04"

################################################################################################ NFC Tags ################################################################################################

  - platform: rdm6300
    uid: !secret red_nfc_tag
    name: "NFC tag: RED"
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debouces it a single activation
      - delayed_off: 200ms
    on_press:
      then:
        - button.press: rfid_detected

  - platform: rdm6300
    uid: !secret blue_nfc_tag
    name: "NFC tag: BLUE"
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debouces it a single activation
      - delayed_off: 200ms
    on_press:
      then:
        - button.press: rfid_detected

  - platform: rdm6300
    uid: !secret green_nfc_tag
    name: "NFC tag: GREEN"
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debouces it a single activation
      - delayed_off: 200ms
    on_press:
      then:
        - button.press: rfid_detected

````

# Available GPIO pins

![ESP32 WRoom Pinout](/esp32_pinout.jpg)

Initially I planned on adding a multiplexer chip to make more inputs available, but it turns out the standard ESP32Wroom modules have 19 pins that we can use for an alarm system, 15 pins for general IO and 4 more pins that are input only - which is perfect for out PIR and magnetic door sensors.

I need a maximum of 4 PIRs and 4 Door sensors, so I've assigned my pins like this:

34,35,36,39 - PIR sensors
25,26,32,33 - Door sensors
23 - Siren (connected to MOSFET)
13 - RFID reader (UART RX connected to RM6300 TX pin)
4 - RED LED to indicate armed status

11 PINS used, 8 available for other things:

- A piezo buzzer for audio feedback
... other stuff