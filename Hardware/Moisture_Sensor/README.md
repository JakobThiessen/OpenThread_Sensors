# Moisture Sensor (OpenThread) <img src="Moisture%20Sensor.png" alt="Moisture Sensor" width="48"> 

> Kapazitiver Bodenfeuchtesensor mit ESP32-H2 und Matter over Thread  
> **Version 1.0 — April 2026**

## Übersicht

Batteriebetriebener kapazitiver Bodenfeuchtesensor für den Einsatz im Smart Home. Die Kommunikation erfolgt drahtlos über **Matter over Thread** mittels eines **ESP32-H2 Supermini** Moduls.

Das Sensordesign misst die Bodenfeuchte kapazitiv über eine interdigitale Elektrodenstruktur auf der PCB-Sonde und nutzt einen **NE555** Timer zur Frequenzerzeugung. Die gemessene Frequenz wird vom ESP32-H2 ausgewertet und über das Thread-Netzwerk übermittelt.

## PCB

| Oberseite | Unterseite |
|-----------|------------|
| ![Top](Moisture%20Sensor_top.png) | ![Bottom](Moisture%20Sensor_bot.png) |

### Abmessungen

| Parameter | Wert |
|-----------|------|
| Gesamtlänge (inkl. Sonde) | ca. 90 mm |
| Breite Elektronikteil | ca. 28,4 mm |
| Breite Sonde | ca. 14,2 mm |

## Hauptkomponenten

| Bauteil | Beschreibung |
|---------|-------------|
| **ESP32-H2 Supermini** | Mikrocontroller mit IEEE 802.15.4 (Thread) |
| **NE555D** | Timer-IC für kapazitive Frequenzmessung |
| **BSS84** | P-Kanal MOSFET (Stromversorgungsschaltung) |
| **BSS123** | N-Kanal MOSFET (Sensor-Enable-Schaltung) |
| **BAS19** | Schaltdiode |
| **3,7 V LiPo** | Lithium-Polymer-Akku (auf der Rückseite montiert) |

## Signale

| Signal | Funktion |
|--------|----------|
| `SensorEN` | Aktiviert die Sensorschaltung (Stromsparen) |
| `SensorOut` | Frequenzausgang des NE555 → ESP32-H2 |
| `Ubat_adc` | Batteriespannungsmessung über ADC |
| `IO4`, `IO5` | Weitere GPIOs des ESP32-H2 |

## Funktionsprinzip

1. Der ESP32-H2 aktiviert die Sensorschaltung über `SensorEN` (MOSFET-Schaltstufe: Q1 BSS123 → Q2 BSS84).
2. Der NE555 erzeugt ein Rechtecksignal, dessen Frequenz von der Kapazität der Sonde abhängt.
3. Die Sonde nutzt interdigitale Kupferflächen — eine höhere Bodenfeuchte erhöht die Kapazität und senkt die Frequenz.
4. Der ESP32-H2 misst die Frequenz am `SensorOut`-Pin und berechnet den Feuchtigkeitswert.
5. Nach der Messung wird die Sensorschaltung abgeschaltet, um Strom zu sparen.
6. Die Batteriespannung wird über `Ubat_adc` überwacht (Spannungsteiler R1/R6: 2MΩ/4,7MΩ → ca. 0,55 µA Ruhestrom).
7. Die Messwerte werden per **Matter over Thread** an den Thread Border Router gesendet.

## Schaltungsanalyse — NE555 Astable

### Timing-Beschaltung

Der NE555 ist als **Astable-Multivibrator** konfiguriert:

| Parameter | Bauteil | Wert |
|-----------|---------|------|
| Ra (VCC → Discharge) | R2 | 330 kΩ |
| Rb (Discharge → Threshold/Trigger) | R3 | 1,6 kΩ |
| C (Timing-Kondensator) | C3 | 470 pF |
| C_probe (Sonden-Kapazität) | PCB-Elektroden | variabel |
| CV-Bypass (Pin 5) | C6 | 10 nF |
| Abblock-Kondensator | C4 | 100 nF |

### Frequenzformel (NE555 Astable)

$$f = \frac{1{,}44}{(R_A + 2 \cdot R_B) \cdot C_{total}}$$

