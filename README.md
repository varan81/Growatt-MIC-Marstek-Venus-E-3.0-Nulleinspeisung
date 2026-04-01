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

## 📅 Updates / Changelog

* **01.04.2026:** * **Anti-Oscillation Update (Flow Smoothing):** The Node-RED flow (`flow.json`) has been massively optimized to prevent the inverter from oscillating ("ping-pong effect") during rapid load changes in the house. The *"Echtzeit 3-Phasen Mathematik"* node now includes an integrated moving average (`alpha = 0.9`) to gently smooth out hard spikes. This heavily stabilizes the zero-export control without noticeably affecting the response time.

* **18.03.2026:** * **New Visualizations:** Added screenshots of the Home Assistant dashboard and the Node-RED flow structure for better traceability.
  * **New Chapter "Fine-Tuning":** Added an optional Home Assistant automation to prevent a slight oscillation of the control loop ("ping-pong effect") when the battery is full.

## 🏓 Fine-Tuning: Prevent Ping-Pong Effect at 100% Battery (Testing Phase)

When the battery reaches 100% SoC, the inverter and the battery might start to slightly regulate against each other. 
**Don't worry: This effect is not technically severe!** The system still operates safely and reliably. It just takes a few seconds longer for the zero-export to level out smoothly after major load changes in the house.

However, if you want a perfectly calm dashboard and immediate regulation, you can let Home Assistant play the "referee":
The battery is forced into `manual` mode at 100% (freezing its state) and leaves the zero-export completely to the Growatt. Only when the house draws more than 50W from the grid for over 30 seconds, the battery is woken up and set back to `anti_feed` mode.

To achieve this, add the following code to Home Assistant as a new automation (in YAML mode):

```yaml
alias: "Marstek Ping-Pong Protection (Manual Timer)"
description: >-
  Prevents the Growatt and Marstek from oscillating. Switches the battery to manual at 100% and wakes it back up in Anti-Feed mode only after 30 seconds of grid consumption.
mode: single
trigger:
  # Trigger 1: Battery is completely full
  - platform: numeric_state
    entity_id: sensor.marstek_venus_modbus_batterie_ladezustand
    above: 99
    id: battery_full
    
  # Trigger 2: House draws more than 50W for 30 seconds
  - platform: numeric_state
    entity_id: sensor.shellypro3em_leistung
    above: 50
    for:
      hours: 0
      minutes: 0
      seconds: 30
    id: power_needed

condition: []

action:
  - choose:
      # Action 1: Battery full -> Switch to manual mode
      - conditions:
          - condition: trigger
            id: battery_full
        sequence:
          - action: select.select_option
            target:
              entity_id: select.marstek_venus_modbus_modus
            data:
              option: "manual" 
              
      # Action 2: Power needed -> Revert to Anti-Feed
      - conditions:
          - condition: trigger
            id: power_needed
          # Small safeguard: Only wake up if the battery is not completely empty
          - condition: numeric_state
            entity_id: sensor.marstek_venus_modbus_batterie_ladezustand
            above: 5
        sequence:
          - action: select.select_option
            target:
              entity_id: select.marstek_venus_modbus_modus
            data:
              option: "anti_feed"
```



## ⚠️ Reality Check & Disclaimer (Grid Leakage)
While this software aims for a perfect zero-export, physics and Modbus polling intervals (usually 1-2 seconds) dictate that a mathematically perfect 100% zero-export is impossible in a grid-tied setup. 

When a heavy load (like an A/C, microwave, or kettle) suddenly turns off, the inverter needs a split second to adjust its output. During this brief window, a tiny amount of power *will* leak into the public grid (Grid Leakage / Spillover). 
If you are running a "guerrilla" or "stealth" solar setup and your local utility's smart meter is strictly configured to flag *any* backfeeding, please be aware of this. Use this software at your own risk!
