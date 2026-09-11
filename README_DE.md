# Mitsubishi + Home Assistant: externe Raumtemperatur

> **Update 11.09.2026:** Für die lokale Integration `pymitsubishi/homeassistant-mitsubishi` ist **Remote Temperature jetzt der empfohlene Weg**. Wenn diese Funktion mit Adapter und Klimagerät funktioniert, wird die bisherige Offset-/Stufenlogik für die Hauptregelung nicht mehr benötigt.

## Die vier Lösungswege

### 1. Home Assistant simuliert AUTO und wählt HEAT oder COOL

Home Assistant entscheidet anhand eines oder mehrerer externer Raumfühler selbst, ob die Mitsubishi in `HEAT` oder `COOL` laufen soll.

Diese Variante ist nur sinnvoll, wenn der native Mitsubishi-AUTO-Modus nicht gewünscht oder nicht brauchbar ist.

Typische ältere Dateien:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`

**Status:** Legacy / Fallback.

---

### 2. MELCloud Home + echter Mitsubishi-AUTO + Sollwert-Offset

Mitsubishi `AUTO` bleibt echtes `AUTO`. Home Assistant schaltet nicht zwischen HEAT und COOL um, sondern korrigiert nur den Geräte-Sollwert anhand der Abweichung zwischen externer Raumtemperatur und Wunschtemperatur.

Hauptdatei:

- `melcloud_home_external_temperature_control_public.yaml`

**Status:** weiterhin sinnvoll, wenn MELCloud Home bewusst der Primärweg bleiben soll.

---

### 3. Lokale `pymitsubishi`-Integration + echter Mitsubishi-AUTO + Sollwert-Offset

Die Steuerung läuft lokal über den Mitsubishi-Adapter, aber Home Assistant verwendet weiterhin unsere bisherige Kompensationslogik:

```text
externe Raumtemperatur -> Abweichung -> Offset / 0,5-°C-Stufe -> künstlicher Mitsubishi-Sollwert
```

AUTO bleibt dabei echtes Mitsubishi-AUTO.

**Status:** Legacy/Fallback für lokale Installationen, bei denen Remote Temperature nicht verfügbar oder nicht zuverlässig nutzbar ist.

---

### 4. Lokale `pymitsubishi`-Integration + Remote Temperature — empfohlen

Aktuell bevorzugte Architektur:

```text
Home Assistant -> lokales LAN/WLAN -> MAC-577IF2-E -> Mitsubishi
```

Einrichtung:

1. **Experimental Features** aktivieren
2. unter **External Temperature Sensor** den echten Raumfühler oder einen HA-Durchschnittssensor auswählen
3. **Temperature Source** auf `Remote` stellen
4. als Mitsubishi-Sollwert direkt die echte Wunschtemperatur verwenden
5. alte Offset-/Stufenregelung für die Hauptklima deaktivieren

Das Prinzip ist jetzt:

```text
externer Raumfühler -> Mitsubishi Remote Temperature
Wunschtemperatur    -> Mitsubishi-Sollwert
```

Keine künstliche +0,5 / +1,0 / +1,5 °C Sollwertkorrektur mehr, wenn Remote Temperature zuverlässig funktioniert.

Aktuelle Anleitung:

**[`REMOTE_TEMPERATURE_RECOMMENDED.md`](REMOTE_TEMPERATURE_RECOMMENDED.md)**

MELCloud Home kann parallel als manueller Fallback bestehen bleiben.

---

## Was von der bisherigen Arbeit weiter wichtig bleibt

Remote Temperature ersetzt nur den **Temperatur-Kompensationsmotor der Hauptklima**. Weiterhin nützlich bleiben:

- lokale MAC-577-Steuerung
- MELCloud-Home-Fallback
- `hvac_action`
- Zusatzheizungslogik
- Winterreserve
- Urlaubs-/Sicherheitslogik
- Wunschtemperatur-Helfer
- Sensor-Durchschnitt
- Lamellensteuerung
- Neustart-/Fallback-Sicherheit

Alte lokale Offset-/Stufen-Dokumente bleiben als **Archiv/Legacy und Rückfallweg** erhalten.

## Wichtig

Nicht zwei vollständige Hauptregelungen gleichzeitig gegen dasselbe Klimagerät laufen lassen.

Vor Änderungen an Integration, Entity-IDs oder Automationen immer ein Home-Assistant-Backup erstellen.
