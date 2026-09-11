# Mitsubishi + Home Assistant: externe Raumtemperatur

> **Update 11.09.2026:** Für die lokale Integration `pymitsubishi/homeassistant-mitsubishi` ist **Remote Temperature jetzt der empfohlene Weg**. Wenn diese Funktion mit Adapter und Klimagerät funktioniert, wird die bisherige Offset-/Stufenlogik für die Hauptregelung nicht mehr benötigt.

## Aktuell empfohlene Einrichtung

In der lokalen Mitsubishi-Integration:

1. **Experimental Features** aktivieren
2. unter **External Temperature Sensor** den echten Raumfühler oder einen HA-Durchschnittssensor auswählen
3. **Temperature Source** auf `Remote` stellen
4. als Mitsubishi-Sollwert direkt die echte Wunschtemperatur verwenden
5. alte Offset-/Stufenregelung für die Hauptklima deaktivieren

Aktuelle Anleitung:

**[`REMOTE_TEMPERATURE_RECOMMENDED.md`](REMOTE_TEMPERATURE_RECOMMENDED.md)**

Das Prinzip ist jetzt:

```text
externer Raumfühler -> Mitsubishi Remote Temperature
Wunschtemperatur    -> Mitsubishi-Sollwert
```

Keine künstliche +0,5 / +1,0 / +1,5 °C Sollwertkorrektur mehr, wenn Remote Temperature zuverlässig funktioniert.

---

## Die drei Lösungswege

### 1. Legacy / klassische MELCloud-Regelung

Die älteren externen Sensor-/Offset-Lösungen bleiben als Fallback für Anlagen erhalten, die den lokalen Remote-Temperature-Weg nicht nutzen können.

### 2. MELCloud Home als Primärweg

Wer bewusst bei der Cloud-Steuerung bleibt, kann die Offset-Automation weiterverwenden:

`melcloud_home_external_temperature_control_public.yaml`

### 3. Lokale Mitsubishi-Integration mit Remote Temperature — empfohlen

```text
Home Assistant -> lokales LAN/WLAN -> MAC-577IF2-E -> Mitsubishi
```

MELCloud Home kann parallel als manueller Fallback bestehen bleiben.

---

## Was von der bisherigen Arbeit weiter wichtig bleibt

Remote Temperature ersetzt nur die **Temperaturkompensation der Hauptklima**. Weiterhin nützlich bleiben:

- lokale MAC-577-Steuerung
- `hvac_action`
- Zusatzheizungslogik
- Winterreserve
- Urlaubs-/Sicherheitslogik
- Sensor-Durchschnitt
- Lamellensteuerung
- Neustart-/Fallback-Sicherheit

Alte lokale Offset-/Stufen-Dokumente sind nur noch **Archiv/Legacy**. Wer über einen alten Link hierher kommt, sollte die aktuelle Anleitung oben verwenden.

---

## Wichtig

Nicht zwei vollständige Hauptregelungen gleichzeitig gegen dasselbe Klimagerät laufen lassen.

Vor Änderungen an Integration, Entity-IDs oder Automationen immer ein Home-Assistant-Backup erstellen.
