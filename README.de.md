*For the English version, click here: [🇬🇧 README.md](README.md)*

# Growatt MIC & Marstek Venus: 100% Software Nulleinspeisung (Node-RED)

Dieses Projekt ermöglicht eine echte, dynamische Nulleinspeisung für **Growatt Wechselrichter (MIC-Serie)** in Kombination mit dem **Marstek Venus E 3.0** Batteriespeicher, **OHNE** den teuren Original-Zähler (Eastron/Chint) und **OHNE Eingriffe in das EEPROM** oder das Flashen von Mikrocontrollern.

## 📌 Aktueller Projektstatus (April 2026)

**Das Grundsystem läuft absolut stabil und ist im produktiven Einsatz.** Aktuell wird – auch durch Feedback aus der Community – noch an den Feinheiten gefeilt. Dazu gehört insbesondere die Optimierung des mathematischen Smoothings (Regelgeschwindigkeit), um das Zusammenspiel zwischen der Trägheit des Wechselrichters und der Reaktionsgeschwindigkeit von AC-Speichern (wie dem Marstek) noch harmonischer abzustimmen.

## ⚠️ Warum die Software-Lösung (Emulator) sicherer ist

Die meisten bisherigen Ansätze zur Nulleinspeisung versuchen, die Leistung des Growatt-Wechselrichters direkt über Modbus-Register zu drosseln. Das ist aus zwei Gründen brandgefährlich:

Begrenzte Schreibzyklen (EEPROM-Verschleiß):
Die internen Speicherzellen (EEPROM) eines Wechselrichters sind nicht dafür ausgelegt, sekündlich neue Werte zu empfangen. Sie haben eine technisch begrenzte Lebensdauer (oft nur 100.000 Schreibvorgänge). Wer die Leistung jede Sekunde direkt regelt, erreicht dieses Limit bereits nach ca. 28 Stunden. Danach ist der Speicher physisch defekt, und der Wechselrichter wird zum Totalschaden (Briefbeschwerer), da er seine Einstellungen nicht mehr speichern kann.

Garantieverlust & Risiko:
Direkte Eingriffe in die Leistungsregister werden vom Hersteller protokolliert. Bei einem Defekt lässt sich sofort nachweisen, dass das Gerät außerhalb der Spezifikationen betrieben wurde.

Mein Ansatz via SDM-Emulator:
Dieses Projekt nutzt einen völlig anderen Weg. Wir beschreiben keine internen Register. Wir simulieren stattdessen lediglich einen externen Stromzähler (Eastron SDM630). Der Growatt "denkt", er kommuniziert mit einem offiziell unterstützten Smartmeter und regelt seine Leistung intern über seine eigene, sichere Logik.

Kein Verschleiß: Da keine internen Speicherzellen überschrieben werden, bleibt die Hardware geschützt.

Maximale Sicherheit: Der Wechselrichter arbeitet innerhalb seiner vorgesehenen Betriebsparameter.

Die gesamte Steuerung läuft als reine Software-Lösung über Home Assistant und einen Node-RED Modbus-Emulator.

## 🌟 Das Problem & Die Lösung

Normalerweise akzeptiert der Growatt nur Daten von direkt angeschlossenen, zertifizierten Smartmetern. Bisherige Workarounds erforderten oft das riskante Beschreiben des Wechselrichter-EEPROMs oder das Basteln von ESP32-Hardware. 

Dieser Lösungsansatz nutzt **Node-RED**, um dem Wechselrichter über einen einfachen USB-Adapter einen **Eastron SDM630 V2** Zähler vorzuspielen.

---

## 🛠️ Mein Hardware-Setup

