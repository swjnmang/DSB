# DSB
DSB DHWS
# DSB Kalender-Spots (TermineSuS.html / AlleTermine.html) – Notizen

## Problem (Juli 2026)
Kalender in den DSB-HTML-Spots luden nicht mehr ("Lade Kalender..." blieb ewig stehen).

## Root Causes (mehrschichtig)
1. ICS-Export von Google Calendar hat keine CORS-Header → Direct-Fetch aus dem Browser
   funktioniert nie, es braucht immer einen Proxy/eine API mit CORS-Support.
2. Die bisherigen CORS-Proxies (Cloudflare Worker, github.com/cors-anywhere-Link) waren
   entweder nie funktionsfähig oder ausgefallen.
3. Schul-Firewalls blockieren proxy-artige Drittanbieter-Domains (workers.dev,
   corsproxy.io etc.) häufig als "Proxy/Anonymizer" – googleapis.com dagegen so gut wie nie.
4. Das eigentliche DSB-Anzeigegerät nutzt eine ALTE Browser-Engine ohne `AbortController`
   (deshalb sind fetch-/Promise-Polyfills im Code) – Code darf sich NICHT auf moderne
   Web-APIs wie AbortController verlassen.

## Lösung
Umstieg von ICS-Export + ical.js-Parsing + Proxy-Kaskade auf die
**Google Calendar API v3 (JSON)**, die native CORS-Header liefert – kein Proxy mehr nötig.
- Endpoint: `https://www.googleapis.com/calendar/v3/calendars/{calendarId}/events?key={KEY}&timeMin=...&timeMax=...&singleEvents=true&orderBy=startTime`
- `singleEvents=true` löst wiederkehrende Termine serverseitig auf → keine manuelle
  RRULE-Iteration mehr nötig, ical.js komplett entfernt.
- Timeout-Handling bewusst OHNE `AbortController` gebaut (nur `setTimeout` + `Promise`),
  wegen der alten Geräte-Engine.
- Debug-Log-Box bleibt im Normalbetrieb immer ausgeblendet (nur Konsolen-Logs).
- Bei komplettem Ladefehler erscheint eine sichtbare Warnung statt eines stillen
  leeren Kalenders.

## Setup
- Google Cloud Console → Projekt → "Google Calendar API" aktivieren → API-Key erstellen
  → HTTP-Verweis-Einschränkung auf `https://swjnmang.github.io/*`
- Key steht im Code als `GOOGLE_API_KEY` ganz am Anfang des `<script>`-Blocks.
- Gehostet über GitHub Pages im Repo `swjnmang/DSB`, in DSBControl als
  **Internetseiten-Spot** (nicht HTML-Spot-Upload!) eingebunden – wichtig, weil ein
  HTML-Spot-Upload/lokale Datei einen "null"-Origin hat, den manche CORS-Konfigurationen
  ablehnen.

## Layout / Ordnungsdienst-Logik
Bewusst unverändert gelassen: komplettes CSS/Layout in TermineSuS.html.

Die Ordnungsdienst-Klassenzuordnung basiert nicht mehr auf Kalender-Titel-
Parsing + Schüler-CSV, sondern auf einem festen Zeitplan `ordnungsdienstZeitplan`
(Kalenderwoche -> Klasse) gemäß dem Aushang der Schule. Siehe `getISOWeek` und
`displayTodayOrdnungsdienst` in TermineSuS.html. Schülernamen werden nicht mehr
angezeigt.
