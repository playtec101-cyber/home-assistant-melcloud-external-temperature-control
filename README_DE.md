# Deutsche Kurzfassung

Dieses Repository zeigt eine in Home Assistant getestete Regelung für Mitsubishi-Electric-Klimaanlagen mit **externen Raumtemperatursensoren als Komfortreferenz**.

Es gibt jetzt zwei Hauptvarianten:

1. **Lokale Mitsubishi-Steuerung als Primärweg, MELCloud Home als Fallback**
2. **MELCloud Home als Primärweg**

Die ältere klassische MELCloud-Variante bleibt ebenfalls dokumentiert.

## Empfohlene Architektur: lokal primär, MELCloud als Reserve

Getesteter Primärweg:

```text
Home Assistant -> lokales Netzwerk -> MAC-577IF2-E -> Klimaanlage
```

Fallback:

```text
MELCloud Home -> Mitsubishi Cloud -> Klimaanlage
```

Vorteile:

- Bei DSL-/Internetausfall bleibt die lokale Home-Assistant-Steuerung verfügbar.
- Bei MELCloud-Störung läuft die lokale Regelung weiter.
- Bei Ausfall von Home Assistant/Server kann MELCloud Home weiter als Reserve dienen, sofern Internet und Mitsubishi-Cloud verfügbar sind.
- Die MELCloud-App kann auf Handy/Tablet weiterhin manuell verwendet werden.
- Rückkopplungsschleifen werden durch den Helfer für den zuletzt automatisch gesetzten Zielwert verhindert.

Die getestete lokale Integration ist:

`pymitsubishi/homeassistant-mitsubishi`

Ausführliche Anleitung:

`LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md`

## Sollwert-Synchronisation

Beide Richtungen wurden getestet:

```text
Home Assistant lokal -> Klimaanlage -> MELCloud Home
```

und

```text
MELCloud Home -> Klimaanlage -> lokale Home-Assistant-Entität
```

Dadurch kann eine manuelle Änderung in MELCloud weiterhin als echter Nutzerwunsch erkannt und in den Wunschtemperatur-Helfer übernommen werden.

## IR-Fernbedienung ebenfalls getestet

Auch Änderungen mit der originalen Mitsubishi-IR-Fernbedienung wurden erfolgreich zurückgemeldet.

Getesteter Signalweg:

```text
IR-Fernbedienung -> Platine der Inneneinheit -> interne Schnittstelle/CN105 -> MAC-577IF2-E
```

Danach wurde der neue Sollwert:

- von der lokalen Home-Assistant-Entität übernommen,
- über die vorhandene Wunschwert-Synchronisation in den Wunschtemperatur-Helfer geschrieben,
- und auch in MELCloud Home angezeigt.

Im getesteten System erschien die Änderung lokal innerhalb weniger Sekunden.

Die IR-Fernbedienung bleibt damit zusätzlich ein direkter Bedienweg, der weder Home Assistant noch Internet benötigt.

## Externe Raumtemperaturregelung

Die Klimaanlage regelt nicht allein nach ihrem internen Sensor. Home Assistant berechnet die notwendige Sollwertkorrektur aus der Abweichung zwischen externer Raumtemperatur und Wunschtemperatur.

Im aktuellen vollständigen Controller bleibt der vom Nutzer gewählte Mitsubishi-Modus erhalten:

- AUTO bleibt AUTO
- HEAT bleibt HEAT
- COOL bleibt COOL

Im AUTO-Modus werden feste 0,5-°C-Korrekturstufen verwendet. HEAT und COOL nutzen außerhalb der Neutralzone eine lineare 1:1-Korrektur.

## Horizontale Lamellen

Bei der getesteten lokalen Integration lautet die Mittelstellung:

```text
center
```

Bei MELCloud Home lautet sie:

```text
centre
```

`Swing` funktioniert in beiden getesteten Varianten.

## Wichtig bei der Umstellung

Nicht zwei vollständige Regelungs-YAMLs gleichzeitig gegen dieselben Klimageräte laufen lassen.

MELCloud Home darf als Integration und manueller Fallback aktiv bleiben. Aktiv sein soll aber nur **eine** Hauptautomatik: entweder lokal oder cloudbasiert.

Eine einfache Migration ist möglich, indem die bisherige MELCloud-Entität z. B. von:

```text
climate.raum
```

auf

```text
climate.raum_melcloud
```

umbenannt wird und die neue lokale Entität anschließend den bisherigen Hauptnamen `climate.raum` erhält. Dann müssen bestehende Automationen nicht überall umgeschrieben werden.

## Datenschutz

Die veröffentlichten Beispiele enthalten keine IP-Adressen, Passwörter, Tokens, API-Keys, E-Mail-Adressen, MAC-Adressen oder privaten Hostnamen. Die Entity-IDs sind neutrale Beispielnamen.

Die englische `README.md` enthält die ausführlichere Gesamtdokumentation.
