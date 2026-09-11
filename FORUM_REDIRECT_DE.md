## Hinweis – dieser Beitrag ist veraltet

Die hier beschriebene Offset-/Stufenregelung war ein älterer Lösungsweg.

Seit dem **11.09.2026** nutze ich für die lokale Mitsubishi-Integration den deutlich einfacheren **Remote-Temperature-Weg**:

- Experimental Features aktivieren
- externen Raumtemperatursensor auswählen
- Temperature Source = `Remote`
- echter Wunschwert direkt als Mitsubishi-Sollwert
- keine zusätzliche Offset-/Stufenberechnung mehr

Die aktuelle Anleitung steht hier:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control/blob/main/REMOTE_TEMPERATURE_RECOMMENDED.md

Übersicht aller drei Wege (Legacy MELCloud / MELCloud Home / lokal Remote Temperature):

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control
