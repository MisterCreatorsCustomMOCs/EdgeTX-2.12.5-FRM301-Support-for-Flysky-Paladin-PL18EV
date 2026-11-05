# EdgeTX 2.12.5 – FRM301 Support for Flysky Paladin PL18EV

**English summary:** This package adds support for the **FlySky FRM301** RF module (the module that ships with the Paladin PL18EV) to EdgeTX 2.12.5, used as an external AFHDS3 module just like the FRM303. Binding, two-way telemetry, channel output and automatic reconnect after a restart all work. The receiver ID is stored in the model file. The package also includes a fix for the 3-state stick buttons (EXT3/EXT4), a reworked charging logic modelled on FlySkyOS, and the ready-to-flash custom build **MC 1.2**. The changes are being prepared as pull requests for the official EdgeTX `main` branch and should land there soon. Until then, this package is the way to use them on 2.12.5.

---

> **Hinweis:** Diese Änderungen werden demnächst als Pull Request in den offiziellen EdgeTX-Branch (`main`, kommende Version 3.0) eingebracht. Die Fassungen für `main` liegen im Ordner `patches-main/`. Sobald sie offiziell übernommen sind, braucht man diese Patches nicht mehr.

## Inhalt des Ordners

| Datei / Ordner | Beschreibung |
|---|---|
| `EdgeTX-2.12.5-PL18EV-MC1.2.bin` | Fertige Firmware „MC 1.2“ (MisterCreators Custom MOCs) für die Paladin PL18EV, enthält alle Patches unten |
| `patches/01-pl18ev-mc-custom.patch` | Basis des MC-Builds: internes Crossfire am Bluetooth-Port, Lua `usbConnected()`/`getChargeState()`, Ladebildschirm, Versionsanzeige |
| `patches/02-pl18ev-frm301.patch` | **FRM301-Unterstützung** als externes AFHDS3-Modul |
| `patches/03-pl18ev-stickbuttons.patch` | Knüppeltasten EXT3/EXT4: Ruhe 0 %, oben +100 %, unten −100 % |
| `patches/04-pl18ev-charging.patch` | Ladelogik wie in FlySkyOS (Ladebildschirm bei ausgeschalteter Funke) |
| `patches-main/` | FRM301 und Knüppeltasten als saubere Commits für EdgeTX `main` (3.0), für den Pull Request |

Die Patches unter `patches/` gehören auf **EdgeTX v2.12.5** und werden in der Nummernfolge angewendet. 02 und 04 setzen 01 voraus. 03 (Knüppeltasten) geht auch allein auf ein unverändertes 2.12.5.

```
git clone --recursive -b v2.12.5 https://github.com/EdgeTX/edgetx.git
cd edgetx
git apply ../patches/01-pl18ev-mc-custom.patch
git apply ../patches/02-pl18ev-frm301.patch
git apply ../patches/03-pl18ev-stickbuttons.patch
git apply ../patches/04-pl18ev-charging.patch
mkdir build && cd build
cmake -DPCB=PL18 -DPCBREV=PL18EV -DPL18EV_INTERNAL_CRSF=ON -DMC_VERSION=1.2 -DCMAKE_BUILD_TYPE=Release ..
make firmware
```

Getestet mit arm-none-eabi-gcc 14.2.1 (xPack). Für die YAML-Tabellen braucht man Python mit `clang` (libclang 18) und `asciitree`.

---

## Firmware MC 1.2 mit EdgeTX Buddy flashen

**Vorher:**
- Akku voll laden.
- **SD-Karte bzw. internen Speicher sichern** (Ordner `MODELS`, `RADIO`, `SCRIPTS`, `WIDGETS`).
- Wer noch FlySkyOS drauf hat: zuerst das FlySky-Original sichern (siehe EdgeTX-Anleitung „Flashing EdgeTX to Flysky PL18 or Paladin EV“, Abschnitt „Backing up Flysky Bootloader“, ca. 2 MB). Ohne diese Sicherung kommt man nicht mehr zu FlySkyOS zurück.

