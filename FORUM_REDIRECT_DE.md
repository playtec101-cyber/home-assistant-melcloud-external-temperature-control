## Hinweis – dieser Beitrag ist veraltet

Die hier beschriebene Offset-/Stufenregelung war ein älterer Lösungsweg.

Seit dem **11.09.2026** gibt es eine neue, übersichtlich strukturierte Dokumentation mit **vier getrennten Wegen**:

1. Home Assistant simuliert AUTO und wählt HEAT/COOL selbst.
2. MELCloud Home + echter Mitsubishi-AUTO + Sollwert-Offset.
3. Lokale `pymitsubishi`-Integration + echter AUTO + Sollwert-Offset.
4. Lokale `pymitsubishi`-Integration + **Remote Temperature** — aktuell empfohlen.

Beim empfohlenen Remote-Temperature-Weg gilt:

- Experimental Features aktivieren
- externen Raumtemperatursensor auswählen
- Temperature Source = `Remote`
- echter Wunschwert direkt als Mitsubishi-Sollwert
- keine zusätzliche Offset-/Stufenberechnung mehr

Aktuelle Übersicht aller vier Wege:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control

Empfohlene Remote-Temperature-Anleitung:

https://github.com/playtec101/home-assistant-melcloud-external-temperature-control/blob/main/REMOTE_TEMPERATURE_RECOMMENDED.md
