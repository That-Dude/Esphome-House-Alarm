# Esphome-House-Alarm
Inexpensive alarm system based on Esphome with Home Assistant integration.

# Background
I live in a property with 4 well spaced out buildings that I would like to protectand and monitor, a house, two barns and a static caravan.

My goal is to build an alarm system for each building around the following objectives:

- [x] Built around the well supported ESP32 and Esphome
- [x] Off the shelf parts
- [ ] Supports multiple wired PIR sensros and magnetic doors sensors
- [ ] Supports NFC tags to arm and disarm
- [x] Inexpensive to build
- [x] Easy to extend
- [x] Easy to program (Esphome)
- [x] Works independanly with it's own web page but  also has native Home Assistant itegration
- [x] Fully wired
- [ ] Battery backup - I'm not sure how to apprach this atm
- [ ] Ethernet with PoE -Take advancte of the UPS on my network switch and reliabilty of Ethernet over WiFi

## Why not use WiFi / Zigbee / Z-Wave sensors?

I have experentmented with all 3 of these wireless systems but I've stuggled with reliabilty, namely:

- Connections dropping / Interferance
- Battery issues
- Home Assisant / Z2M / Wifi issues
- Power loss
- Updates
- Reboots

# Prototype setup

![alt text](/prototype.jpg)

Bill of Materials (prototype) all parts from AliExpress:

- Esp32Wroom £2.30
- Mains to 12vdc 1A trasformer (LED driver) £1.71
- 12vdc PIR Motion Sensor Wired Alarm Dual Infrared Detector Pet Immune (2 pack) £10.13
- Magnetic door sensor (10 pack) £11.19
- 12vdc Flashing Siren, extremely loud £2.65
- RDM6300 NFC reader 125Khz RFID tag reader £1.52
- RFID tags (10 pack) £2.17
- MOSFET realy module to tigger the siren £1.10 - I had this laying aroung, you could use a logic level transister which would cost £0.10
- DC-DC Buck Power Supply Module (12v to 5v) £0.62 (I'm using something similar that I had laying around)
- Wago 5 way connectors £0.9 each
- 100m Roll of 6 core alarm cable £10.00 (inc delivery) for CPC Farnell in the UK

Total build cost for Prototype: £42.47 or $53 if that's your thing :-)

You could make this cheaper, I've opted for PIR sensors with pet avoidance which cost £5.06 each, I saw PIR's for less than £1.00 that would probabply work in a pinch.

NB: You can also buy a fully working alarm kit from AliExpress for a lower cost, which is way easier! But I wanted the niceties of an opensource system with bespoke parts to suit my needs. Plus the fun of deleoping it myself.


