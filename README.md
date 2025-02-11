# Esphome-House-Alarm
Inexpensive self contained alarm system based on Esphome with optional Home Assistant integration.

I'm very open to critique and feedback to improve all aspects of this project, let's talk!

![Esphome webserver](/esphome-webserver-screenshot.png)

# Background
I live in a property with 4 separate buildings that I would like to protect and and monitor, a house, two barns and a static caravan.

My goal is to build an alarm system for each building around the following objectives:

- [x] Fully self contained and works locally with no Internet or Home Assistant required
- [x] When Internet is available sends updates to mobile devices using PushOver (could also use telegram or whatever)
- [x] Built around ESP32 and Esphome software stack
- [x] Off The Shelf components
- [x] Supports multiple wired PIR sensors and magnetic door/window sensors
- [x] Supports NFC/RFID tags to arm and disarm
- [x] Inexpensive to build
- [x] Easy to extend
- [x] Easy to program (Esphome)
- [x] Has it's own password protected web management page and native Home Assistant integration
- [x] Fully wired
- [x] All sensors available in Home Assistance for non-alarm purposes such as presence detection and to check door/window status, you can also use those sensors in Alarmo if you don't need a self contained system.

## Todo:
- [ ] Battery backup - I'm not sure how to approach this atm, maybe a 12v battery pack and inline mains sensor? Ideally an off the shelf part exists. I might resolve this using PoE as my network switch has a large UPS.
- [ ] Ethernet option (with or without PoE)
- [ ] 3D Printed enclosure or adapt an off the shelf project box?
- [ ] Learn Kicad and produce a custom PCB via PCBway or JLpcb (this looks like a fun learning experience)
- [ ] Maybe swap some PIR sensors for MmWave sensors, they are extremely cheap and might be a good option in some spaces. I'm thinking above rooms where ceiling access is available.

## Why not use WiFi / Zigbee / Z-Wave sensors?
TL:DR - Reliability

I have a lot of experimented with all 3 of these wireless options in combination with Home Assistant but I've struggled with reliability, namely:

- Connections dropping / Interference
- Battery depletion on sensors
- Home Assistant / Z2M / Wifi issues
- Power loss
- Breaking updates to devices and Home Assistant 
- Reboots of any critical component

When it works it's great! But it's simply not reliable enough to meet my needs due to the number of moving parts. It's hard to beat a single lower power devices with all sensors hard wired to it!

# Prototype setup
![Ptototype image](/prototype.jpg)

The first step was to put together a working prototype. After lots of trial and error these are the components that worked for me.

