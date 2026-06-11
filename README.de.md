*For the English version, click here: [🇬🇧 README.md](README.md)*

# Growatt MIC & Marstek Venus: 100% Software Nulleinspeisung (Node-RED)

Dieses Projekt ermöglicht eine echte, dynamische Nulleinspeisung für **Growatt Wechselrichter (MIC-Serie)** in Kombination mit dem **Marstek Venus E 3.0** Batteriespeicher, **OHNE** den teuren Original-Zähler (Eastron/Chint) und **OHNE Eingriffe in das EEPROM** oder das Flashen von Mikrocontrollern.

## 📌 Aktueller Projektstatus (April 2026)

**Wir sind auf einem extrem guten Weg!** Das Grundsystem läuft absolut stabil und befindet sich bereits erfolgreich im produktiven Einsatz. 

Aktuell wird – auch durch das wertvolle Feedback und die Tests aus der Community – noch an den letzten Feinheiten gefeilt. Dazu gehört insbesondere die laufende Optimierung der neuen PI-Regelung (Reaktionsgeschwindigkeit und Offset-Puffer). Das große Ziel ist es, das Zusammenspiel zwischen der netzwerk- und hardwarebedingten Trägheit des Growatt-Wechselrichters und der blitzschnellen Reaktionszeit von AC-Speichern (wie dem Marstek) noch harmonischer und effizienter abzustimmen.

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
* Verbinde **Pin 7** (RS485A) des Growatt-Steckers mit **A+** am USB-Stick.
* Verbinde **Pin 8** (RS485B) des Growatt-Steckers mit **B-** am USB-Stick.
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
  - trigger: state
    entity_id: sensor.shellypro3em_leistung
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

<img width="809" height="330" alt="Screenshot (289)" src="https://github.com/user-attachments/assets/4175f6b6-b699-4316-bb6c-d5c6b0d3a038" />


---

## 📊 Das Marstek Steuerungs-Dashboard

> **⚡ Update Juni 2026:** Das Dashboard wurde von Grund auf neu entwickelt. Die alte Version (Mushroom-Slider, einfache Status-Anzeige) wurde durch ein vollständiges **Glassmorphism-Dashboard** mit dynamischen Balken, Animationen und Echtzeit-Visualisierungen ersetzt.

### ✨ Was ist neu?

| Feature | Alt | Neu |
|---|---|---|
| Leistungsbalken | ✗ | ✅ Dynamisch grün/rot je nach Richtung |
| Spannungs-/Frequenzanzeige | Nur Wert | ✅ Zonenbalken mit Marker & Pfeil |
| Ladestandsanzeige | Text | ✅ SoC-Balken mit fließenden Punkten |
| Animationen | ✗ | ✅ Laden ⚡ / Entladen / Standby |
| Warnungen | ✗ | ✅ Kritisch <10%, Überhitzung >40°C |
| Energieübersicht | ✗ | ✅ Netz · PV · Haus im Header |
| Steuerung | Slider | ✅ Kompakte +/− Buttons |

### 🖼️ Screenshots

> <img width="1323" height="904" alt="Screenshot (481)" src="https://github.com/user-attachments/assets/76a8742a-ab92-47a8-a507-6e00911f33c2" />


### ✨ Features im Detail

- **Animierter Header** mit dynamischem Batterie-Icon (Laden ⚡ / Entladen / Standby), Online-Status und Echtzeit-Energieübersicht (Netzbezug · PV · Hausverbrauch)
- **4 Status-Buttons** (Laden, Entladen, SoC, Restzeit) mit animierten Icons — springendes ⚡ beim Laden, pulsierendes 🔋 beim Entladen
- **Dynamische Leistungsbalken** — grün beim Laden, rot beim Entladen — für Leistung AC/DC und Strom AC/DC
- **Zonenbalken mit Marker und Pfeil** für Netz-Spannung (200–260V) und AC-Frequenz (47–53Hz) — Warnung bei Abweichung vom Normbereich
- **Spannung DC** als Zonenbalken (44–58V) mit Marker bei 51.2V (LiFePO4 Nennspannung)
- **Temperaturwarnung** mit pulsierendem Ring bei >40°C Innentemperatur
- **Konfetti 🎉** bei SoC >95% und roter Blinkeffekt bei SoC <10%
- **SoC-Balken** mit fließenden Punkten beim Laden/Entladen
- **Vollzyklen-Anzeige**
- **Backup-Funktion** als Toggle-Button
- **Steuerung** per kompakten +/− Buttons: Ladeleistung, Max Laden, Max Entladen, Max SoC (je ±50W / ±5%)

