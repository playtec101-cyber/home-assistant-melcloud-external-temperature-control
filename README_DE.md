# Deutsche Kurzfassung

Dieses Repository zeigt getestete Home-Assistant-Regelungen für Mitsubishi-Electric-Klimaanlagen mit **externen Raumtemperatursensoren als Komfortreferenz**.

Es gibt **drei unterschiedliche Lösungswege**, die bewusst nebeneinander dokumentiert bleiben.

## Lösung 1 – älterer / simulierter AUTO-Betrieb mit HEAT und COOL

Bei dieser älteren Variante übernimmt Home Assistant einen größeren Teil der Betriebsentscheidung selbst.

Je nach Abweichung zwischen externer Raumtemperatur und Wunschtemperatur entscheidet die YAML-Logik bewusst zwischen:

```text
HEAT
oder
COOL
```

Damit kann ein AUTO-ähnlicher Betrieb entstehen, auch wenn der native Mitsubishi-AUTO-Modus nicht zufriedenstellend arbeitet.

Die älteren Beispiele enthalten unter anderem:

- eine Home-Assistant-gesteuerte HEAT/COOL-Umschaltung mit Hysterese bzw. Stabilitätszeit;
- eine Variante, die beim Start zunächst HEAT oder COOL wählt und später wieder in nativen Mitsubishi-AUTO zurückkehrt.

