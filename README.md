# ⚡ 100% Software Zero-Export for Growatt MIC & Marstek Venus (NO EEPROM Wear)

*Read the full detailed step-by-step guide in German: [🇩🇪 README.de.md](README.de.md)*

## ⚠️ The Problem: EEPROM Flash Memory Destruction
Standard Modbus zero-export scripts constantly write the power limit to the inverter's holding registers. Doing this every second will physically destroy the EEPROM flash memory of your Growatt inverter within weeks (resulting in Error 417).

## 💡 The Solution: SDM630 Emulation
This Home Assistant & Node-RED flow emulates an Eastron SDM630 smart meter on Modbus ID 2. We only send volatile meter data to the inverter's RAM. The inverter regulates itself naturally based on the simulated grid data. **Zero hardware wear, 100% safe.**


## ✨ Key Features
* **Smart Charging:** Charges the Marstek battery with full power (1500W) up to 100% SoC, then automatically switches to true Zero-Export to cover your house load.
* **ID 1 Bug Fix:** Filters out the notorious Modbus communication bug.
* **Power Smoothing:** Mathematical smoothing to prevent inverter oscillation during sudden load changes (e.g., fridge compressors).
* **Hardware Saver:** Replaces expensive physical smart meters while protecting your inverter.

## 📋 Requirements
* Home Assistant with Node-RED Add-on
* USB-RS485 Dongle connected to the inverter's Modbus port
* Smart Meter reading your house load (e.g., Shelly Pro 3EM)

## 🇺🇸 Note for US & Canada Users (120V/240V Split-Phase)
If you are using a split-phase grid, your home energy monitor will provide separate readings for L1 and L2. 
To use this flow, simply create a Template Sensor in Home Assistant that adds the wattage of both phases together (`L1 + L2 = Total Grid Consumption`). Feed this combined total sensor into the Node-RED flow as your main input.

## 🛠️ Quick Installation
1. Import the provided `flow.json` into Node-RED.
2. Adjust your specific MQTT/Modbus nodes and sensor entities to match your Home Assistant setup.
3. *(For the detailed step-by-step setup, refer to the [German documentation](README.de.md) and use your browser's translation feature).*

## 🤝 Support the Project
This project took countless hours of reverse-engineering and testing to prevent hardware damage and build a perfectly smoothed zero-export logic. If it saved your inverter's EEPROM, helped your solar setup, or saved you from buying an expensive hardware smart meter, I’d be incredibly grateful for a small tip to keep the development going!

<a href="https://www.buymeacoffee.com/varan81" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 50px !important;width: 200px !important;" ></a>

---
*Happy tinkering and enjoy your free solar energy! ☀️*