### Bill of Materials (prototype) all parts from AliExpress:
- Esp32Wroom £2.30
- Mains to 12vdc 1A transformer (LED driver) £1.71
- 12vdc PIR Motion Sensor Wired Alarm Dual Infrared Detector Pet Immune (2 pack) £10.13
- Magnetic door sensor - high quality (10 pack) £11.19
- 12vdc Flashing Siren, extremely loud £2.65
- RDM6300 NFC reader 125Khz RFID tag reader £1.52
- RFID tags (10 pack) £2.17
- MOSFET driver module to trigger the siren £1.10 - I had this laying around, you could use a logic level transistor which would cost £0.10
- DC-DC Buck Power Supply Module (12v to 5v) £0.62 (I'm using something similar that I had laying around)
- Wago 5 way connectors (temporary solution) £0.9 each
- 100m Roll of 6 core alarm cable £10.00 (inc delivery) for CPC Farnell in the UK

Total build cost for Prototype around £42 or $53 if that's your thing :-)

You could certainly make this cheaper, I've opted for PIR sensors with pet avoidance which cost £5.06 each, I saw PIR's for less than £1.00 that would probably work in a pinch, MmWare sensors are also very cheap.

# Esphome code
Here is my working code, it supports:

- 4 Magnet sensors (supports many - see section below code)
- 4 PIRs (supports many - see section below code)
- 1 siren output (supports many)
- 1 NFC reader (2 supported)
- 3 NFC Tags (supports many)
- 1 Status LED
- 1 Buzzer for feedback

The code is commented and (hopefully) easy to read and alter for you needs.

````yaml
esphome:
  name: "house-alarm"
  friendly_name: "House Alarm"
  on_boot:
    priority: 250
    then:
      - text_sensor.template.publish:
          id: po_message
          state: "Alarm booted"
      - script.execute: pushover_message

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


################################### start ###################################
substitutions:
  buzzer_pin: GPIO5
  siren_pin: GPIO23
  led_pin: GPIO4
  rfid_reader_pin: GPIO13
  pir01_pin: GPIO39
  pir02_pin: GPIO36
  pir03_pin: GPIO35
  pir04_pin: GPIO34
  magnet01_pin: GPIO32
  magnet02_pin: GPIO33
  magnet03_pin: GPIO25
  magnet04_pin: GPIO26


output:
  - platform: ledc
    # One buzzer leg connected to buzzer_pin, the other to GND
    pin: $buzzer_pin
    id: buzzer

# Siren output
# alarm siren is on pin siren_pin. We activate the mosfet gate allowing 12v to pass
# A relay or trasister would do this just fine, I just had a mosfet module to hand
switch:
  - platform: gpio
    pin: 
      number: $siren_pin
      mode: output
    id: siren

# Status LED
  - platform: gpio
    pin: 
      number: $led_pin
      mode: output
    id: status_led

# Alarm activation state
# This switch tracks the state of the alarm, and can be used to manually activate it from the web UI or Home Assistant
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

# I'm using this as a string variable for the pushover messaging script
text_sensor:
  - platform: template
    name: "Pushover message"
    id: po_message

script:
   # Pushover message script    
  - id: pushover_message
    then:
      - http_request.post:
          url: https://api.pushover.net/1/messages.json
          headers:
            Content-Type: application/json
          json:
            token: !secret pushover_api_token
            user: !secret pushover_user_key
            title: "House Alarm"
            message: !lambda |-
              return id(po_message).state;

# Set a grace period when arming / disarming the alarm with specific sensors, in my case I want the main door manget to:
# : On Entry: Don't set off the siren immediately, but give me XX senconds to present an RFID tag to disable the alarm
# : On exit: give me XX seconds to close the main door before arming.
  - id: grace_period
    then:
      - repeat: # beep and flash the LED x times
          count: 10
          then:
            - switch.turn_on: status_led
            - delay: 200ms
            - output.turn_on: buzzer
            - output.ledc.set_frequency:
                id: buzzer
                frequency: "2000Hz"
            - output.set_level:
                id: buzzer
                level: "50%"
            - delay: 200ms
            - output.turn_off: buzzer
            - switch.turn_off: status_led
            - if:
                condition: # in the loop, check to see if an rfid tag as presented to disarm during the grace period
                  or:
                    - binary_sensor.is_on: green
                    - binary_sensor.is_on: red
                    - binary_sensor.is_on: blue
                then:
                  - script.stop: rfid_detected
                  - text_sensor.template.publish:
                      id: po_message
                      state: "INFO: Disarmed during grace period"
                  - script.execute: pushover_message
                  - script.stop: grace_period

  # Error beep sound    
  - id: error_beep
    then:
      - output.turn_on: buzzer
      - output.ledc.set_frequency:
          id: buzzer
          frequency: "2000Hz"
      - output.set_level:
          id: buzzer
          level: "50%"
      - delay: 3s
      - output.ledc.set_frequency:
          id: buzzer
          frequency: "1800Hz"
      - output.set_level:
          id: buzzer
          level: "50%"
      - delay: 3s
      - output.turn_off: buzzer
 
  # Process to arm / disarm the alarm after an RFID is presented
  - id: rfid_detected
    then:
      if:
        condition:
          switch.is_off: armed_state
        then: # arm the alarm
          - logger.log:
              format: "** INFO: Alaarm is off, we are arming it..."
              level: info
          - delay: 300ms # required to allow rfid binary sensor time to tunr off
          - text_sensor.template.publish:
              id: po_message
              state: "INFO: Arming..."
          - script.execute: pushover_message
          - script.execute: grace_period
          - script.wait: grace_period
          - if: # make sure these sensors are OFF (door closed) before actually arming...
              condition:
                binary_sensor.is_on: PIR02
              then:
                - logger.log:
                    format: "*** ERROR: PIR02 prevented Arming"
                    level: info
                - text_sensor.template.publish:
                    id: po_message
                    state: "ERROR: PIR02 prevented Arming"
                - script.execute: pushover_message                
          - if:
              condition:
                binary_sensor.is_on: MAGNET01
              then:
                - script.execute: error_beep
                - logger.log:
                    format: "*** *** ERROR: MAGNET01 prevented Arming"
                    level: info
                - text_sensor.template.publish:
                    id: po_message
                    state: "ERROR: MAGNET01 prevented Arming"
                - script.execute: pushover_message     
              else:
                - switch.turn_on: armed_state # all sensors are CLEAR - tunn on the alarm
                - logger.log:
                    format: "*** INFO: Sensors are clear - Alarm Armed"
                    level: info
                - text_sensor.template.publish:
                    id: po_message
                    state: "INFO: Alarm is now ARMED"
                - script.execute: pushover_message     
        else:
          # disarm the alarm
          - switch.turn_off: armed_state
          - logger.log:
              format: "*** INFO: Alarm DISARMED"
              level: info
          - text_sensor.template.publish:
              id: po_message
              state: "INFO: Alarm DISARMED"
          - script.execute: pushover_message
          
############ Trigger the siren  ############
  - id: alarm_triggered
    then:
    # Sends a high priority alert notifcation to my pushover devices
    # Phone will sound even during quiet hours and you have to acknowstatus_ledge the message to stop it
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
    # turn on the siren
      - switch.turn_on: siren


# Setup the UART for the RDM6300 RFID reader
uart:
  rx_pin: $rfid_reader_pin
  baud_rate: 9600

# Enable the RFID reader
rdm6300:

binary_sensor:
################################### PIR Sensors ###################################

############# PIR setup pins 34,35,36,39 ############

# These require the internal pullup resister to be disabled

  # - platform: gpio
  #   id: PIR01
  #   name: "PIR 01"
  #   device_class: motion
  #   filters:
  #     - delayed_off: 300ms # the PIR sends a lot of activation traffic, this debounces it a single trigger
  #   pin:  
  #       number: $pir01_pin
  #       mode:
  #           input: true
    # on_press:
    #   then:
    #     - if:
    #         condition:
    #            switch.is_on: armed_state
    #         then:
    #           - button.press: AlarmTriggered
    #           - logger.log: "*** ALERT: Alarm Tripped - PIR01"

  - platform: gpio
    id: PIR02
    name: "PIR 02"
    device_class: motion
    filters:
      - delayed_off: 300ms # the PIR sends a lot of activation traffic, this debounces it a single trigger
    pin:  
        number: $pir02_pin
        mode:
            input: true
    on_press:
      then:
        - if:
            condition:
               switch.is_on: armed_state
            then:
              - script.execute: alarm_triggered
              - logger.log:
                  format: "*** ALERT: Alarm Tripped - PIR02"
                  level: info     
                          
  # - platform: gpio
  #   id: PIR03
  #   name: "PIR 03"
  #   device_class: motion
  #   filters:
  #     - delayed_off: 300ms # the PIR sends a lot of activation traffic, this debounces it a single trigger
  #   pin:  
  #       number: $pir03_pin
  #       mode:
  #           input: true
  #   on_press:
  #     then:
  #       - if:
  #           condition:
  #              switch.is_on: armed_state
  #           then:
  #             - button.press: AlarmTriggered
  #             - logger.log: "*** ALERT: Alarm Tripped - PIR03"

  # - platform: gpio
  #   id: PIR04
  #   name: "PIR 04"
  #   device_class: motion
  #   filters:
  #     - delayed_off: 300ms # the PIR sends a lot of activation traffic, this debounces it a single trigger
  #   pin:  
  #       number: $pir04_pin
  #       mode:
  #           input: true
  #   on_press:
  #     then:
  #       - if:
  #           condition:
  #              switch.is_on: armed_state
  #           then:
  #             - button.press: AlarmTriggered
  #             - logger.log: "*** ALERT: Alarm Tripped - PIR03"

################################### Magnet Sensors ###################################

# MAGNET01 is the MAIN DOOR and allows a 10s delay for the user to disarm the alarm before triggering
  - platform: gpio
    id: MAGNET01
    name: "MAGNET 01"
    device_class: door
    pin:  
        number: $magnet01_pin
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
              - logger.log:
                  format: "*** INFO: Main Door opened, starting entry grace period..."
                  level: info
              - script.execute: grace_period
              - script.wait: grace_period
              - if:
                  condition:
                    switch.is_off: armed_state
                  then:
                    - logger.log:
                        format: "*** INFO: Disarmed during entry grace period"
                        level: info                    
                  else:
                    - script.execute: alarm_triggered
                    - logger.log:
                        format: "*** ALERT: Alarm Tripped - MAGNET01 - user did NOT disable during grace period"
                        level: info     

  - platform: gpio
    id: MAGNET02
    name: "MAGNET 02"
    device_class: door
    pin:  
        number: $magnet02_pin
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
              - script.execute: alarm_triggered
              - logger.log:
                  format: "*** ALERT: Alarm Tripped - MAGNET02"
                  level: info              

  - platform: gpio
    id: MAGNET03
    name: "MAGNET 03"
    device_class: door
    pin:  
        number: $magnet03_pin
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
              - script.execute: alarm_triggered             
              - logger.log: "*** ALERT: Alarm Tripped - MAGNET03"

  - platform: gpio
    id: MAGNET04
    name: "MAGNET 04"
    device_class: door
    pin:  
        number: $magnet04_pin
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
              - script.execute: alarm_triggered
              - logger.log: "*** ALERT: Alarm Tripped - MAGNET04"

################################### NFC Tags ###################################

  - platform: rdm6300
    uid: !secret red_nfc_tag

    name: "NFC tag: RED"
    id: red
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debounces it a single activation
      - delayed_off: 500ms
    on_press:
      then:  
        # - text_sensor.template.publish:
        #     id: po_message
        #     state: "INFO: RED tag presented..."
        # - script.execute: pushover_message
        - script.execute: rfid_detected

  - platform: rdm6300
    uid: !secret blue_nfc_tag
    name: "NFC tag: BLUE"
    id: blue
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debounces it a single activation
      - delayed_off: 500ms
    on_press:
      then:  
        - script.execute: rfid_detected

  - platform: rdm6300
    uid: !secret green_nfc_tag
    name: "NFC tag: GREEN"
    id: green
    device_class: presence
    filters: # the NFC reader sends a lot of activation traffic, this debounces it a single activation
      - delayed_off: 500ms
    on_press:
      then:  
        - script.execute: rfid_detected
````

# Available GPIO pins and more sensors

![ESP32 Wroom Pinout](/esp32_pinout.jpg)

Initially I planned on adding a multiplexer chip to make more inputs available, but it turns out the standard ESP32Wroom modules have 19 pins that we can use for an alarm system, 15 pins for general IO and 4 more pins that are input only - ideal for sensors. 

I need a maximum of 4 PIRs and 4 Door sensors, so I've assigned my pins like this:

- 34,35,36,39 - PIR sensors
- 25,26,32,33 - Door sensors
- 23 - Siren (connected to MOSFET)
- 13 - RFID reader (UART RX connected to RM6300 TX pin)
- 4 - RED LED to indicate armed status

11 PINS used, 8 available for other things.

If you need more GPIOs then a PCF8574 IO Expansion Board would be a good choice (£1.10 each), the boards connect to an I2C-Bus and have 8 GPIOs each, they can be connected in series to a maximum of 4 board or 32 additional GPIOs. The ESP32 supports dual I2C buses so that's 64 GPIOs on the expanders, plus 19 GPIOs on the esp32 for a total of 83 GPIOs.

# FAQ

#### Q: Why didn't you add a keypad?
A: I chose RFID tags as they allow me to track the alarm users - the logs show me who activated / deactivated the alarm and when. You also have the web interface or Home Assistant if you need to arm/disarm the alarm remotely. It's totally possible to add keypads if you need them.

#### Q: The ESPhome log complains about "components taking a long time"
A: You can ignore this, the "long time" is the debouncing delay for the RFID reader, the PIR sensors and the website interface updates, all built in ESPhome features.

#### Q: Do I have to recompile and update the ESPhome code in order to revoke or add a new RFID tag?

A: Yes. I can't think of a good solution to avoid this, but the code is easy to manage so it only takes a minute to update.