**Schritte:**
1. Funke **ausschalten**.
2. Am linken Griff die Gummiauflage abziehen. Darunter sitzen zwei Taster: **oben DFU**, unten Not-Aus.
3. USB-Kabel nur an der Funke einstecken, Funke flach hinlegen.
4. **DFU-Taster gedrückt halten** und dabei das andere Kabelende in den PC stecken, dann loslassen. Die Funke meldet sich als DFU-Gerät (Display bleibt dunkel).
5. Im Browser **Chrome oder Edge** die Seite **https://buddy.edgetx.org** öffnen.
6. Unter **Flash** statt einer Version aus der Liste die **lokale Firmware-Datei** wählen (Datei-Upload) und `EdgeTX-2.12.5-PL18EV-MC1.2.bin` auswählen.
7. **„Flash via USB“** klicken, im Browser-Dialog das DFU-Gerät (STM32 BOOTLOADER) auswählen und warten (ca. 2 Minuten). Kabel währenddessen nicht ziehen.
8. Nach „Fertig“ das USB-Kabel abziehen und den unteren versteckten Taster kurz drücken (schaltet die Funke aus).
9. Funke normal einschalten. Unter *System → Version* steht jetzt **„VERS: MC 1.2 (based on EdgeTX 2.12.5)“**.

**Alternative ohne PC-Treiber (EdgeTX-Bootloader):** Die `.bin` in den Ordner `FIRMWARE` auf der SD-Karte/dem internen Speicher kopieren. Funke aus, beide horizontalen Trimmer (TR7 und TR8) zueinander drücken und beide Power-Tasten drücken. Im Bootloader „Write Firmware“ wählen und die Datei auswählen.

Wer schon MC 1.0 oder 1.1 hat, kann direkt drüberflashen. Modelle und Einstellungen bleiben erhalten, das FRM301 muss nicht neu gebunden werden.

---

## FRM301 benutzen

1. *Modell → Externes Modul → Typ* auf **AFHDS3** stellen.
2. **RF module** auf *Auto* lassen (erkennt FRM301/FRM303 selbst) oder fest *FRM301* wählen.
3. **Two way** auf *Ja*, wenn Telemetrie gewünscht ist.
4. Empfänger in den Bind-Modus bringen, dann **Bind** drücken. Die Funke startet das Modul kurz neu (das ist gewollt), das Modul blinkt schnell grün und verbindet sich.
5. Unter *Optionen* den Bus-Typ (z. B. **SBUS**) und PWM einstellen. Die Werte werden im Modell gespeichert und an das Modul übertragen.

Empfänger-ID und Moduloptionen stehen in der Modelldatei. Nach einem Neustart verbindet sich der Empfänger wieder von selbst. Getestet mit dem **FTr16S**.

---

## Änderungen im Detail

### 01 – MC-Basis (`pl18ev-mc-custom.patch`)

Wird nur mit `-DPL18EV_INTERNAL_CRSF=ON` aktiv, dann ist `MC_CUSTOM_BUILD` definiert.