Dazu gehören insbesondere:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`
- `melcloud_refresh_5min.yaml`
- `optional_horizontal_swing.yaml`

Diese Dateien stammen überwiegend aus der Zeit der klassischen MELCloud-Integration.

---

## Lösung 2 – MELCloud Home als Primärweg, echter Mitsubishi-AUTO-Modus

Hier bleibt der vom Nutzer gewählte Mitsubishi-Modus erhalten:

- AUTO bleibt AUTO
- HEAT bleibt HEAT
- COOL bleibt COOL

Home Assistant simuliert den AUTO-Modus also **nicht mehr durch Umschalten zwischen HEAT und COOL**.

Stattdessen wird nur der Mitsubishi-Sollwert anhand der externen Raumtemperatur korrigiert.

Aktueller vollständiger Controller:

`melcloud_home_external_temperature_control_public.yaml`

### AUTO-Korrektur

Im AUTO-Modus werden feste 0,5-°C-Korrekturstufen verwendet:

| Absolute Raumabweichung | Korrektur |
| --- | --- |
| `<= 0,25 °C` | `0,0 °C` |
| `> 0,25 bis 0,75 °C` | `0,5 °C` |
| `> 0,75 bis 1,25 °C` | `1,0 °C` |
| `> 1,25 bis 1,75 °C` | `1,5 °C` |
| `> 1,75 bis 2,25 °C` | `2,0 °C` |
| `> 2,25 bis 2,75 °C` | `2,5 °C` |
| `> 2,75 °C` | `3,0 °C` |

HEAT und COOL verwenden dieselbe Neutralzone von ±0,25 °C und außerhalb davon eine lineare 1:1-Korrektur.

Der Helfer für den zuletzt automatisch gesetzten Zielwert verhindert, dass die eigene Sollwertkorrektur als neuer Nutzerwunsch zurück in die Wunschtemperatur geschrieben wird.

### Nachteil

Der primäre Steuerweg hängt bei dieser Variante von MELCloud Home bzw. der Mitsubishi-Cloud ab.

---

## Lösung 3 – lokale Primärsteuerung, MELCloud Home nur als Fallback

Das ist die aktuelle lokale Architektur.

Primärweg:

```text
Home Assistant -> lokales Netzwerk -> MAC-577IF2-E -> Klimaanlage
```

Fallback / manueller Zweitweg:

```text
MELCloud Home -> Mitsubishi Cloud -> Klimaanlage
```

Die getestete lokale Integration ist:

`pymitsubishi/homeassistant-mitsubishi`

Ausführliche Anleitung:

`LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md`

### Was gegenüber Lösung 2 gleich bleibt

Die eigentliche Regelungslogik bleibt im Wesentlichen gleich:

- echter Mitsubishi-AUTO-Modus bleibt AUTO;
- externe Raumtemperatursensoren bestimmen die Sollwertkorrektur;
- Wunschtemperatur- und Rückkopplungsschutz-Helfer bleiben erhalten.

### Was sich ändert

Die normalen Home-Assistant-Befehle laufen lokal über den MAC-577IF2-E und benötigen für den Primärweg nicht mehr die Mitsubishi-Cloud.

MELCloud Home bleibt trotzdem eingerichtet und kann weiter als manueller Fallback genutzt werden.

### Getestete Synchronisation

Beide Richtungen wurden praktisch getestet:

```text
Home Assistant lokal -> Klimaanlage -> MELCloud Home
```

und

```text
MELCloud Home -> Klimaanlage -> lokale Home-Assistant-Entität
```

Damit kann auch eine manuelle Änderung über MELCloud weiterhin von Home Assistant erkannt und in den Wunschtemperatur-Helfer übernommen werden, ohne eine Rückkopplungsschleife zu erzeugen.

### Ausfallverhalten

- DSL/Internet ausgefallen: lokale Home-Assistant-Steuerung läuft weiter, solange Server, LAN/WLAN und Adapter funktionieren.
- MELCloud gestört: lokale Regelung läuft weiter.
- Home Assistant/Server ausgefallen, Internet vorhanden: MELCloud Home kann weiter als manueller Fallback dienen.

### Horizontale Lamellen

Bei der getesteten lokalen Integration lautet die Mittelstellung:

```text
center
```

Bei MELCloud Home:

```text
centre
```

`Swing` funktioniert in beiden getesteten Varianten.

---

## Infrarot-Fernbedienung

Die originale Mitsubishi-IR-Fernbedienung sollte die Inneneinheit in **allen drei Lösungswegen weiterhin direkt bedienen können**, weil sie unabhängig von Home Assistant mit der Klimaanlage kommuniziert.

**Nicht getestet ist bisher die Rückmeldung dieser IR-Änderung in die jeweilige Home-Assistant-Regelung.**

Wir behaupten deshalb derzeit ausdrücklich nicht als verifiziert, wie schnell oder zuverlässig ein per IR geänderter Sollwert anschließend erscheint in:

- der klassischen MELCloud-Entität aus Lösung 1;
- der MELCloud-Home-Entität aus Lösung 2;
- der lokalen MAC-577IF2-E-Entität aus Lösung 3;
- dem Wunschtemperatur-Helfer;
- MELCloud Home selbst.

Gerade bei Lösung 3 ist es technisch plausibel, dass der geänderte Gerätezustand über den MAC-577IF2-E lokal wieder von Home Assistant gelesen wird. Bis zum Praxistest wird dies aber nur als **wahrscheinlich/erwartet**, nicht als getestet dokumentiert.

Ein sinnvoller Test ist:

1. Sollwert mit der IR-Fernbedienung um 1 °C ändern.
2. Temperaturattribut der aktiven Home-Assistant-Klimaentität beobachten.
3. Wunschtemperatur-Helfer beobachten.
4. MELCloud Home kontrollieren.
5. Verzögerung notieren.

---

## Wichtig bei Lösung 3

Nicht zwei vollständige Hauptregelungen gleichzeitig gegen dieselben Klimageräte laufen lassen.

MELCloud Home darf als Integration und manueller Fallback aktiv bleiben. Aktiv sein soll aber nur **eine Hauptautomatik**.

Eine einfache Migration ist möglich, indem die bisherige MELCloud-Entität zum Beispiel von:

```text
climate.raum
```

auf

```text
climate.raum_melcloud
```

umbenannt wird und die neue lokale Entität anschließend den bisherigen Hauptnamen `climate.raum` erhält.

---

## Datenschutz

Die veröffentlichten Beispiele enthalten keine IP-Adressen, Passwörter, Tokens, API-Keys, E-Mail-Adressen, MAC-Adressen oder privaten Hostnamen. Die Entity-IDs sind neutrale Beispielnamen.

Die englische `README.md` enthält die ausführlichere Gesamtdokumentation.
