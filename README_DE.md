# Home Assistant + Mitsubishi Electric – externe Raumtemperatur

> **Update 11.09.2026:** Nutzer der lokalen Integration `pymitsubishi/homeassistant-mitsubishi`, bei denen **Remote Temperature** funktioniert, benötigen die bisherige Offset-/Stufenlogik für die Hauptklima nicht mehr. Stattdessen wird der externe Raumfühler direkt an die Mitsubishi übergeben. Der Geräte-Sollwert bleibt der echte Wunschwert.

Dieses Repository dokumentiert weiterhin drei Lösungswege. Die älteren Varianten bleiben bewusst als **Legacy/Fallback** erhalten, damit Nutzer ohne funktionierende lokale Remote-Temperature-Unterstützung weiterhin eine Lösung haben.

---

## Welche Lösung soll ich nehmen?

### Lösung 1 – Legacy / klassische MELCloud-Regelung

Für ältere Installationen oder Setups, bei denen Home Assistant HEAT/COOL selbst steuern bzw. den Gerätesollwert kompensieren muss.

Typische Dateien:

- `living_room_multi_sensor.yaml`
- `bedroom_single_sensor.yaml`
- `melcloud_refresh_5min.yaml`
- `optional_horizontal_swing.yaml`

**Status:** weiterhin nutzbar, aber nicht mehr der bevorzugte Weg für eine funktionierende lokale Mitsubishi-Integration mit Remote Temperature.

---

### Lösung 2 – MELCloud Home als Primärweg

Wenn die Cloud-Steuerung bewusst beibehalten werden soll, bleibt die Offset-/Automationslösung sinnvoll:

- AUTO bleibt AUTO
- HEAT bleibt HEAT
- COOL bleibt COOL
- Home Assistant korrigiert den Mitsubishi-Sollwert anhand der externen Raumtemperatur

Datei:

`melcloud_home_external_temperature_control_public.yaml`

**Status:** weiterhin sinnvoll für Cloud-Primärbetrieb.

---

### Lösung 3 – empfohlen: lokale Mitsubishi-Integration mit Remote Temperature

Getestete Integration:

`pymitsubishi/homeassistant-mitsubishi`

Primärweg:

```text
Home Assistant -> lokales LAN/WLAN -> MAC-577IF2-E -> Klimaanlage
```

MELCloud Home kann parallel als manueller Fallback bestehen bleiben:

```text
MELCloud Home -> Mitsubishi Cloud -> Klimaanlage
```

### Einrichtung

1. In der lokalen Mitsubishi-Integration **Experimental Features** aktivieren.
2. Unter **External Temperature Sensor** den gewünschten externen Raumfühler auswählen.
3. Danach bei der neuen Entität **Temperature Source** den Wert von `Internal` auf `Remote` stellen.
4. Die Wunschtemperatur direkt als Gerätesollwert setzen.
5. Keine zusätzliche Offset-/Stufenberechnung mehr für die Hauptklima verwenden.

Beispiel:

```text
Externer Raumfühler: 24,3 °C
Wunschtemperatur:    23,0 °C
Geräte-Sollwert:     23,0 °C
Temperature Source:  Remote
```

Die Mitsubishi erhält damit die externe Raumtemperatur als Regelgröße und entscheidet selbst über Heizen, Kühlen oder Idle.

### Wichtig

Bei `Remote` ist der externe Sensor die führende Raumtemperaturquelle. Der interne Mitsubishi-Sensor bleibt als interner/Fallback-Wert vorhanden, ist aber im normalen Remote-Betrieb nicht mehr die Komfortreferenz.

Wenn der externe Sensor während laufendem Home Assistant ungültig/unavailable wird, kann die Integration auf den internen Sensor zurückfallen. Fällt dagegen Home Assistant oder die Netzwerkverbindung zur Klimaanlage vollständig aus, kann die Anlage den zuletzt empfangenen Remote-Wert zunächst weiterverwenden. Deshalb: Sensorqualität, HA-Verfügbarkeit und Netzwerkstabilität beachten.

### Externe Sensoren

