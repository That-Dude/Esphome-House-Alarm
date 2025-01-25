# Esphome-House-Alarm
Inexpensive alarm system based on Esphome with Home Assistant integration.

# Background
I live in a property with 4 well spaced out buildings that I would like to protectand and monitor, a house, two barns and a static caravan.

My goal is to build an alarm system for each building around the following objectives:

- [x] Built around the well supported ESP32 and Esphome
- [x] Off the shelf parts
- [x] Inexpensive to build
- [x] Easy to extend
- [x] Easy to program (Esphome)
- [x] Works independanly but can also be integrated with Home Assistant
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
