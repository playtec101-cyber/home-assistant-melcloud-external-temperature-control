# Mitsubishi + Home Assistant: externe Raumtemperatur ohne Offset – Remote Temperature

> **Update 11.09.2026:** Für Nutzer der lokalen Integration `pymitsubishi/homeassistant-mitsubishi` ist ein deutlich einfacherer Weg verfügbar: **Remote Temperature**. Wenn diese Funktion mit dem eigenen Mitsubishi-Adapter/Klimagerät funktioniert, wird die bisherige Offset-/Stufenlogik für die Hauptklima nicht mehr benötigt.

## Kurzfassung

Bisher:

```text
externe Raumtemperatur
-> Abweichung berechnen
-> Offset / 0,5-°C-Stufe
-> künstlichen Mitsubishi-Sollwert setzen
```

Jetzt empfohlen:

```text
externer Raumfühler -> Mitsubishi Remote Temperature
Wunschtemperatur    -> Mitsubishi-Sollwert
```

Die Mitsubishi bekommt also direkt die reale externe Raumtemperatur und regelt mit ihrer eigenen Inverter-/AUTO-Logik auf den echten Wunschwert.

## Voraussetzungen

- Home Assistant
- lokale Integration `pymitsubishi/homeassistant-mitsubishi`
- Mitsubishi-Adapter/Innengerät mit funktionierender Remote-Temperature-Unterstützung
- externer Temperaturfühler oder ein HA-Durchschnittssensor

## Einrichtung

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
- Sensor-Durchschnitt
- Lamellensteuerung
- Restart-/Fallback-Logik

## Ältere Wege

Die alten Offset-/Stufenlösungen bleiben als Fallback relevant für:

1. klassische/Legacy-MELCloud-Installationen;
2. MELCloud Home als bewussten Primärweg;
3. lokale Installationen, bei denen Remote Temperature nicht zuverlässig funktioniert.

Aktuelle vollständige Anleitung und Legacy-Dateien:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control

Empfohlene Remote-Temperature-Anleitung:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control/blob/main/REMOTE_TEMPERATURE_RECOMMENDED.md
