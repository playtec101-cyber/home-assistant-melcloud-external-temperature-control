# Deutsche Kurzfassung

Dieses Repository zeigt eine in Home Assistant getestete Lösung für Mitsubishi-Electric-Klimaanlagen über die **klassische MELCloud-Integration**.

## Ziel

Die interne Temperaturmessung des Innengeräts wird nicht als alleinige Referenz verwendet. Stattdessen regelt Home Assistant anhand externer Raumtemperatursensoren.

Enthalten sind zwei Varianten:

- **Schlafzimmer:** ein externer Temperatursensor
- **Wohnzimmer:** mehrere Temperatursensoren, deren Mittelwert verwendet wird

Zusätzlich enthalten:

- Korrektur des Mitsubishi-Sollwerts
- Übernahme manueller Sollwertänderungen aus MELCloud/App/Sprachsteuerung
- Schutz vor Rückkopplungsschleifen
- zwei Lösungen für den problematischen Automatikbetrieb
- zusätzlicher MELCloud-Abruf alle fünf Minuten
- optionale horizontale Lamellensteuerung

## Wichtig zum Automatikmodus

Im getesteten System funktionierten festes HEAT und COOL grundsätzlich brauchbar. Das deutlich größere Problem war der Mitsubishi-AUTO-Modus.

### Wohnzimmer

Beim Einschalten von AUTO entscheidet Home Assistant zunächst anhand des externen Mittelwerts:

- ab Wunsch +0,5 °C -> COOL
- ab Wunsch -0,5 °C -> HEAT
- innerhalb ±0,5 °C -> AUTO bleibt direkt aktiv

HEAT/COOL bleibt gegebenenfalls 15 Minuten aktiv. Danach geht die Anlage zurück in echtes Mitsubishi-AUTO. Dort wird der Sollwert dynamisch korrigiert:

```text
Mitsubishi-Soll =
Mitsubishi-Ist - (externer Mittelwert - Wunschtemperatur)
```

### Schlafzimmer

Hier wird der schlechte native Automatikbetrieb umgangen, indem Home Assistant bewusst zwischen HEAT und COOL umschaltet:

- HEAT -> COOL erst nach 15 Minuten bei mindestens Wunsch +1,0 °C
- COOL -> HEAT erst nach 15 Minuten bei höchstens Wunsch -1,0 °C

Dadurch entstehen keine schnellen Moduswechsel.

## MELCloud von ca. 15 Minuten auf maximal ca. 5 Minuten

Mit `homeassistant.update_entity` wird die MELCloud-Klimaentität alle fünf Minuten zusätzlich aktualisiert.

Der Test hat funktioniert: Eine Änderung in MELCloud auf 22,5 °C wurde beim nächsten 5-Minuten-Zeitpunkt um 19:10:02 von Home Assistant übernommen.

Wichtig: `/5` bedeutet die Zeitpunkte `:00`, `:05`, `:10`, ... und nicht „genau fünf Minuten nach der Änderung“.

## Datenschutz

Die veröffentlichten Dateien enthalten keine IP-Adressen, Passwörter, Tokens, API-Keys, E-Mail-Adressen, MAC-Adressen oder Webhook-IDs. Die Entity-IDs wurden durch neutrale Beispielnamen ersetzt.

Die englische `README.md` enthält die vollständige Installationsanleitung.