| Datei | Änderung |
|---|---|
| `targets/pl18/CMakeLists.txt` | Neue Optionen `PL18EV_INTERNAL_CRSF` (internes CRSF-Modul an den Bluetooth-Pads, USART3), `PL18EV_INTMODULE_PWR_PI8` (Stromversorgung des internen Moduls über PI.08/BT_EN), `MC_VERSION`, `MC_CHARGE_DIAG` (Diagnose, standardmäßig aus) |
| `targets/pl18/hal.h`, `board.h` | Internes Modul an TX3/RX3 (USART3), Ein/Aus über PI.08. UART7 bleibt frei. |
| `targets/pl18/battery_driver.cpp/.h` | Ladebildschirm mit Akkuspannung und Prozent, LED-Leiste, Anzeige 10 s nach Berührung/Trimmer/Power bzw. 30 s nach dem Einstecken, Laden auch bei laufender Funke (`mcChargeTick`), Abfragen `mcChargerConnected/mcGetChargeState/mcGetChargeSource` |
| `targets/pl18/board.cpp` | Ladebildschirm: ADC und Akku-Kalibrierung laden, Trimmer und echter Touch-Kontakt (3× über I2C geprüft) wecken das Display |
| `targets/pl18/touch_driver.cpp/.h` | `mcTouchHasContact()`: liest direkt, ob ein Finger aufliegt (der INT-Pin pulst beim Laden auch ohne Berührung) |
| `edgetx.cpp` | Ruft im 10-ms-Takt `mcChargeTick()` auf (Ladezustand bei laufender Funke) |
| `main.cpp` | Keine Akku-Warnung, solange ein Ladegerät angeschlossen ist |
| `lua/api_general.cpp` | Neue Lua-Funktionen `usbConnected()` und `getChargeState()` (Zustand 0/1/2 und Quelle USB/Wireless) |
| `gui/colorlcd/radio/radio_version.cpp` | Versionsseite zeigt „FW: FlySky Paladin PL18 EV / VERS: MC x.y (based on EdgeTX …)“. Die internen Versionsstempel bleiben unverändert, damit Companion und die SD-Prüfung weiter funktionieren. |

### 02 – FRM301 (`pl18ev-frm301.patch`)

Option `PL18EV_FRM301` (Standard: an) definiert `AFHDS3_FRM301_EXT`. Das FRM301 sitzt am selben Port wie das FRM303, spricht aber die FlySkyOS-Variante von AFHDS3 (Geräteadresse 2): Das Modul fragt seine Konfiguration selbst bei der Funke an, nutzt andere MODULE_MODE-Werte und speichert keine Bindungen.

| Datei | Änderung |
|---|---|
| `pulses/afhds3.cpp` | Hauptteil: automatische Erkennung FRM301/FRM303 (Adresswechsel, solange keine Antwort kommt). Beantwortet die Anfragen des Moduls (Config-Teile 1–3, „Host bereit?“, Empfängerblock) sofort aus dem Empfangspfad. Baut die Konfiguration wie FlySkyOS (PHY-Konstanten, Standard-Empfängerblock aus einem Mitschnitt, Two-way-Flag, Endbyte von Teil 2 mit automatischer Umschaltung für ältere Modul-Firmware). Bind wie FlySkyOS: Modul 300 ms stromlos, nach dem Hochfahren 1,5 s warten, Config als „Bind“ (Art 1) + `0x05` senden, alle 3 s wiederholen, bis das Modul „Binding/Verbunden“ meldet. Nach dem Verbinden das Empfänger-Kommando `0x7030`. Speichert den Empfängerblock (inkl. Empfänger-ID) und die Moduloptionen im Modell und spielt sie beim Start zurück. Kanaldaten laufend, Status etwa einmal pro Sekunde, Zykluszeit 2,5 ms. Keine Open-Protocol-Extras (virtueller Failsafe usw.) an das FRM301. |
| `pulses/afhds3_transport.cpp/.h` | Anfragen vom Modul (GET / SET_EXPECT_DATA / SET_EXPECT_ACK) erkennen und mit gleicher Frame-Nummer aus einem eigenen Puffer beantworten. Frame-Nummer läuft weiter (das Modul verwirft Wiederholungen). Adresse lesbar/setzbar. Diagnosezähler nur mit `FRM301_DEBUG`. |
| `pulses/afhds3_config.h`, `afhds3_module.h` | `isFRM301()`, größerer Sendepuffer (Config-Antworten bis 169 Byte) |
| `pulses/pulses.h`, `telemetry/telemetry.h` | Größere Puffer für die langen Antworten / den Empfängerblock (168 Byte) |
| `gui/colorlcd/module/afhds3_settings.cpp` | Auswahl „RF-Modul“ (Auto/FRM303/FRM301) und „Two way“ (Nein/Ja). SD-Log und Diagnose nur mit `FRM301_DEBUG`. |
| `gui/colorlcd/module/afhds3_options.cpp` | Titel zeigt FRM301 bzw. FRM303 |
| `main.cpp` | `afhds3::frmBackgroundTask()` im UI-Task: Speichern im Modell (der Mixer-Task hat dafür zu wenig Stack) |
| `datastructs_private.h`, `datastructs.h` | Neue Modellfelder `frm301Rx` (Empfängerblock als Hex, 337 Zeichen) und `frm301Cfg` (Optionen, 235 Zeichen) plus 2-Bit-Füllfeld, CHKSIZE angepasst |
| `storage/yaml/yaml_datastructs_pl18.cpp` | YAML-Einträge für die neuen Felder |
| `storage/yaml/yaml_tree_walker.cpp` | `copy_string`: Längenparameter von `uint8_t` auf `uint16_t`, sonst werden Texte über 255 Zeichen abgeschnitten |
| `targets/pl18/CMakeLists.txt` | Option `PL18EV_FRM301` |

