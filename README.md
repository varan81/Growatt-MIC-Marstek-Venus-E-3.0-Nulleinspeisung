Growatt MIC & Marstek Venus: 100% Software Nulleinspeisung (Node-RED)Dieses Projekt ermöglicht eine echte, dynamische Nulleinspeisung für Growatt Wechselrichter (MIC-Serie) in Kombination mit dem Marstek Venus E 3.0 Batteriespeicher, OHNE den teuren Original-Zähler (Eastron/Chint) und OHNE Eingriffe in das EEPROM oder das Flashen von Mikrocontrollern.Die gesamte Steuerung läuft als reine Software-Lösung über Home Assistant und einen Node-RED Modbus-Emulator.🌟 Das Problem & Die LösungNormalerweise akzeptiert der Growatt nur Daten von direkt angeschlossenen, zertifizierten Smartmetern. Bisherige Workarounds erforderten oft das riskante Beschreiben des Wechselrichter-EEPROMs oder das Basteln von ESP32-Hardware.Dieser Lösungsansatz nutzt Node-RED, um dem Wechselrichter über einen einfachen USB-Adapter einen Eastron SDM630 V2 Zähler vorzuspielen.Der entscheidende Fix (Der Chint ID 1 Bug):Beim morgendlichen Booten scannt der Growatt das Netz und sucht hartnäckig auf der Modbus ID 1 nach einem Chint-Zähler. Antwortet man ihm dort mit Eastron-Float-Werten, entsteht ein Integer-Overflow (die App zeigt astronomische Verbräuche wie 214 Mio. kWh). Der hier bereitgestellte Node-RED Flow enthält einen strikten Filter, der Anfragen auf ID 1 ignoriert und den Growatt zwingt, dauerhaft und stabil auf ID 2 (Eastron) zu arbeiten.🛠️ Mein Hardware-SetupHome Assistant Server: HP t630 Thin ClientWechselrichter: Growatt MIC 1500TL-X👉 Growatt MIC 1500 auf Amazon ansehen (Affiliate-Link)Batteriespeicher: Marstek Venus E 3.0 (für maximale Stabilität per LAN ins Netzwerk eingebunden)👉 Marstek Speicher auf Amazon ansehen (Affiliate-Link)Haus-Smartmeter: Shelly Pro 3EM (ebenfalls per LAN verbunden)👉 Shelly Pro 3EM auf Amazon ansehen (Affiliate-Link)Modbus-Adapter: RS485 zu USB Stick👉 Empfohlener RS485 zu USB Adapter auf Amazon (Affiliate-Link)💡 Profi-Tipp zur Software: Für die perfekte Einbindung und Steuerung des Akkus in Home Assistant empfehle ich dringend die Marstek Venus Modbus Integration (verfügbar über HACS). Damit harmoniert dieses Setup am besten!🚀 Schritt-für-Schritt Anleitung1. Home Assistant: configuration.yamlFüge diesen Code in deine configuration.yaml ein, um die Datenmenge zu reduzieren und die Hardware zu schonen:input_number:
  fake_sdm_power:
    name: "Fake SDM Power"
    min: -15000
    max: 15000
    step: 0.1
    unit_of_measurement: "W"
    mode: box

recorder:
  exclude:
    entities:
      - input_number.fake_sdm_power
2. Home Assistant: Die Automatisierung (Sekundengenau)Erstelle eine neue Automatisierung und füge diesen YAML-Code ein. Diese Logik berechnet jede Sekunde den exakten Wert für den Growatt:alias: "SDM Emulator: Werte für Growatt berechnen (Sekundengenau)"
description: >-
  Regelt den Growatt via Node-RED. Nulleinspeisung bei manual Modus oder vollem
  Akku im anti_feed Modus.
triggers:
  - seconds: /1
    trigger: time_pattern
