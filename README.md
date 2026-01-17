# Netatmo Controller - HSL 2.0 für HomeServer 4.13

## Übersicht

Es gibt vier Bausteine:

| Baustein | ID | Beschreibung |
|----------|-----|--------------|
| **OAuth2 TokenManager** | 20030 | OAuth2 Refresh Token Flow mit Auto-Refresh |
| **Netatmo Presence** | 20031 | Smart Outdoor Camera (Status, Events, Flutlicht) |
| **Netatmo Homecoach** | 20032 | Smart Indoor Air Quality Monitor (CO2, Temperatur, Luftfeuchtigkeit) |
| **Netatmo Weather** | 20033 | Wetterstation (Wind, Regen, Temperatur) |

---

## Features

| Feature | 20030 | 20031 | 20032 | 20033 |
|---------|-------|-------|-------|-------|
| OAuth2 Refresh Token Flow | ✅ | - | - | - |
| Automatische Token-Erneuerung | ✅ (5 Min vor Ablauf) | - | - | - |
| Auth Code Exchange (PIN 8) | ✅ | - | - | - |
| Kamera-Status (An/Aus) | - | ✅ | - | - |
| Letztes Event (Typ, Zeit, Person) | - | ✅ | - | - |
| Event-Snapshot & Vignette | - | ✅ | - | - |
| Flutlicht-Status | - | ✅ | - | - |
| Lokale Kamera-URL | - | ✅ | - | - |
| Innen-Temperatur | - | - | ✅ | ✅ |
| Außen-Temperatur | - | - | - | ✅ |
| CO2-Gehalt | - | - | ✅ | ✅ |
| Luftfeuchtigkeit (Innen/Außen) | - | - | ✅ | ✅ |
| Lärmpegel | - | - | ✅ | ✅ |
| Luftdruck | - | - | ✅ | ✅ |
| Gesundheitsindex (0-4) | - | - | ✅ | - |
| Windgeschwindigkeit | - | - | - | ✅ |
| Windrichtung (Grad + Text) | - | - | - | ✅ |
| Böengeschwindigkeit | - | - | - | ✅ |
| Regenmenge (aktuell/1h/24h) | - | - | - | ✅ |
| Token-Persistierung | ✅ | - | - | - |
| Auto-Refresh Timer | ✅ | - | ✅ | ✅ |
| Gira Expert Debug Panel | ✅ | ✅ | ✅ | ✅ |
| Change-Only Outputs | ✅ | ✅ | ✅ | ✅ |


---

## Installation

1. Baustein `20030_OAuth2_TokenManager.py` in den Gira Homeserver importieren
2. In der Logik platzieren und verkabeln
3. Netatmo App-Credentials eintragen

---

## Netatmo OAuth2 Setup

### 1. Netatmo App erstellen