* **Home Assistant Server:** HP t630 Thin Client
* **Wechselrichter:** Growatt MIC 1500TL-X
  👉 **[Growatt MIC 1500 auf Amazon ansehen](https://amzn.to/4bcaEP3)
* **Batteriespeicher:** Marstek Venus E 3.0 (für maximale Stabilität per **LAN** ins Netzwerk eingebunden)
  👉 **[Marstek Speicher auf Amazon ansehen](https://amzn.to/4cPGQZN)
* **Haus-Smartmeter:** Shelly Pro 3EM (ebenfalls per **LAN** verbunden)
  👉 **[Shelly Pro 3EM auf Amazon ansehen](https://amzn.to/3PmjXTX)
* **Modbus-Adapter:** RS485 zu USB Stick
  👉 **[Empfohlener RS485 zu USB Adapter auf Amazon](https://amzn.to/4rwb8EE)

*💡 Profi-Tipp zur Software:* Für die perfekte Einbindung und Steuerung des Akkus in Home Assistant empfehle ich dringend die **Marstek Venus Modbus** Integration (verfügbar über HACS). Damit harmoniert dieses Setup am besten!

### 🔌 Verkabelung (Growatt zu USB-Stick)
Die Verbindung ist extrem simpel. Der Growatt hat unten einen "SYS" (System) Modbus-Anschluss.
* Verbinde **Pin 3** (RS485A) des Growatt-Steckers mit **A+** am USB-Stick.
* Verbinde **Pin 4** (RS485B) des Growatt-Steckers mit **B-** am USB-Stick.
* Also die gleichen Pins, die auch ein Eastron benutzt.
* Stecke den USB-Stick in deinen Home Assistant Server (HP t630).

---

## 🚀 Schritt-für-Schritt Anleitung

### 1. Home Assistant: `configuration.yaml`
Füge den folgenden Code in deine `configuration.yaml` ein (und starte Home Assistant danach neu):

```yaml
input_number:
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
```
💡 Wozu dient das?

1. "Gib Gas" (Lademodus): Das System schreibt hier 1500W rein, um den Akku mit maximaler Leistung zu laden.

2. "Nulleinspeisung - Akku voll": Das System schreibt hier exakt den aktuellen Hausverbrauch rein, um den Netzbezug auf Null zu regeln.
Node-RED greift diesen Wert im Sekundentakt ab und sendet ihn als simuliertes Smartmeter-Signal an den Wechselrichter.. Da sich dieser Wert alle paar Sekunden ändert, würde er extrem schnell die Datenbank von Home Assistant aufblähen. Der recorder-Eintrag schließt diesen virtuellen Sensor von der Speicherung aus. Das schont deine SSD/SD-Karte massiv und hält das System schnell.

### 2. Home Assistant: `Die Automatisierung`

Erstelle eine neue Automatisierung, klicke oben rechts auf die drei Punkte, wähle "Als YAML bearbeiten" und füge dies ein (passe die Sensoren-Namen an deine an):

```yaml
alias: "SDM Emulator: Werte für Growatt berechnen (Sekundengenau)"
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

        {% set akku_soc =
        states('sensor.marstek_venus_modbus_batterie_ladezustand') | float(0) %}

        {% set shelly_watt = states('sensor.shellypro3em_leistung') | float(0)
        %}

        {% set marstek_modus = states('select.marstek_venus_modbus_modus') %}


        {# 1. PRIORITÄT: Marstek steht auf 'manual' #}

        {% if marstek_modus == 'manual' %}
          {{ shelly_watt }}

        {# 2. PRIORITÄT: Marstek steht auf 'anti_feed' oder 'self_consumption'
        #}

        {% elif marstek_modus == 'anti_feed' or marstek_modus ==
        'self_consumption' %}
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
```
💡 Wozu dient das?

Die intelligente Ladesteuerung (Logik)
Diese Automatisierung ist das Herzstück der Anlage. Sie steuert dynamisch, wie viel Leistung der Growatt-Wechselrichter erbringen soll:

1. Lade-Priorität: Solange der Akku-Ladestand (SoC) unter 100% liegt, täuscht das System dem Wechselrichter einen hohen Verbrauch von 1500 Watt vor. Dadurch produziert der Growatt maximale Energie, um den Marstek-Speicher schnellstmöglich aufzuladen.

2. Nulleinspeisung: Sobald der Akku 100% erreicht hat oder wenn das System auf manuell (in der App oder dem Dashboard) gestellt wird, schaltet die Logik um. Jetzt wird exakt der aktuelle Hausverbrauch vom Shelly an den Wechselrichter gemeldet. Der Growatt regelt punktgenau auf diesen Wert herunter, sodass kein Strom ins Netz verschenkt wird.

### 3. Node-RED: Der Emulator-Flow

<img width="793" height="295" alt="Screenshot (236)" src="https://github.com/user-attachments/assets/452f594d-9fec-416c-a971-6576927c58ae" />

Importiere die in diesem Repository beiliegende flow.json Datei in dein Node-RED.
Der Flow liest alle 5 Sekunden die input_number.fake_sdm_power aus Home Assistant aus, bereitet die Daten mathematisch für 3 Phasen auf und antwortet dem Growatt in perfektem Eastron-Modbus-Hexadezimalcode.
Vergiss nicht, im Node-RED Flow den richtigen USB-Port für deinen RS485-Stick auszuwählen!

### 4. Das Marstek Steuerungs-Dashboard (Optional)

<img width="1110" height="765" alt="Screenshot (250)" src="https://github.com/user-attachments/assets/cb422a58-d301-4081-9477-9abad7363eef" />


Als Option ist hier der YAML-Code für eine "hochmoderne" ;), im Glas-Design gehaltene Home Assistant Dashboard-Karte, mit der du den Marstek-Akku überwachen und steuern kannst (benötigt Mushroom Cards, Card-Mod und Stack-in-Card via HACS). Tablet und Handy kompatibel

```yaml
type: custom:mod-card
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
      card_mod:
        style: |
          ha-card {
            background: rgba(0, 0, 0, 0.25) !important;
            border-radius: 18px !important;
            border: 1px solid rgba(255, 255, 255, 0.05) !important;
            box-shadow: none !important;
          }
    - type: custom:button-card
      show_name: false
      show_icon: false
      styles:
        card:
          - height: 2px
          - background: transparent
          - border: none
          - box-shadow: none
          - padding: 0
          - margin: 0
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
          layout: horizontal
          card_mod:
            style: |
              ha-card {
                background: rgba(0, 0, 0, 0.25) !important;
                border-radius: 18px !important;
                border: 1px solid rgba(255, 255, 255, 0.05) !important;
                box-shadow: none !important;
              }
        - type: custom:mushroom-number-card
          entity: number.marstek_venus_modbus_maximale_entladeleistung
          name: Max Entladen
          icon: mdi:battery-arrow-up
          display_mode: slider
          layout: horizontal
          card_mod:
            style: |
              ha-card {
                background: rgba(0, 0, 0, 0.25) !important;
                border-radius: 18px !important;
                border: 1px solid rgba(255, 255, 255, 0.05) !important;
                box-shadow: none !important;
              }
        - type: custom:mushroom-number-card
          entity: number.marstek_venus_modbus_maximale_ladeleistung
          name: Max Laden
          icon: mdi:battery-arrow-down
          display_mode: slider
          layout: horizontal
          card_mod:
            style: |
              ha-card {
                background: rgba(0, 0, 0, 0.25) !important;
                border-radius: 18px !important;
                border: 1px solid rgba(255, 255, 255, 0.05) !important;
                box-shadow: none !important;
              }
        - type: custom:mushroom-number-card
          entity: number.marstek_venus_modbus_maximaler_soc
          name: Max SoC
          icon: mdi:battery-charging-100
          display_mode: slider
          layout: horizontal
          card_mod:
            style: |
              ha-card {
                background: rgba(0, 0, 0, 0.25) !important;
                border-radius: 18px !important;
                border: 1px solid rgba(255, 255, 255, 0.05) !important;
                box-shadow: none !important;
              }
    - type: custom:button-card
      show_name: false
      show_icon: false
      styles:
        card:
          - height: 2px
          - background: transparent
          - border: none
          - box-shadow: none
          - padding: 0
          - margin: 0
    - type: custom:layout-card
      layout_type: grid
      layout:
        grid-template-columns: repeat(4, minmax(0, 1fr))
        gap: 8px
      cards:
        - type: custom:button-card
          entity: sensor.marstek_batterie_laden
          name: >
            [[[ return parseFloat(entity.state) > 0 ? `Laden: ${entity.state}W`
            : "Laden"; ]]]
          icon: mdi:battery-charging
          styles:
            card:
              - height: 90px
              - border-radius: 20px
              - background: >
                  [[[ return parseFloat(entity.state) > 0 ?
                  "linear-gradient(135deg,#00b894,#55efc4)" :
                  "rgba(255,255,255,0.1)"; ]]]
        - type: custom:button-card
          entity: sensor.marstek_batterie_entladen
          name: >
            [[[ return parseFloat(entity.state) > 0 ? `Entladen:
            ${entity.state}W` : "Entladen"; ]]]
          icon: mdi:battery-charging
          styles:
            card:
              - height: 90px
              - border-radius: 20px
              - background: >
                  [[[ return parseFloat(entity.state) > 0 ?
                  "linear-gradient(135deg,#e17055,#fab1a0)" :
                  "rgba(255,255,255,0.1)"; ]]]
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
                  [[[ 
                    return (entity.state === 'Standby' || entity.state === '--') 
                      ? "rgba(255,255,255,0.1)" 
                      : "linear-gradient(135deg, #0984e3, #74b9ff)"; 
                  ]]]
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
              icon: mdi:sine-wave
              icon_color: purple
              card_mod:
                style: >
                  ha-card { background: rgba(0,0,0,0.3) !important;
                  border-radius: 1
```
### Schritt 5 (Optionales Feintuning): siehe weiter unten bei den Updates

Hui das war lang, sorry dafür ;) Hab ich noch was vergessen? Ach ja der

☕ Support / Die Kaffeekasse

  Angefangen hat alles mit einem optimistischen "Das kann doch nicht so schwer sein..." – Wochen später sitze ich hier mit rauchenden Modbus-Registern und einer Menge grauer Haare mehr.

  Es war ein echter Kampf gegen die Growatt-Logik:

  Den berüchtigten ID 1 Bug (astronomische Fehlwerte) besiegt.

  Den nervigen Backflow-Fail und die 401-Abschaltungen eliminiert.

  Das hektische An- und Ausschalten bei jeder kleinen Laständerung durch eine glatte mathematische Regelung ersetzt.

  Wenn dir dieser Code den Kauf des Original-Zählers erspart hat – und vor allem die wochenlange Fehlersuche, die ich für dich schon erledigt habe – freue ich mich riesig über eine kleine Unterstützung für die nächste   Tasse Kaffee! Meine Nerven und mein Koffein-Spiegel danken es dir!

  [![Buy Me A Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-FFDD00?style=for-the-badge&logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/varan81)


## 📅 Updates / Changelog

* **01.04.2026:** * **Anti-Schwingungs-Update (Flow Smoothing):** Der Node-RED Flow (`flow.json`) wurde massiv optimiert, um ein Regelschwingen (Oszillieren / "Ping-Pong-Effekt") des Wechselrichters bei schnellen Lastwechseln im Haus zu verhindern. Im Node *"Echtzeit 3-Phasen Mathematik"* sorgt nun ein integrierter gleitender Mittelwert (`alpha = 0.9`) dafür, dass harte Spitzen sanft geglättet werden. Dies stabilisiert die Nulleinspeisung extrem, ohne die Reaktionszeit spürbar zu beeinträchtigen.

* **18.03.2026:** * **Neue Visualisierungen:** Screenshots des Home Assistant Dashboards und der Node-RED Flow-Struktur zur besseren Nachvollziehbarkeit hinzugefügt.
  * **Neues Kapitel "Feintuning":** Optionale Home Assistant Automatisierung ergänzt, um bei vollem Akku ein leichtes Aufschwingen der Regelung ("Ping-Pong-Effekt") zu verhindern.

## 🏓 Feintuning: Ping-Pong Effekt bei 100% Akku verhindern (Testphase)

Wenn die Batterie 100% SoC erreicht, kann es passieren, dass der Wechselrichter und die Batterie anfangen, minimal gegeneinander zu regeln. 
**Keine Sorge: Dieser Effekt ist technisch nicht gravierend!** Das System funktioniert trotzdem sicher und zuverlässig. Es braucht bei größeren Lastwechseln im Haus lediglich ein paar Sekunden länger, bis sich die Nulleinspeisung wieder sauber eingepegelt hat.

Wer jedoch ein perfekt ruhiges Dashboard und eine sofortige Ausregelung möchte, kann Home Assistant die Rolle des "Schiedsrichters" übergeben:
Die Batterie wird bei 100% in den `manual` Modus gezwungen (friert ein) und überlässt dem Growatt die Nulleinspeisung. Erst wenn das Haus für mehr als 30 Sekunden über 50W aus dem Netz zieht, wird die Batterie wieder in den `anti_feed` Modus aufgeweckt.

Füge dafür diesen Code in Home Assistant als neue Automatisierung (im YAML-Modus) ein:
```yaml
alias: "Marstek Ping-Pong Schutz (Manual Timer)"
description: >-
  Verhindert das Aufschwingen von Growatt und Marstek. Schaltet den Akku bei 100% auf manuell und weckt ihn erst nach 30 Sekunden Netzbezug wieder im Anti-Feed Modus auf.
mode: single
trigger:
  # Auslöser 1: Akku ist knallvoll
  - platform: numeric_state
    entity_id: sensor.marstek_venus_modbus_batterie_ladezustand
    above: 99
    id: akku_voll
    
  # Auslöser 2: Haus zieht mehr als 50W für 30 Sekunden
  - platform: numeric_state
    entity_id: sensor.shellypro3em_leistung
    above: 50
    for:
      hours: 0
      minutes: 0
      seconds: 30
    id: strom_wird_gebraucht

condition: []

action:
  - choose:
      # Aktion 1: Akku voll -> Ab in den manuellen Modus
      - conditions:
          - condition: trigger
            id: akku_voll
        sequence:
          - action: select.select_option
            target:
              entity_id: select.marstek_venus_modbus_modus
            data:
              option: "manual" 
              
      # Aktion 2: Strom wird gebraucht -> Wieder auf Anti-Feed
      - conditions:
          - condition: trigger
            id: strom_wird_gebraucht
          # Kleine Sicherheit: Nur wecken, wenn der Akku nicht komplett leer ist
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


  ⚖️ Haftungsausschluss (Disclaimer)

Dieses Projekt ist im Rahmen einer privaten Installation entstanden und dient rein zu Dokumentations- und Informationszwecken.

    Nutzung auf eigene Gefahr: Die Umsetzung der hier beschriebenen Inhalte erfolgt ausschließlich auf eigenes Risiko. Ich übernehme keinerlei Haftung für Schäden an Hardware (Wechselrichter, Batterien, etc.), Software oder für Folgeschäden.

    Keine Garantie: Es gibt keine Gewährleistung auf Richtigkeit, Vollständigkeit oder Funktionalität der Skripte.

    Elektrotechnik: Arbeiten am 230V-Netz dürfen in Deutschland nur von zertifiziertem Fachpersonal durchgeführt werden. Achte stets auf die Einhaltung deiner lokalen Vorschriften und Sicherheitsbestimmungen.

Kurz gesagt: Bei mir läuft das System seit Wochen stabil und schont die Hardware – aber was du mit deinem Wechselrichter machst, ist deine Entscheidung! 😉