conditions: []
actions:
  - action: input_number.set_value
    target:
      entity_id: input_number.fake_sdm_power
    data:
      value: >-
        {# Variablen laden #}
        {% set akku_soc = states('sensor.marstek_venus_modbus_batterie_ladezustand') | float(0) %}
        {% set shelly_watt = states('sensor.shellypro3em_leistung') | float(0) %}
        {% set marstek_modus = states('select.marstek_venus_modbus_modus') %}

        {# 1. PRIORITÄT: Marstek steht auf 'manual' #}
        {% if marstek_modus == 'manual' %}
          {{ shelly_watt }}

        {# 2. PRIORITÄT: Marstek steht auf 'anti_feed' oder 'self_consumption' #}
        {% elif marstek_modus == 'anti_feed' or marstek_modus == 'self_consumption' %}
          {# Wenn Akku noch nicht ganz voll ist (bis 100%), gib Gas für die Ladung #}
          {% if akku_soc < 100 %}
            1500
          {% else %}
            {# Wenn Akku bei 100% ist, gehe auf Nulleinspeisung #}
            {{ shelly_watt }}
          {% endif %}

        {# STANDARDFALL #}
        {% else %}
          {{ shelly_watt }}
        {% endif %}
3. Das Marstek Steuerungs-Dashboard (Vollversion)Hier ist der vollständige Code für die High-End Dashboard-Karte (benötigt Mushroom Cards, Card-Mod und Stack-in-Card via HACS).<details><summary><b>KLICKE HIER FÜR DEN VOLLSTÄNDIGEN DASHBOARD-CODE</b></summary>type: custom:mod-card
column_span: 2
grid_options:
  columns: 24
card_mod:
  style: |
    ha-card {
      border: 1px solid rgba(255, 255, 255, 0.1) !important;
      box-shadow: 0 8px 32px 0 rgba(0, 0, 0, 0.37) !important;
      background: rgba(0, 0, 0, 0.18) !important;
      backdrop-filter: blur(16px);
      -webkit-backdrop-filter: blur(16px);
      border-radius: 26px !important;
      padding: 16px !important;
      color: rgba(255, 255, 255, 0.95) !important;
    }
card:
  type: custom:stack-in-card
  mode: vertical
  card_mod:
    style: |
      ha-card {
        border: none !important;
        box-shadow: none !important;
        background: transparent !important;
      }
  cards:
    - type: custom:button-card
      show_name: false
      show_state: false
      show_icon: false
      show_label: true
      label: |
        [[[
          return `
            <div style="display:flex;flex-direction:column;gap:8px;">
              <div style="font-size:1.55em; font-weight:800; border-bottom:1px solid rgba(255,255,255,0.08); padding-bottom:10px;">
                Batteriespeicher
              </div>
            </div>
          `;
        ]]]
      styles:
        card:
          - background: transparent
          - box-shadow: none
          - padding: 4px 6px 12px 6px
    - type: custom:button-card
      entity: sensor.marstek_venus_modbus_wechselrichter_status
      show_name: false
      show_state: false
      show_icon: false
      show_label: true
      label: |
        [[[
          const status = entity?.state ?? 'Unbekannt';
          const wlan = states['binary_sensor.marstek_venus_modbus_wlan_status']?.state === 'on' ? '🟢 Online' : '🔴 Offline';
          const mode = states['select.marstek_venus_modbus_modus']?.state ?? '—';
          const isNulleinspeisung = mode === 'anti_feed';
          const modeText = isNulleinspeisung ? 'Nulleispeisung aktiv' : (mode === 'manual' ? 'Manuell' : 'Trade Mode');
          const modeColor = isNulleinspeisung ? '#00ff88' : 'rgba(255,255,255,0.9)';
          return `
            <div style="display:flex; justify-content:space-between; align-items:center; font-size:1.0em; border-bottom:1px solid rgba(255,255,255,0.08); padding-bottom:10px;">
              <div>⚡ System-Status: <b>${status}</b></div>
              <div style="display:flex; align-items:center; gap:10px;">
                <div style="padding:4px 12px; border-radius:999px; background:rgba(255,255,255,0.06); border:1px solid rgba(255,255,255,0.1); color:${modeColor}; font-weight:800;">
                  ${modeText}
                </div>
                <div style="font-weight:800;">Verbindung: ${wlan}</div>
              </div>
            </div>
          `;
        ]]]
      styles:
        card:
          - background: transparent
          - box-shadow: none
          - padding: 4px 6px 12px 6px
    - type: custom:mushroom-select-card
      entity: select.marstek_venus_modbus_modus
      name: Steuerung / Modus
      icon: mdi:tune-variant
      layout: horizontal
    - type: custom:layout-card
      layout_type: grid
      layout:
        grid-template-columns: repeat(2, 1fr)
        gap: 8px
      cards:
        - type: custom:mushroom-number-card
          entity: number.marstek_venus_modbus_ladeleistung_einstellen
          name: Ladeleistung Set
          icon: mdi:battery-charging-high
          display_mode: slider
        - type: custom:mushroom-number-card
          entity: number.marstek_venus_modbus_maximale_entladeleistung
          name: Max Entladen
          icon: mdi:battery-arrow-up
          display_mode: slider
        - type: custom:mushroom-number-card
          entity: number.marstek_venus_modbus_maximale_ladeleistung
          name: Max Laden
          icon: mdi:battery-arrow-down
          display_mode: slider
        - type: custom:mushroom-number-card
          entity: number.marstek_venus_modbus_maximaler_soc
          name: Max SoC
          icon: mdi:battery-charging-100
          display_mode: slider
    - type: custom:layout-card
      layout_type: grid
      layout:
        grid-template-columns: repeat(4, minmax(0, 1fr))
        gap: 8px
      cards:
        - type: custom:button-card
          entity: sensor.marstek_batterie_laden
          name: >
            [[[ return parseFloat(entity.state) > 0 ? `Laden: ${entity.state}W` : "Laden"; ]]]
          icon: mdi:battery-charging
          styles:
            card:
              - height: 90px
              - border-radius: 20px
              - background: >
                  [[[ return parseFloat(entity.state) > 0 ? "linear-gradient(135deg,#00b894,#55efc4)" : "rgba(255,255,255,0.1)"; ]]]
        - type: custom:button-card
          entity: sensor.marstek_batterie_entladen
          name: >
            [[[ return parseFloat(entity.state) > 0 ? `Entladen: ${entity.state}W` : "Entladen"; ]]]
          icon: mdi:battery-charging
          styles:
            card:
              - height: 90px
              - border-radius: 20px
              - background: >
                  [[[ return parseFloat(entity.state) > 0 ? "linear-gradient(135deg,#e17055,#fab1a0)" : "rgba(255,255,255,0.1)"; ]]]
        - type: custom:button-card
          entity: sensor.marstek_venus_modbus_batterie_ladezustand
          name: |
            [[[ return `SoC: ${entity.state}%`; ]]]
          icon: mdi:battery-high
          styles:
            card:
              - height: 90px
              - border-radius: 20px
              - background: "linear-gradient(135deg, #0984e3, #74b9ff)"
        - type: custom:button-card
          entity: sensor.marstek_batterie_zeitprognose
          name: |
            [[[ 
              const val = entity.state;
              return (val === 'Standby' || val === '--') ? "Standby" : `Rest: ${val}`; 
            ]]]
          icon: mdi:clock-fast
          styles:
            card:
              - height: 90px
              - border-radius: 20px
              - background: |
                  [[[ return (entity.state === 'Standby' || entity.state === '--') ? "rgba(255,255,255,0.1)" : "linear-gradient(135deg, #0984e3, #74b9ff)"; ]]]
    - type: custom:layout-card
      layout_type: grid
      layout:
        grid-template-columns: repeat(4, 1fr)
        gap: 8px
      cards:
        - type: vertical-stack
          cards:
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_ac_leistung
              primary: Leistung AC
              secondary: "{{ states(entity) }} W"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_ac_strom
              primary: Strom AC
              secondary: "{{ states(entity) }} A"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_energie_laden_kwh
              primary: Geladen Tag
              secondary: "{{ states(entity) }} kWh"
        - type: vertical-stack
          cards:
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_batterieleistung
              primary: Leistung DC
              secondary: "{{ states(entity) }} W"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_batteriestrom
              primary: Strom DC
              secondary: "{{ states(entity) }} A"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_energie_entladen_kwh
              primary: Entladen Tag
              secondary: "{{ states(entity) }} kWh"
        - type: vertical-stack
          cards:
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_ac_spannung
              primary: Netz AC
              secondary: "{{ states(entity) }} V"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_batteriespannung
              primary: Spannung DC
              secondary: "{{ states(entity) }} V"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_batterie_gesamtenergie
              primary: Kapazität
              secondary: "{{ states(entity) }} kWh"
        - type: vertical-stack
          cards:
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_innentemperatur
              primary: Temp. Innen
              secondary: "{{ states(entity) }} °C"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_ac_frequenz
              primary: AC Frequenz
              secondary: "{{ states(entity) }} Hz"
            - type: custom:mushroom-template-card
              entity: sensor.marstek_venus_modbus_gesamt_roundtrip_effizienz
              primary: Effizienz
              secondary: "{{ states(entity) }} %"
</details>4. Node-RED: Der Emulator-FlowImportiere die in diesem Repository beiliegende flow.json Datei in dein Node-RED. Der Flow übernimmt die Kommunikation zwischen Home Assistant und dem RS485-Stick.☕ Support / KaffeekasseDieses Projekt hat Wochen an Analyse, Modbus-Debugging und Fehlersuche erfordert. Wenn dir dieser Code hunderte Euro für Original-Hardware erspart hat, freue ich mich riesig über eine kleine Unterstützung!
