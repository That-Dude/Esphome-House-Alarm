# Esphome-House-Alarm
Inexpensive self contained alarm system based on Esphome with optional Home Assistant integration.

# Background
I live in a property with 4 well spaced out buildings that I would like to protect and and monitor, a house, two barns and a static caravan.

My goal is to build an alarm system for each building around the following objectives:

- [x] Built around the well supported ESP32 and Esphome
- [x] Off the shelf parts
- [ ] Supports multiple wired PIR sensors and magnetic doors sensors
- [ ] Supports NFC tags to arm and disarm
- [x] Inexpensive to build
- [x] Easy to extend
- [x] Easy to program (Esphome)
- [x] Works independently with it's own web page but  also has native Home Assistant integration
- [x] Fully wired
- [ ] Battery backup - I'm not sure how to approach this atm, maybe a 12v battery pack and inline mains sensor?
- [ ] Ethernet with PoE - Take advantage of the UPS on my network switch and reliability of Ethernet over WiFi
- [ ] 3D Printed enclosure or adapt existing project box?

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