### 🔧 Benötigte HACS-Komponenten

| Komponente | Typ |
|---|---|
| `button-card` | Frontend |
| `mushroom` | Frontend |
| `layout-card` | Frontend |
| `stack-in-card` | Frontend |
| `mod-card` | Frontend |

### 📋 Verwendete Entities

# Marstek Venus Modbus
sensor.marstek_venus_modbus_ac_leistung
sensor.marstek_venus_modbus_batterieleistung
sensor.marstek_venus_modbus_ac_strom
sensor.marstek_venus_modbus_batteriestrom
sensor.marstek_venus_modbus_batterie_ladezustand
sensor.marstek_venus_modbus_ac_spannung
sensor.marstek_venus_modbus_ac_frequenz
sensor.marstek_venus_modbus_batteriespannung
sensor.marstek_venus_modbus_innentemperatur
sensor.marstek_venus_modbus_gesamt_roundtrip_effizienz
sensor.marstek_venus_modbus_vollzyklen_berechnet
sensor.marstek_batterie_laden
sensor.marstek_batterie_entladen
sensor.marstek_energie_laden_kwh
sensor.marstek_energie_entladen_kwh
sensor.marstek_batterie_zeitprognose
binary_sensor.marstek_venus_modbus_wlan_status
select.marstek_venus_modbus_modus
switch.marstek_venus_modbus_backup_funktion
number.marstek_venus_modbus_ladeleistung_einstellen
number.marstek_venus_modbus_maximale_entladeleistung
number.marstek_venus_modbus_maximale_ladeleistung
number.marstek_venus_modbus_maximaler_soc

# Energieübersicht im Header
sensor.shellypro3em_leistung      # Netzbezug (Shelly Pro 3EM)
sensor.pv_power                   # PV-Leistung
sensor.haus_leistung_gesamt       # Hausverbrauch


### 📥 Installation

1. Datei `batterie_dashboard.yaml` herunterladen
2. In Home Assistant eine neue **Section** anlegen
3. Karte hinzufügen → **YAML-Editor** → Inhalt einfügen
4. Entity-IDs ggf. an eigene Installation anpassen

> **Hinweis:** Das Dashboard ist für das **Sections-Layout** in Home Assistant optimiert (`column_span: 24`). Die `sensor.shellypro3em_leistung`, `sensor.pv_power` und `sensor.haus_leistung_gesamt` Entities im Header können durch eigene Sensoren ersetzt oder entfernt werden.
>
> 
### Schritt 5 (Optionales Feintuning): siehe weiter unten bei den Updates

Hui das war lang, sorry dafür ;) Hab ich noch was vergessen? Ach ja der

☕ Support / Die Kaffeekasse

  Angefangen hat alles mit einem optimistischen "Das kann doch nicht so schwer sein..." – Wochen später sitze ich hier mit rauchenden Modbus-Registern und einer Menge grauer Haare mehr.

  Es war ein echter Kampf gegen die Growatt-Logik:

  Den berüchtigten ID 1 Bug (astronomische Fehlwerte) besiegt.

  Den nervigen Backflow-Fail und die 401-Abschaltungen eliminiert.

  Das hektische An- und Ausschalten bei jeder kleinen Laständerung durch eine glatte mathematische Regelung ersetzt.

  Wenn dir dieser Code den Kauf des Original-Zählers erspart hat – und vor allem die wochenlange Fehlersuche, die ich für dich schon erledigt habe – freue ich mich riesig über eine kleine Unterstützung für die nächste   Tasse Kaffee! Meine Nerven und mein Koffein-Spiegel danken es dir!

  <a href="https://www.buymeacoffee.com/varan81" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" style="height: 50px !important;width: 200px !important;" ></a>


## 📅 Updates / Changelog

