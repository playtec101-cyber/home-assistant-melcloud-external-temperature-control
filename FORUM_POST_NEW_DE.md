# Mitsubishi + Home Assistant: vier Wege für externe Raumtemperatur – Remote Temperature jetzt empfohlen

> **Update 11.09.2026:** Für Nutzer der lokalen Integration `pymitsubishi/homeassistant-mitsubishi` ist mit **Remote Temperature** inzwischen ein deutlich einfacherer Weg verfügbar. Wenn diese Funktion mit dem eigenen Mitsubishi-Adapter/Klimagerät funktioniert, wird die bisherige Offset-/Stufenlogik für die Hauptklima nicht mehr benötigt.

## Die vier möglichen Wege

### 1. Home Assistant simuliert AUTO und wählt HEAT oder COOL

Home Assistant entscheidet anhand externer Raumfühler selbst, ob die Mitsubishi in `HEAT` oder `COOL` laufen soll.

**Status:** ältere/Legacy-Lösung. Sinnvoll nur, wenn der native Mitsubishi-AUTO-Modus nicht verwendet werden soll oder nicht zufriedenstellend arbeitet.

### 2. MELCloud Home + echter Mitsubishi-AUTO + Sollwert-Offset

AUTO bleibt echtes Mitsubishi-AUTO. Home Assistant schaltet den Modus nicht um, sondern korrigiert nur den Gerätesollwert anhand der externen Raumtemperatur.

**Status:** weiterhin sinnvoll, wenn MELCloud Home bewusst der Primärweg bleiben soll.

### 3. Lokale `pymitsubishi`-Integration + echter AUTO + Sollwert-Offset

Die Steuerung läuft lokal über den Mitsubishi-Adapter. AUTO bleibt echtes Mitsubishi-AUTO, aber Home Assistant verwendet weiterhin die bisherige Offset-/Stufenlogik.

```text
externe Raumtemperatur
-> Abweichung berechnen
-> Offset / 0,5-°C-Stufe
-> künstlichen Mitsubishi-Sollwert setzen
```

**Status:** lokaler Fallback, wenn Remote Temperature nicht verfügbar oder nicht zuverlässig nutzbar ist.

### 4. Lokale `pymitsubishi`-Integration + Remote Temperature — empfohlen

Jetzt empfohlen:

```text
externer Raumfühler -> Mitsubishi Remote Temperature
Wunschtemperatur    -> Mitsubishi-Sollwert
```

Die Mitsubishi bekommt direkt die reale externe Raumtemperatur und regelt mit ihrer eigenen Inverter-/AUTO-Logik auf den echten Wunschwert.

## Einrichtung von Remote Temperature

1. Mitsubishi-Air-Conditioner-Integration öffnen.
2. **Experimental Features** aktivieren.
3. Unter **External Temperature Sensor** den gewünschten externen Raumfühler auswählen.
4. Integration neu laden, falls die neue Entität nicht sofort erscheint.
5. Die neue Entität **Temperature Source** öffnen.
6. Von `Internal` auf `Remote` stellen.
7. Nach einem Home-Assistant-Neustart prüfen, dass `Remote` erhalten bleibt.
8. Als Klimasollwert direkt die echte Wunschtemperatur verwenden.
9. Alte Hauptregelung mit Offset/Stufen deaktivieren.

## Mehrere Raumfühler

Wer mehrere unabhängige Temperaturfühler verwendet, kann in Home Assistant einen Kombinations-/Statistik-Helfer mit **arithmetischem Mittel** erstellen und diesen Durchschnitt als External Temperature Sensor auswählen.

Für die Raumreferenz sind unabhängige Raumfühler meist besser geeignet als Heizkörperthermostate, weil deren Messwert durch die Nähe zum Heizkörper verfälscht sein kann.

## Verhalten / Fallback

Wenn der externe Sensor während laufendem Home Assistant ungültig oder nicht verfügbar wird, kann die Integration auf den internen Mitsubishi-Sensor zurückfallen.

Wichtig: Wenn Home Assistant selbst oder die Netzwerkverbindung zur Klimaanlage komplett ausfällt, kann die Mitsubishi den zuletzt empfangenen Remote-Wert zunächst weiterverwenden, bis die Kommunikation wiederhergestellt ist.

## Was bleibt von den alten Automationen sinnvoll?

Remote Temperature ersetzt nur die Haupt-Temperaturkompensation. Weiterhin nützlich bleiben z. B.:

- `hvac_action`
- Zusatzheizungen / Heizkörperlogik
- Winterreserve
- Urlaubs-/Sicherheitsabschaltungen
- Wunschtemperatur-Helfer
- Sensor-Durchschnitt
- Lamellensteuerung
- Restart-/Fallback-Logik

## Vollständige Dokumentation

Übersicht aller vier Wege:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control

Empfohlene Remote-Temperature-Anleitung:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control/blob/main/REMOTE_TEMPERATURE_RECOMMENDED.md