1. Gehe zu [Netatmo Developer](https://dev.netatmo.com/)
2. Erstelle eine neue App
3. Notiere **Client ID** und **Client Secret**

### 2. Verfügbare Scopes

| Scope | Beschreibung | API-Zugriff |
|-------|--------------|-------------|
| `read_station` | Wetterstation lesen | Getstationsdata, Getmeasure |
| `read_thermostat` | Thermostat lesen | Homestatus, Getroommeasure |
| `write_thermostat` | Thermostat steuern | Synchomeschedule, Setroomthermpoint |
| `read_camera` | Smart Indoor Camera lesen | Gethomedata, Getcamerapicture |
| `write_camera` | Indoor Camera steuern | Setpersonsaway, Setpersonshome |
| `access_camera` | Live-Stream Indoor Camera* | Videos, Live-Stream |
| `read_presence` | Smart Outdoor Camera lesen | Gethomedata, Getcamerapicture |
| `access_presence` | Live-Stream Outdoor Camera* | Videos, Live-Stream |
| `read_smokedetector` | Rauchmelder lesen | Gethomedata, Geteventsuntil |
| `read_homecoach` | Indoor Air Quality Monitor | Gethomecoachsdata |

**Hinweis:** Ohne Scope-Angabe wird automatisch `read_station` verwendet.

\* Die `access_*` Scopes müssen bei Netatmo separat beantragt werden (Datenschutz).

### Scope-Kombinationen

Mehrere Scopes werden mit Leerzeichen getrennt:

```
read_camera write_camera access_camera
read_station read_thermostat write_thermostat
read_presence access_presence
```

### 3. Token erhalten (Auth Code Exchange - NEU v2.14)

Ab Version 2.14 ist kein externes Tool (curl/PowerShell) mehr nötig!

#### Automatischer Workflow (empfohlen)

```
┌─────────────────────────────────────────────────────────────────┐
│  1. Token fehlt/abgelaufen                                      │
│     └─► Ausgang 7 (s_auth_url) zeigt Login-Link                │
│                                                                 │
│  2. Link im Browser öffnen                                      │
│     └─► Bei Netatmo anmelden und Zugriff erlauben              │
│                                                                 │
│  3. Redirect nach http://localhost/?code=XXXX&state=test        │
│     └─► Den "code" Parameter aus der URL kopieren              │
│                                                                 │
│  4. Code in Eingang 8 (s_auth_code) eingeben                    │
│     └─► Baustein tauscht Code automatisch gegen Tokens         │
│                                                                 │
│  5. Fertig!                                                     │
│     └─► s_auth_url wird leer, Auto-Refresh läuft              │
└─────────────────────────────────────────────────────────────────┘
```

#### Schritt-für-Schritt

1. **Auth URL kopieren** → Ausgang 7 enthält die fertige URL mit Client-ID und Scopes
2. **URL im Browser öffnen** → Bei Netatmo anmelden
3. **Zugriff erlauben** → Netatmo leitet zu `http://localhost/?code=XXXX&state=test` weiter
4. **Code extrahieren** → Den Wert von `code=` aus der URL kopieren (ca. 57 Zeichen)
5. **Code eingeben** → Am Eingang 8 (`s_auth_code`) einfügen
6. **Automatischer Austausch** → Der Baustein:
   - Tauscht den Code gegen Access + Refresh Token
   - Speichert den Refresh Token remanent
   - Startet den Auto-Refresh Timer
   - Leert die Auth-URL (Token ist jetzt gültig)

#### Manueller Weg (Alternative)

Falls du den klassischen Weg bevorzugst:

**Schritt 1:** Authorization Code im Browser holen:

```
https://api.netatmo.com/oauth2/authorize?client_id=YOUR_CLIENT_ID&redirect_uri=http://localhost&scope=read_camera%20write_camera%20access_camera&state=test
```

Nach Zustimmung wirst du zu einer URL wie dieser weitergeleitet:
```
http://localhost/?state=test&code=AUTHORIZATION_CODE_HERE
```

**Schritt 2:** Authorization Code gegen Tokens tauschen:

```bash
curl -X POST "https://api.netatmo.com/oauth2/token" \
  -d "grant_type=authorization_code" \
  -d "client_id=YOUR_CLIENT_ID" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "code=AUTHORIZATION_CODE_HERE" \
  -d "redirect_uri=http://localhost" \
  -d "scope=read_camera write_camera access_camera"
```

Die Response enthält das `refresh_token` für Eingang 4.

---

## Baustein 20030: OAuth2 TokenManager

### Eingänge

| # | Eingang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | TRIGGER | Number | 1 = Token erneuern |
| 2 | CLIENT_ID | String | Netatmo App Client ID |
| 3 | CLIENT_SECRET | String | Netatmo App Client Secret |
| 4 | REFRESH_TOKEN | String | OAuth2 Refresh Token |
| 5 | TOKEN_URL | String | `https://api.netatmo.com/oauth2/token` |
| 6 | SCOPE | String | z.B. `read_presence access_presence` (leer = automatisch) |
| 7 | TOKENFILE | String | Pfad zur Token-Datei (optional) |
| 8 | AUTH_CODE | String | **NEU v2.15:** Authorization Code (wird automatisch gegen Tokens getauscht) |

### Ausgänge

| # | Ausgang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | ACCESS_TOKEN | String | Aktuelles Access Token |
| 2 | EXPIRES | Number | Gültigkeit in Sekunden |
| 3 | STATUS | Number | 1 = OK, 0 = Fehler |
| 4 | ERROR | String | Fehlermeldung |
| 5 | READY | Number | 1 = Token verfügbar |
| 6 | INSTANCE_ID | Number | Laufzeit-ID für Inter-Instanz-Kommunikation |
| 7 | AUTH_URL | String | Authorization URL (erscheint wenn Token abgelaufen/ungültig) |

### Automatischer Token-Tausch (NEU v2.15)

1. **Auth URL kopieren** (Ausgang 7) und im Browser öffnen
2. Bei Netatmo anmelden und Zugriff erlauben
3. **Code aus der Redirect-URL** kopieren (`http://localhost?code=XXXXX&state=...`)
4. **Code am Eingang 8** eingeben → Tokens werden automatisch geholt!

---

## Baustein 20031: Netatmo Presence

### Eingänge

| # | Eingang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | TRIGGER | Number | 1 = Daten abrufen |
| 2 | ACCESS_TOKEN | String | Von OAuth2 TokenManager (20030) |
| 3 | HOME_ID | String | Netatmo Home ID (optional, sonst erstes Home) |
| 4 | CAMERA_ID | String | Kamera ID (optional, sonst erste Outdoor-Kamera) |

### Ausgänge

| # | Ausgang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | CAMERA_NAME | String | Name der Kamera |
| 2 | CAMERA_STATUS | Number | 1 = An, 0 = Aus/Disconnected |
| 3 | CAMERA_URL | String | VPN-URL für lokalen Zugriff |
| 4 | LAST_EVENT_TYPE | String | Typ des letzten Events |
| 5 | LAST_EVENT_TIME | String | Zeit des letzten Events |
| 6 | LAST_EVENT_PERSON | String | Erkannte Person (falls bekannt) |
| 7 | LAST_EVENT_MESSAGE | String | Event-Nachricht |
| 8 | LAST_EVENT_SNAPSHOT | String | Snapshot-URL des letzten Events (Vollbild) |
| 9 | LAST_EVENT_VIGNETTE | String | Vignette-URL des letzten Events (Thumbnail) |
| 10 | LIGHT_MODE | Number | 1 = Flutlicht an, 0 = aus |
| 11 | STATUS | Number | 1 = OK, 0 = Fehler |
| 12 | ERROR | String | Fehlermeldung |

### Event-Typen

| Event | Beschreibung |
|-------|--------------|
| `Person erkannt` | Unbekannte Person im Bild |
| `Bekannte Person` | Gespeicherte Person erkannt |
| `Tier erkannt` | Tier im Bild |
| `Fahrzeug erkannt` | Auto/Fahrzeug im Bild |
| `Bewegung erkannt` | Allgemeine Bewegung |

---

## Baustein 20032: Netatmo Homecoach

### Eingänge

| # | Eingang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | TRIGGER | Number | 1 = Daten abrufen |
| 2 | ACCESS_TOKEN | String | Von OAuth2 TokenManager (20030) - Alternative zu Instance ID |
| 3 | DEVICE_ID | String | Homecoach Device ID (optional, sonst erstes Gerät) |
| 4 | OAUTH_INSTANCE_ID | Number | Instance ID des OAuth-Bausteins (Alternative zu ACCESS_TOKEN) |
| 5 | INTERVAL | Number | Auto-Refresh Intervall in Sekunden (0=aus, default 300) |

### Ausgänge

| # | Ausgang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | DEVICE_NAME | String | Name des Homecoach |
| 2 | TEMPERATURE | Number | Temperatur in °C |
| 3 | CO2 | Number | CO2-Gehalt in ppm |
| 4 | HUMIDITY | Number | Luftfeuchtigkeit in % |
| 5 | NOISE | Number | Lärmpegel in dB |
| 6 | PRESSURE | Number | Luftdruck in mbar |
| 7 | HEALTH_IDX | Number | Gesundheitsindex (0-4) |
| 8 | HEALTH_STATUS | String | Gesundheitsstatus als Text |
| 9 | WIFI_STRENGTH | Number | WLAN-Stärke in % |
| 10 | LAST_UPDATE | String | Zeitstempel der letzten Messung |
| 11 | STATUS | Number | 1 = OK, 0 = Fehler |
| 12 | ERROR | String | Fehlermeldung |

### Gesundheitsindex

| Index | Status | Beschreibung |
|-------|--------|--------------|
| 0 | Gesund | Ideale Luftqualität |
| 1 | Gut | Gute Luftqualität |
| 2 | Akzeptabel | Akzeptable Luftqualität |
| 3 | Schlecht | Schlechte Luftqualität - Lüften empfohlen |
| 4 | Ungesund | Ungesunde Luftqualität - sofort lüften! |

### CO2-Richtwerte

| CO2 (ppm) | Bewertung |
|-----------|-----------|
| < 800 | Sehr gut |
| 800-1000 | Gut |
| 1000-1400 | Akzeptabel |
| 1400-2000 | Schlecht |
| > 2000 | Kritisch |

---

## Baustein 20033: Netatmo Weather Station

### Eingänge

| # | Eingang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | TRIGGER | Number | 1 = Daten abrufen |
| 2 | ACCESS_TOKEN | String | Von OAuth2 TokenManager (20030) - Alternative zu Instance ID |
| 3 | DEVICE_ID | String | Wetterstation ID (optional, sonst erste Station) |
| 4 | OAUTH_INSTANCE_ID | Number | Instance ID des OAuth-Bausteins (Alternative zu ACCESS_TOKEN) |
| 5 | INTERVAL | Number | Auto-Refresh Intervall in Sekunden (0=aus, default 300) |

### Ausgänge

| # | Ausgang | Typ | Beschreibung |
|---|---------|-----|--------------|
| 1 | STATION_NAME | String | Name der Wetterstation |
| 2 | TEMP_OUT | Number | Außentemperatur in °C |
| 3 | HUMIDITY_OUT | Number | Außen-Luftfeuchtigkeit in % |
| 4 | TEMP_IN | Number | Innentemperatur in °C |
| 5 | HUMIDITY_IN | Number | Innen-Luftfeuchtigkeit in % |
| 6 | CO2 | Number | CO2-Gehalt in ppm |
| 7 | PRESSURE | Number | Luftdruck in mbar |
| 8 | NOISE | Number | Lärmpegel in dB |
| 9 | WIND_STRENGTH | Number | Windgeschwindigkeit in km/h |
| 10 | WIND_ANGLE | Number | Windrichtung in Grad (0-360) |
| 11 | GUST_STRENGTH | Number | Böengeschwindigkeit in km/h |
| 12 | GUST_ANGLE | Number | Böenrichtung in Grad |
| 13 | WIND_DIRECTION | String | Windrichtung als Text (N/NO/O/SO/S/SW/W/NW) |
| 14 | RAIN | Number | Aktueller Regen in mm |
| 15 | RAIN_1H | Number | Regen letzte Stunde in mm |
| 16 | RAIN_24H | Number | Regen letzte 24 Stunden in mm |
| 17 | LAST_UPDATE | String | Zeitstempel der letzten Messung |
| 18 | STATUS | Number | 1 = OK, 0 = Fehler |
| 19 | ERROR | String | Fehlermeldung |

### Unterstützte Module

| Modultyp | ID | Beschreibung |
|----------|-----|--------------|
| Hauptstation | NAMain | Innenmodul (Temperatur, CO2, Luftfeuchtigkeit, Lärm, Druck) |
| Außenmodul | NAModule1 | Außentemperatur und Luftfeuchtigkeit |
| Windmesser | NAModule2 | Windgeschwindigkeit, Windrichtung, Böen |
| Regenmesser | NAModule3 | Regenmenge (aktuell, 1h, 24h) |
| Zusatzmodul | NAModule4 | Weiteres Innenmodul |

### Windrichtung-Umrechnung

| Grad | Richtung |
|------|----------|
| 0-22.5° | N (Nord) |
| 22.5-67.5° | NO (Nordost) |
| 67.5-112.5° | O (Ost) |
| 112.5-157.5° | SO (Südost) |
| 157.5-202.5° | S (Süd) |
| 202.5-247.5° | SW (Südwest) |
| 247.5-292.5° | W (West) |
| 292.5-337.5° | NW (Nordwest) |
| 337.5-360° | N (Nord) |

---

## Konfigurationsbeispiele

### Presence mit OAuth2 TokenManager

```
=== OAuth2 TokenManager (20030) ===
TRIGGER       = 1
CLIENT_ID     = "your_client_id"
CLIENT_SECRET = "your_client_secret"
REFRESH_TOKEN = "your_refresh_token"
TOKEN_URL     = "https://api.netatmo.com/oauth2/token"
SCOPE         = "read_presence access_presence"

=== Netatmo Presence (20031) ===
TRIGGER       = 1  (nach Token-Ready)
ACCESS_TOKEN  = [von 20030 ACCESS_TOKEN]
HOME_ID       = ""  (leer = erstes Home)
CAMERA_ID     = ""  (leer = erste Outdoor-Kamera)
```

### Logik-Verkabelung

```
[Startup] ──► [20030 TRIGGER]
[Konstanten] ──► [20030 Credentials]

[20030 ACCESS_TOKEN] ──► [20031 ACCESS_TOKEN]
[20030 READY = 1] ──► [20031 TRIGGER]

[20031 CAMERA_STATUS] ──► [KNX Statusanzeige]
[20031 LAST_EVENT_TYPE] ──► [KNX Textanzeige]
[20031 LAST_EVENT_SNAPSHOT] ──► [Visualisierung Bild-URL]
[20031 LIGHT_MODE] ──► [KNX Lampen-Feedback]
```

---

## Auto-Refresh

Der Baustein erneuert das Token automatisch **5 Minuten vor Ablauf**:

| Ablauf | Auto-Refresh | Ergebnis |
|--------|--------------|----------|
| 3600 Sek. (1h) | Nach 3300 Sek. | Immer gültiges Token |
| 10800 Sek. (3h) | Nach 10500 Sek. | Immer gültiges Token |

Der Timer wird nach jedem erfolgreichen Refresh neu gestartet.

---

## Debug-Seite

Erreichbar unter: `http://<HomeServer-IP>/hslist`

### OAuth2 TokenManager (20030)

| Feld | Beschreibung |
|------|--------------|
| Baustein ID | 20030 |
| Version | Aktuelle Modulversion (1.1) |
| Status | "Initialisiert" / "Token gueltig" / "Fehler" |
| Token URL | Konfigurierte OAuth2 URL |
| Scope | Angeforderte Berechtigungen |
| Token File | Pfad zur Persistierung |
| Expires In | Token-Gültigkeit in Sekunden |
| Refresh Count | Anzahl der Erneuerungen seit Start |
| Last Refresh | Zeitpunkt der letzten Erneuerung |
| Next Refresh | Geplanter nächster Auto-Refresh |
| Last Error | Letzte Fehlermeldung |

### Netatmo Presence (20031)

| Feld | Beschreibung |
|------|--------------|
| Baustein ID | 20031 |
| Version | Aktuelle Modulversion (1.0) |
| Status | "Initialisiert" / "OK" / "Fehler" |
| Home ID | Erkannte Home ID |
| Camera ID | Erkannte Kamera ID |
| Camera Name | Name der Kamera |
| Camera Status | on / off / disconnected |
| Last Event | Letzter Event-Typ |
| Last Update | Zeitpunkt der letzten Aktualisierung |
| API Calls | Anzahl der API-Aufrufe |
| Last Error | Letzte Fehlermeldung |

---

## Troubleshooting

### HTTP 400: invalid_grant

**Ursache:** Refresh Token ist abgelaufen oder ungültig.

**Lösung:** Neues Refresh Token holen (siehe "Refresh Token erhalten" oben).

### HTTP 400: invalid_scope

**Ursache:** Angeforderte Scopes nicht autorisiert.

**Lösung:** SCOPE-Eingang leer lassen - der Server verwendet dann automatisch die ursprünglich genehmigten Scopes.

### HTTP 401: Unauthorized

**Ursache:** Client ID oder Secret falsch.

**Lösung:** Credentials in der Netatmo Developer Console prüfen.

### HTTP 403: Forbidden

**Ursache:** Scope fehlt oder App nicht autorisiert.

**Lösung:** Neue Autorisierung im Browser durchführen.

### Kein Token nach Trigger

1. Debug-Seite prüfen → "Last Error" anzeigen
2. Token URL korrekt? (`https://api.netatmo.com/oauth2/token`)
3. Alle Pflichtfelder ausgefüllt?

### Token läuft ständig ab

- Auto-Refresh funktioniert nur wenn Modul durchgehend läuft
- Nach Homeserver-Neustart: Trigger erneut senden

---

## Netatmo API Beispiele

Mit dem erhaltenen Access Token können folgende APIs aufgerufen werden:

### Homes Data abrufen

```
GET https://api.netatmo.com/api/homesdata
Authorization: Bearer {ACCESS_TOKEN}
```

### Camera Events abrufen

```
GET https://api.netatmo.com/api/geteventsuntil
    ?home_id=HOME_ID
    &event_id=EVENT_ID
Authorization: Bearer {ACCESS_TOKEN}
```

### Live-Stream URL

```
GET https://api.netatmo.com/api/getcamerapicture
    ?image_id=IMAGE_ID
Authorization: Bearer {ACCESS_TOKEN}
```

---

## Technische Details

### Abhängigkeiten

- Python 2.7 (HLS 2.0 Umgebung)
- `json`, `time`, `os`, `threading`
- `urllib2` (Python 2) / `urllib.request` (Python 3)

### Encoding

- **Eingabe/Ausgabe**: UTF-8
- **API-Kommunikation**: application/x-www-form-urlencoded

### Timeout

- HTTP-Requests: 30 Sekunden

### Change-Only Outputs

Alle Ausgänge werden nur bei Wertänderung aktualisiert, um KNX-Bus-Traffic zu minimieren.

---

## Changelog

### Version 1.1 - Netatmo Homecoach
- **Auto-Refresh Timer** - Konfigurierbares Intervall (default 300 Sek.)
- Timer startet automatisch nach erstem Trigger/Token
- Intervall 0 deaktiviert den Timer

### Version 1.0 - Netatmo Homecoach
- **Temperatur, CO2, Luftfeuchtigkeit** - Alle Raumluftwerte
- **Lärmpegel** - Geräuschpegel in dB
- **Luftdruck** - Atmosphärischer Druck in mbar
- **Gesundheitsindex** - 0-4 mit deutschem Statustext
- **WLAN-Stärke** - Signal-Qualität in Prozent
- Benötigt Scope: `read_homecoach`

### Version 1.8 - Netatmo Presence
- **Event Snapshot URL** - Vollbild-URL des letzten erkannten Events
- **Event Vignette URL** - Thumbnail-URL des letzten Events
- Funktioniert mit `read_presence` Scope (kein `access_camera` nötig)

### Version 2.14 - OAuth2 TokenManager
- **NEU: Authorization Code Eingang** (Pin 8) - Code wird automatisch gegen Tokens getauscht
- **NEU: Auth URL Ausgang** (Pin 7) - erscheint nur wenn Token fehlt/abgelaufen
- Kompletter OAuth2 Flow ohne externe Tools (PowerShell/curl)
- `grant_type=authorization_code` wird intern ausgeführt
- Refresh Token wird automatisch remanent gespeichert
- Auth URL wird nach erfolgreichem Token-Tausch automatisch geleert

### Version 2.13 - OAuth2 TokenManager
- **Auto-Refresh Timer** - Token wird automatisch vor Ablauf erneuert
- **Remanent Storage** - Token überlebt Homeserver-Neustart
- **Inter-Instanz-Kommunikation** - Andere Bausteine können Token direkt abrufen
- **READY-Puls** - 150ms Signal wenn Token bereit

### Version 1.0
- Initiale Version mit OAuth2 Refresh Token Flow
- Debug Panel Integration
- Token-Persistierung
- Netatmo-kompatible Implementierung

---

## Lizenz

MIT License

Copyright 2023-2026

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the "Software"), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.