* **04.04.2026:** * ** PI-Regler & Latenz-Kompensation (Anti-Schwingungs-Update v2):** Die einfache Glättung aus dem vorherigen Update wurde durch einen echten, intelligenten PI-Regler mit Anti-Windup-Schutz im Node *"Echtzeit 3-Phasen Mathematik"* ersetzt. Dies löst das Problem der Netzwerklatenz (Shelly -> WLAN -> HA -> Node-RED), die den internen Regler des Growatt bei harten Lastwechseln (z. B. Wasserkocher) bisher aus dem Tritt gebracht hat.
  * **Neuer Offset-Puffer ("Stoßdämpfer"):** Über die neue Variable `mein_puffer` (Standard: 40W) zielt das System nun absichtlich auf einen minimalen Netzbezug ab. Das fängt das Überschwingen beim Abschalten großer Lasten ab und verhindert effektiv ungewollte Einspeisungs-Spitzen in das Stromnetz.
  * **Wichtige Timing-Updates:** Der Inject-Node des Modbus-Servers in Node-RED muss für einen stabilen Datenbus auf 0,25 Sekunden gestellt werden. (Hinweis: Die `automations.yaml` wurde im Repository ebenfalls für den neuen und schnelleren State-Trigger aktualisiert).
  * **Neuer Flow:** Die Datei `flow.json` wurde komplett durch die neue, finale Version ersetzt und kann einfach im Ganzen neu importiert werden.

* **01.04.2026:** * **Anti-Schwingungs-Update (Flow Smoothing):** Der Node-RED Flow (`flow.json`) wurde massiv optimiert, um ein Regelschwingen (Oszillieren / "Ping-Pong-Effekt") des Wechselrichters bei schnellen Lastwechseln im Haus zu verhindern. Im Node *"Echtzeit 3-Phasen Mathematik"* sorgt nun ein integrierter gleitender Mittelwert (`alpha = 0.9`) dafür, dass harte Spitzen sanft geglättet werden. Dies stabilisiert die Nulleinspeisung extrem, ohne die Reaktionszeit spürbar zu beeinträchtigen.

* **18.03.2026:** * **Neue Visualisierungen:** Screenshots des Home Assistant Dashboards und der Node-RED Flow-Struktur zur besseren Nachvollziehbarkeit hinzugefügt.
  * **Neues Kapitel "Feintuning":** Optionale Home Assistant Automatisierung ergänzt, um bei vollem Akku ein leichtes Aufschwingen der Regelung ("Ping-Pong-Effekt") zu verhindern.

## 🏓 Feintuning: Das System an dein Zuhause anpassen

### 1. Den Regler in Node-RED einstellen (Echtzeit Mathematik)
Jedes Heimnetzwerk hat eine etwas andere Latenz (Verzögerung). Falls dein Growatt bei schnellen, harten Lastwechseln (z.B. beim Ausschalten des Wasserkochers) noch leicht überschwingt oder zu langsam hochfährt, kannst du die Software perfekt auf dein Setup abstimmen. 

Mache dazu in Node-RED einen Doppelklick auf den Node **"Echtzeit 3-Phasen Mathematik"**. Direkt am Anfang des Codes (`PHASE 2`) findest du diese vier Stellschrauben:

* **`mein_puffer = 40;` (Der Stoßdämpfer):** Zielwert für den Netzbezug in Watt. Zwingt den Growatt, absichtlich minimalen Strom aus dem Netz zu ziehen. Verhindert, dass er durch Netzwerklatenz beim Abschalten großer Lasten versehentlich ins Minus (Einspeisung) stürzt. (Tipp: 40W - 80W sind sichere Werte).
* **`Kp = 0.4;` (Die Reaktionsgeschwindigkeit):** Bestimmt, wie hart der Regler sofort auf Änderungen reagiert. Erhöhen (z. B. `0.5`), wenn der Growatt zu langsam hochfährt. Senken (z. B. `0.3`), wenn das System zu unruhig "zittert".
* **`Ki = 0.05;` (Die Genauigkeit):** Das feine Nachjustieren über die Zeit. Sorgt dafür, dass der Ziel-Puffer nach ein paar Sekunden auch wirklich exakt erreicht wird.
* **`maxIntegral = 1000;` (Die Notbremse / Anti-Windup):** Verhindert, dass sich der Regler bei langen, hohen Lasten (z. B. Backofen) einen riesigen "Fehler-Speicher" aufbaut und beim Ausschalten ewig braucht, um die Leistung wieder zu senken.

---

### 2. Ping-Pong Effekt bei 100% Akku verhindern (Testphase)

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
