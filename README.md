# ⚡ 100% Software Zero-Export for Growatt MIC & Marstek Venus (NO EEPROM Wear)

*Read the full detailed step-by-step guide in German: [🇩🇪 README.de.md](README.de.md)*

## ⚠️ The Problem: EEPROM Flash Memory Destruction
Standard Modbus zero-export scripts constantly write the power limit to the inverter's holding registers. Doing this every second will physically destroy the EEPROM flash memory of your Growatt inverter within weeks (resulting in Error 417).

## 💡 The Solution: SDM630 Emulation
This Home Assistant & Node-RED flow emulates an Eastron SDM630 smart meter on Modbus ID 2. We only send volatile meter data to the inverter's RAM. The inverter regulates itself naturally based on the simulated grid data. **Zero hardware wear, 100% safe.**

## 📌 Current Project Status (April 2026)

**We are making excellent progress!** The core system runs absolutely stable and is already successfully in production use. 

Currently – also thanks to valuable feedback and testing from the community – we are fine-tuning the final details. This specifically includes the ongoing optimization of the new PI controller (response time and offset buffer). The main goal is to make the interaction between the network- and hardware-induced inertia of the Growatt inverter and the lightning-fast response time of AC battery systems (like the Marstek) even more harmonious and efficient.

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

* **04.04.2026:** * **🚀 PI Controller & Latency Compensation (Anti-Oscillation Update v2):** The simple smoothing from the previous update has been replaced by a real, intelligent PI controller with anti-windup protection in the *"Echtzeit 3-Phasen Mathematik"* node. This solves the problem of network latency (Shelly -> WLAN -> HA -> Node-RED), which previously threw the internal controller of the Growatt off balance during hard load changes (e.g., turning off the kettle).
  * **New Offset Buffer ("Shock Absorber"):** Using the new variable `mein_puffer` (default: 40W), the system now intentionally aims for a minimal grid consumption. This catches the overshooting when turning off large loads and effectively prevents unwanted export peaks into the power grid.
  * **Important Timing Updates:** The inject node of the Modbus server in Node-RED must be set to 0.25 seconds for a stable data bus. (Note: The `automations.yaml` in the repository has also been updated for the new and faster state trigger).
  * **New Flow:** The file `flow.json` has been completely replaced by the new, final version and can simply be imported as a whole.

* **01.04.2026:** * **Anti-Oscillation Update (Flow Smoothing):** The Node-RED flow (`flow.json`) has been massively optimized to prevent control oscillation ("ping-pong effect") of the inverter during fast load changes in the house. In the *"Echtzeit 3-Phasen Mathematik"* node, an integrated moving average (`alpha = 0.9`) now ensures that hard peaks are gently smoothed out. This extremely stabilizes the zero-export control without noticeably affecting the response time.

* **18.03.2026:** * **New Visualizations:** Added screenshots of the Home Assistant dashboard and the Node-RED flow structure for better comprehensibility.
  * **New Chapter "Fine-Tuning":** Added an optional Home Assistant automation to prevent a slight oscillation of the control ("ping-pong effect") when the battery is full.

## 🏓 Fine-Tuning: Adapting the System to Your Home

### 1. Adjusting the Controller in Node-RED (Real-Time Math)
Every home network has slightly different latency (delay). If your Growatt still overshoots slightly or ramps up too slowly during fast, hard load changes (e.g., when turning off the kettle), you can perfectly tailor the software to your setup. 

To do this, double-click the **"Echtzeit 3-Phasen Mathematik"** node in Node-RED. Right at the beginning of the code (`PHASE 2`), you will find these four parameters:

* **`mein_puffer = 40;` (The Shock Absorber):** Target value for grid consumption in watts. Forces the Growatt to intentionally draw a minimal amount of power from the grid. Prevents it from accidentally dropping into the negative (grid export) due to network latency when large loads are turned off. (Tip: 40W - 80W are safe values).
* **`Kp = 0.4;` (The Response Speed):** Determines how hard the controller reacts immediately to changes. Increase (e.g., `0.5`) if the Growatt ramps up too slowly. Decrease (e.g., `0.3`) if the system "jitters" too restlessly.
* **`Ki = 0.05;` (The Accuracy):** The fine adjustment over time. Ensures that the target buffer is actually reached exactly after a few seconds.
* **`maxIntegral = 1000;` (The Emergency Brake / Anti-Windup):** Prevents the controller from building up a huge "error memory" during long, high loads (e.g., oven) and taking ages to lower the power again when turned off.

---

### 2. Preventing Ping-Pong Effect at 100% Battery (Testing Phase)

When the battery reaches 100% SoC, it can happen that the inverter and the battery start to minimally regulate against each other. 
**Don't worry: This effect is not technically severe!** The system still works safely and reliably. It just takes a few seconds longer for the zero-export control to level out smoothly again during larger load changes in the house.

However, if you want a perfectly quiet dashboard and immediate regulation, you can hand over the role of "referee" to Home Assistant:
The battery is forced into `manual` mode at 100% (freezes) and leaves the zero-export control to the Growatt. Only when the house draws more than 50W from the grid for more than 30 seconds is the battery woken up again into `anti_feed` mode.

To do this, add this code in Home Assistant as a new automation (in YAML mode):

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
