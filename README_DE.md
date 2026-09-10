# Deutsche Kurzfassung

Dieses Repository zeigt Home-Assistant-Regelungen für Mitsubishi-Electric-Klimaanlagen mit **externen Raumtemperatursensoren als Komfortreferenz**.

Es gibt drei bewusst getrennte Lösungswege.

## Lösung 1 – älterer / simulierter AUTO-Betrieb

Home Assistant entscheidet selbst zwischen HEAT und COOL. Diese Variante bleibt als ältere Lösung für Installationen dokumentiert, bei denen der native Mitsubishi-AUTO-Modus nicht zufriedenstellend arbeitet.

Relevante Dateien:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`
- `melcloud_refresh_5min.yaml`
- `optional_horizontal_swing.yaml`

---

## Lösung 2 – MELCloud Home als Primärweg, echter Mitsubishi-AUTO-Modus

AUTO bleibt AUTO, HEAT bleibt HEAT und COOL bleibt COOL. Home Assistant korrigiert nur den Geräte-Sollwert anhand der externen Raumtemperatur.

Datei:

`melcloud_home_external_temperature_control_public.yaml`

Diese Cloud-Variante enthält noch die ältere optionale Rücksynchronisierung von Geräte-Sollwerten in den Wunschtemperatur-Helfer.

---

## Lösung 3 – lokale Primärsteuerung, MELCloud Home nur als Fallback

Das ist die aktuelle bevorzugte Architektur.

```text
Home Assistant -> lokales Netzwerk -> MAC-577IF2-E -> Klimaanlage
```

MELCloud Home kann parallel als manueller Fallback bestehen bleiben:

```text
MELCloud Home -> Mitsubishi Cloud -> Klimaanlage
```

Getestete lokale Integration:

`pymitsubishi/homeassistant-mitsubishi`

Aktuelle neutrale öffentliche Datei:

`local_primary_hvac_action_aux_heating_example.yaml`

Passende Helfer:

`local_primary_helpers_example.yaml`

Ausführliche technische Begründung aller Änderungen:

`LOCAL_CONTROLLER_DESIGN_NOTES.md`

Migrations- und Testanleitung:

`LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md`

### Die wichtigsten Änderungen der lokalen Version

Der native AUTO-Modus bleibt erhalten. Home Assistant korrigiert weiterhin nur den Geräte-Sollwert anhand der externen Raumtemperatur.

Die lokale Integration liefert zusätzlich `hvac_action`. Entscheidend ist die neue Interpretation von `idle`:

```text
HEAT            -> Zusatzheizung darf nach weiteren Bedingungen helfen
AUTO + heating  -> Zusatzheizung darf nach weiteren Bedingungen helfen
AUTO + idle     -> neutral: Zusatzheizung darf nur dann helfen, wenn die externen Bedarfsbedingungen erfüllt sind
AUTO + cooling  -> harte Sperre
AUTO + unklar   -> harte Sperre
COOL/DRY/FAN    -> harte Sperre
OFF             -> normale Zusatzheizung aus, nur separate Winterreserve möglich
```

`idle` wird **nicht** als Heizen interpretiert. Es ist lediglich kein pauschaler Sperrzustand mehr. Hintergrund: Die lokale Integration liefert `idle`, sobald der Verdichter gerade nicht arbeitet. Daraus folgt nicht, dass der externe Raum bereits auf Wunschtemperatur ist.

Damit bleiben die externen Sensoren entscheidend: FRITZ!-Zusatzheizung nur bei echter Untertemperatur und nach 25 Minuten; ein separater Zusatzheizer nur nach seiner eigenen Temperaturbedingung.

`AUTO + cooling` bleibt dagegen immer gesperrt, damit Klimaanlage und Zusatzheizung nie gegeneinander arbeiten.

### Wunschtemperatur ist autoritativ

Die lokale Version schreibt Geräte-Sollwertänderungen bewusst **nicht** mehr zurück in den Wunschtemperatur-Helfer.

Grund: Automatische Sollwertkorrekturen und verzögerte Geräte-Rückmeldungen können sich zeitlich überholen. Dadurch kann ein Zwischenwert fälschlich als neuer Benutzerwunsch interpretiert werden und eine Rückkopplungsschleife entstehen.

Deshalb gilt:

```text
Wunschtemperatur-Helfer = Benutzerwunsch / Quelle der Wahrheit
Klima-Sollwert           = automatisch kompensierter Gerätewert
```

Der Helfer für den zuletzt automatisch gesetzten Zielwert bleibt nur zur Erkennung eigener Schreibvorgänge, zur Vermeidung unnötiger Wiederholungen und für den Sicherheits-Retry erhalten.

### AUTO-Korrektur

Die bisherige AUTO-Kennlinie bleibt unverändert:

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

### Warum die AUTO-Kennlinie vorerst nicht verschärft wurde

Bei einem Praxistest am 10.09.2026 lag die externe Raumtemperatur deutlich über der Wunschtemperatur, während `hvac_action` eine Zeit lang `idle` meldete. Zunächst sah das nach einer zu schwachen AUTO-Anregung aus.

Ohne dass Home Assistant den HVAC-Modus änderte, wechselte die Mitsubishi anschließend selbstständig von `idle` auf `cooling` und blieb dabei vollständig im nativen AUTO-Modus.

Darum wurde die Kennlinie **nicht vorschnell steiler gemacht** und es gibt weiterhin keinen erzwungenen AUTO->COOL/HEAT-Fallback. Erst mehrere reale Zyklen sollen zeigen, ob eine Änderung überhaupt nötig ist.

Zum Beobachten kann eine temporäre Markdown-Karte verwendet werden:

```jinja2
HVAC Action: **{{ state_attr('climate.hauptraum_ac', 'hvac_action') }}**
```

### Optionale Zusatzthermostate

Die neutrale Beispielkonfiguration enthält zwei optionale Heizkörper-/Zusatzthermostate.

Freigabe erst nach 25 Minuten stabiler Untertemperatur von mindestens 0,5 °C, nur bei HEAT oder AUTO mit `heating`/`idle` und nur bei geeigneter Außentemperatur. Bei Freigabe wird als Beispiel `Wunsch - 1,0 °C` gesetzt.

Ein Wechsel `heating -> idle` setzt den 25-Minuten-Zähler nicht allein deshalb zurück, solange der externe Heizbedarf weiterhin besteht. `cooling`, unklarer AUTO-Zustand oder der Wegfall einer externen Bedingung beendet die Freigabe dagegen sofort.

Der 25-Minuten-`for:`-Trigger beginnt nach einem Home-Assistant-Neustart bewusst neu. Das kann Zusatzheizung nur verzögern, niemals zu früh einschalten.

### Winterreserve bei ausgeschalteter Klimaanlage

Wenn die Hauptklima AUS ist:

- nachts 23:00–08:00: Beispielziel 18 °C bei gültiger Außentemperatur <=20 °C;
- tagsüber normalerweise AUS;
- fällt die Raumtemperatur unter 16 °C und ist die Außentemperatur gültig <=20 °C, startet eine Tages-Winterreserve und hält bis 18 °C.

Die Reserve endet ebenfalls bei ungültiger/zu warmer Außentemperatur, eingeschalteter Hauptklima oder Abwesenheit.

Die Klima-AUS-Winterlogik und die Klima-NICHT-AUS-Zusatzlogik sind hart getrennt, damit nicht zwei Automationen gleichzeitig dieselben Thermostate ansteuern.

### Separater Zusatzheizer

Der vollständige Produktionsaufbau kann zusätzlich einen schaltbaren Zusatzheizer enthalten. Für ihn gilt dieselbe AUTO-Interpretation: HEAT sowie AUTO+heating/idle dürfen nach den eigenen externen Bedingungen freigeben; AUTO+cooling/unklar sowie COOL/OFF sperren.

Ein manueller Lauf kann auf vier Stunden begrenzt werden. Der absolute Endzeitpunkt wird in einem `input_datetime` gespeichert und bleibt damit über Neustarts erhalten.

23:00 kann ein einmaliges hartes Abschaltereignis bleiben.

### Template-Sicherheit

Alle relevanten `float(...)`-Konvertierungen verwenden explizite Defaults. Ungültige oder nicht verfügbare Sensorwerte führen in den Heizfreigaben zum sicheren Sperrzustand.

### Horizontale Lamelle

Bei der getesteten lokalen Integration lautet die Mittelstellung:

```text
center
```

MELCloud Home kann dagegen `centre` verwenden.

---

## Wichtig

Nicht zwei vollständige Hauptregelungen gleichzeitig gegen dieselben Klimageräte laufen lassen.

Die veröffentlichten Beispiele enthalten keine IP-Adressen, Passwörter, Tokens, API-Keys, E-Mail-Adressen, MAC-Adressen, privaten Hostnamen oder persönlichen Namen. Entity-IDs sind neutrale Platzhalter.

Das Verhalten von `hvac_action` sollte über mehrere echte AUTO-Heiz-/Kühl-/Idle-Zyklen beobachtet werden, bevor die AUTO-Kennlinie weiter verändert wird.