Debug-Code (SD-Verkehrslog, Diagnosezeilen) ist enthalten, wird aber nur mit `-DFRM301_DEBUG` übersetzt. In der Fassung für `main` ist er komplett entfernt, und dort wird statt neuer Modellfelder die eingebaute `userData`-Ablage benutzt.

### 03 – Knüppeltasten (`pl18ev-stickbuttons.patch`)

| Datei | Änderung |
|---|---|
| `mixer.cpp` | Die Taster oben auf den Knüppeln (EXT3/EXT4) sind 3-Stufen-Widerstandstaster. Der kalibrierte Wert wird umgerechnet: Ruhe 0 %, oben +100 %, unten −100 %. Mit 50 ms Entprellung, weil das Signal auf dem Weg nach „oben“ kurz den „unten“-Pegel durchläuft. Ersetzt das Lua-Mixerscript `btn3p.lua`. |
| `util/hw_defs/pot_config.py` | Standardtyp von EXT3/EXT4 von `MULTIPOS` auf `POT_CENTER` |

Wichtig: Die Taster einmal kalibrieren (*System → Hardware → Kalibrieren*, dabei jeden Taster einmal nach oben und unten drücken). Ein vorhandenes `btn3p.lua` danach aus dem Modell entfernen, sonst wird doppelt umgerechnet.

### 04 – Ladelogik (`pl18ev-charging.patch`)

Nachgebaut aus FlySkyOS (Paladin EV Firmware 1.0.9, Lade-Task alle 100 ms):

| Datei | Änderung |
|---|---|
| `targets/pl18/battery_driver.cpp` | Lade-Pin (USB PB13, Wireless PI10) zählt erst nach 1 s gleichem Pegel. „Voll“ nur, wenn der Pin Ladeende meldet **und** die Akkuspannung (5 Messungen ohne Min/Max gemittelt) mindestens 4,15 V beträgt. Einmal „voll“ bleibt es bis zum Abstecken. Der Ladechip wird 97 s eingeschaltet und 3 s ausgeschaltet (startet den Ladezyklus neu, Pin wird in der Pause nicht ausgewertet). Es ist immer nur ein Lader aktiv, USB hat Vorrang. Wireless-Ladestrom wird nach 240 s hochgeschaltet und 60 s nach dem Abnehmen wieder runter. |

**Status:** noch in Erprobung. Rückmeldungen zum Verhalten am Ladekabel sind willkommen.

---

## Rechtliches

EdgeTX steht unter der GPLv2. Die Patches stehen unter derselben Lizenz. Die FRM301-Unterstützung wurde durch Mitschneiden der seriellen Verbindung zwischen Funke und Modul unter FlySkyOS sowie durch Lesen der Original-Firmware erarbeitet. Es ist kein FlySky-Code enthalten. Nutzung auf eigene Gefahr. Vor dem Fliegen/Fahren immer Reichweite und Failsafe testen.

MisterCreators Custom MOCs – https://mistercreator.de