Mit $R_A = 330\,\text{k}\Omega$, $R_B = 1{,}6\,\text{k}\Omega$:

$$R_A + 2 \cdot R_B = 333{,}2\,\text{k}\Omega$$

Die Gesamtkapazität setzt sich zusammen aus dem Festkondensator C3 und der variablen Sondenkapazität:

$$C_{total} = C_3 + C_{probe} = 470\,\text{pF} + C_{probe}$$

### Berechnete Frequenzen

| Zustand | C_probe (geschätzt) | C_total | Frequenz |
|---------|---------------------|---------|----------|
| Nur C3 (kein Boden) | 0 pF | 470 pF | **9,19 kHz** |
| Luft / trockene Sonde | ~10 pF | 480 pF | **9,00 kHz** |
| Trockener Boden | ~50 pF | 520 pF | **8,31 kHz** |
| Mäßig feucht | ~150 pF | 620 pF | **6,97 kHz** |
| Nasser Boden | ~300 pF | 770 pF | **5,61 kHz** |
| Gesättigt / Wasser | ~500 pF | 970 pF | **4,46 kHz** |

**Erwarteter Frequenzbereich: ca. 4,5 kHz – 9,2 kHz**

### Duty Cycle

$$D = \frac{R_A + R_B}{R_A + 2 \cdot R_B} = \frac{331{,}6\,\text{k}}{333{,}2\,\text{k}} \approx 99{,}5\,\%$$

Das Signal ist stark asymmetrisch (sehr kurze Low-Pulse). Der ESP32-H2 sollte die **Periodendauer** oder **Frequenz** messen (z. B. per Timer-Capture), nicht das Tastverhältnis.

### Bewertung (KI) — Kann das Design funktionieren?

**Grundsätzlich: Ja**, mit folgenden Anmerkungen:

| Aspekt | Bewertung |
|--------|-----------|
| Frequenzbereich 4,5–9,2 kHz | Sehr gut messbar mit ESP32-H2 Timer/Counter |
| Auflösung | Bei 12-bit Timer-Capture ausreichende Auflösung über den gesamten Bereich |
| Power-Gating (BSS84/BSS123) | Sinnvolles Design für Batteriebetrieb |
| Batterie-Monitoring (2M/4,7M Teiler) | Sehr stromsparend (~0,55 µA), gutes Design |
| Interdigitale PCB-Sonde | Bewährtes Prinzip für kapazitive Feuchtemessung |

**Wichtiger Hinweis — Versorgungsspannung:**

> ⚠️ Der klassische **NE555** (bipolar) ist für **min. 4,5 V** spezifiziert. Bei **3,3 V** Betriebsspannung kann er unzuverlässig arbeiten oder gar nicht anlaufen. Es wird empfohlen, eine **CMOS-555-Variante** zu verwenden (z. B. **LMC555**, **TLC555**, **ICM7555**), die ab **1,5 V** arbeitet und zudem weniger Strom verbraucht. Der Footprint (SOIC-8) ist kompatibel.

### Tipps zur Kalibrierung

- Den Sensor in **trockener Luft** und in **Wasser** messen → Frequenz-Endwerte bestimmen.
- Einen linearen oder polynomialen Zusammenhang zwischen Frequenz und Feuchte kalibrieren.
- Die Sondenkapazität hängt stark von Bodenart und Temperatur ab — eine Vor-Ort-Kalibrierung ist sinnvoll.

## Projektdateien

| Datei | Inhalt |
|-------|--------|
| `Moisture Sensor.kicad_sch` | Schaltplan |
| `Moisture Sensor.kicad_pcb` | PCB-Layout |
| `Moisture Sensor.kicad_pro` | KiCad-Projektdatei |
| `Moisture Sensor.step` | 3D-Modell der Platine |
| `Moisture Sensor.pdf` | Schaltplan als PDF |
| `lib/` | Bauteilbibliotheken (Symbole, Footprints, 3D-Modelle) |

## Voraussetzungen

- [KiCad 9.0](https://www.kicad.org/) oder neuer
- Thread Border Router (z. B. Apple HomePod, Google Nest Hub) für die Matter-Integration

## Lizenz

Dieses Projekt ist Teil des **OpenThread Sensors**-Projekts.