Für Remote Temperature sollte möglichst ein echter, unabhängiger Raumfühler verwendet werden. Heizkörperthermostate messen oft zu nah am Heizkörper und können den Raumwert verfälschen.

Bei mehreren Sensoren kann ein Home-Assistant-Kombinationssensor mit **arithmetischem Mittel** verwendet werden. Dieser Average wird dann als Remote-Sensor ausgewählt.

---

## Was bleibt von der bisherigen YAML-Arbeit relevant?

Sehr viel. Nur der **Temperatur-Kompensationsmotor der Hauptklima** wird bei Lösung 3 ersetzt.

Weiterhin relevant bleiben z. B.:

- lokale MAC-577-Anbindung
- MELCloud-Home-Fallback
- `hvac_action`
- Zusatzheizungslogik
- FRITZ!DECT-Freigaben
- Winterreserve
- Urlaubs-/Sicherheitsabschaltungen
- Wunschtemperatur-Helfer
- Sensor-Average
- Lamellensteuerung
- Restart-/Fallback-Sicherheit

Die bisherigen Offset-/Stufen-Dateien bleiben deshalb als Legacy/Fallback dokumentiert und werden nicht gelöscht.

---

## `hvac_action`

Die lokale Integration liefert den echten Betriebszustand:

```text
heating  -> Anlage heizt aktiv
cooling  -> Anlage kühlt aktiv
idle     -> Anlage ist eingeschaltet, Verdichter arbeitet gerade nicht
```

Für Zusatzheizungen bleibt diese Information wichtig. `idle` ist kein Beweis dafür, dass der Raum bereits warm genug ist. Externe Temperaturbedingungen und Hysteresen müssen weiterhin entscheiden, ob Zusatzwärme erforderlich ist.

`AUTO + cooling` bleibt dagegen ein klarer Sperrzustand für Zusatzheizungen.

---

## Wunschtemperatur

Im empfohlenen Remote-Temperature-Weg gilt:

```text
Wunschtemperatur-Helfer = Benutzerwunsch / Quelle der Wahrheit
Klima-Sollwert           = derselbe Wunschwert (nur Geräteauflösung beachten)
Externer Sensor          = reale Raumtemperaturquelle für Mitsubishi
```

Damit entfällt die frühere Logik:

```text
interner Sensor -> Fehler berechnen -> Offset/Stufe -> künstlicher Gerätesollwert
```

---

## Bestehende Dateien / Archiv

| Datei | Status |
| --- | --- |
| `melcloud_home_external_temperature_control_public.yaml` | weiterhin sinnvoll für MELCloud-Home-Primärbetrieb |
| `living_room_multi_sensor.yaml` | Legacy/Fallback |
| `bedroom_single_sensor.yaml` | Legacy/Fallback |
| `melcloud_refresh_5min.yaml` | Legacy MELCloud |
| `optional_horizontal_swing.yaml` | geräteabhängige Legacy-Ergänzung |
| `local_primary_hvac_action_aux_heating_example.yaml` | Zusatz-/Sicherheitslogik weiterhin nützlich, Temperaturkompensation für Remote-Betrieb nicht mehr empfohlen |
| `LOCAL_CONTROLLER_DESIGN_NOTES.md` | technische Historie / Designentscheidungen |
| `LOCAL_CONTROL_WITH_MELCLOUD_FALLBACK.md` | Migrations-/Fallback-Hintergrund; Remote Temperature hat für die Hauptregelung Vorrang |
| `REMOTE_TEMPERATURE_RECOMMENDED.md` | neue empfohlene Anleitung |

---

## Wichtig

Nicht zwei vollständige Hauptregelungen gleichzeitig gegen dieselbe Klimaanlage laufen lassen.

Die veröffentlichten Beispiele enthalten keine Passwörter, Tokens, API-Keys, E-Mail-Adressen, privaten IP-/MAC-Adressen oder persönlichen Namen.

Feedback von anderen Mitsubishi-/MAC-577IF2-E-Nutzern ist willkommen – besonders zu Remote Temperature, AUTO, `hvac_action`, Sensor-Fallback, Polling und Verhalten bei HA-/Netzwerkausfall.
