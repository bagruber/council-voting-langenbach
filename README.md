# Gemeinderat Langenbach – Sitzungsprotokoll

> **Archiviert.** Der Gemeinderat Langenbach ist seit August 2026 ein
> Mandant im Hauptprojekt
> [bagruber/council-voting-tool](https://github.com/bagruber/council-voting-tool)
> und läuft dort aktuell unter
> [bagruber.github.io/council-voting-tool/?rat=langenbach](https://bagruber.github.io/council-voting-tool/?rat=langenbach).
> Dieses Repo ist der eingefrorene Stand des früheren Forks.

Statische Single-Page-App zur digitalen Schriftführung von Gemeinderatssitzungen: Anwesenheiten, Abstimmungen, Ereignis-Log und Export. Läuft komplett im Browser, ohne Backend, ohne Build-Schritt.

Live: <https://bagruber.github.io/council-voting-langenbach/>

## Tech-Stack

| Schicht | Wahl | Anmerkung |
| --- | --- | --- |
| Markup-Host | [index.html](index.html) | Lädt alle CDN-Skripte, definiert das Tailwind-Theme (Farben der Langenbacher CI), enthält App-spezifisches CSS. |
| UI-Runtime | **React 18** (UMD) | Vom unpkg-CDN, kein npm/Bundler. |
| JSX-Transform | **Babel Standalone** | Transpiliert [js/app.js](js/app.js) im Browser zur Ladezeit (~1 s Initialkosten). |
| Styling | **Tailwind Play CDN** | Theme inline in `index.html` konfiguriert, keine Build-Pipeline. |
| Export-ZIP | **JSZip** | Optional für Multi-File-Exports. |
| Daten-Layer | [js/data.js](js/data.js) | Reines IIFE, exportiert `window.COUNCIL_DATA`. Verarbeitet [members.json](members.json) und liefert Helfer für aktive Mitglieder, Sitzplatzordnung und Gremien-Konfiguration. |
| App-Logik | [js/app.js](js/app.js) | React-Komponenten + `useReducer` für den gesamten Sitzungs-State. |
| Stammdaten | [members.json](members.json) | Parteien, Mitglieder (mit Profilen, Amtszeiten, Partei-Historie) und Gremien (aktuell nur das Plenum). |

Keine Server-Komponente, keine Datenbank, keine Auth.

## Komponenten und ihr Zusammenspiel

```
   members.json
        │
        ▼
   js/data.js  ──►  window.COUNCIL_DATA
   (Parsing,        (processRawData, getActiveMembers,
   Datum-Logik)      buildSeatOrder, getBodyConfig)
        │
        ▼
   js/app.js
   ┌────────────────────────────────────────────┐
   │  useReducer-State                          │
   │    ├── session  (Titel, Datum, Status…)    │
   │    ├── seatStates (anwesend/vertretung/…)  │
   │    ├── currentVote / votes                 │
   │    ├── agenda                              │
   │    └── log (chronologisches Event-Array)   │
   │                                            │
   │  React-Komponenten                         │
   │    ├── Header / Body-Auswahl               │
   │    ├── Sitzungs-Halbkreis (Sitze)          │
   │    ├── Abstimmungs-Panel                   │
   │    ├── Tagesordnung & Live-Log             │
   │    └── Export-Panel (JSON / MD / TXT / ZIP)│
   └────────────────────────────────────────────┘
        │
        ▼
   localStorage  ◄── Auto-Backup pro Reducer-Tick
                    (Schlüssel `council-session-backup`,
                     beim Start als Wiederherstellungs-Dialog)
```

**Ablauf einer Sitzung**

1. Beim Seitenladen fetcht der App-Bootstrap `members.json`, übergibt es an `COUNCIL_DATA.processRawData` und ermittelt mit `getActiveMembers(date)` und `getBodyConfig(body, activeMembers)` die zum Sitzungsdatum aktive Besetzung.
2. Der Reducer initialisiert die Sitzplätze (alle anwesend in der Plenums-Sicht). Jede Nutzer-Aktion (Anwesenheit ändern, Abstimmung starten, Tagesordnungs­punkt hinzufügen…) dispatcht eine Reducer-Action und erzeugt im Event-Log einen zeitgestempelten Eintrag.
3. Bei jeder State-Änderung wird der vollständige Sitzungs-State nach `localStorage` geschrieben. Lädt jemand die Seite erneut, bietet die App die Wiederherstellung an.
4. Am Ende lässt sich die Sitzung in mehreren Formaten exportieren; sensible/„nicht öffentliche" Phasen sind im Log markiert und werden im öffentlichen Export entsprechend behandelt.

## Stammdaten pflegen

Alle stadt-/gemeindespezifischen Inhalte stehen in [members.json](members.json):

- **`parties`** – Anzeigename, primäre Fraktionsfarbe (`color`), heller Akzent (`accent`).
- **`seatOrder`** – Reihenfolge der Fraktionen im Halbkreis (links → rechts).
- **`members`** – Personen mit `from`/`to` (Amtszeit), `partyHistory`, `role` (`mayor`/`councillor`), optionalem `profile` (Pronomen, Kontakt, Referate, Ausschüsse).
- **`bodies`** – Gremien. In Langenbach aktuell nur das Plenum; das Datenmodell unterstützt aber Ausschüsse mit Vorsitz, Stellvertretungen und Sitzpaaren (Hauptmitglied + Vertretung).

Datumsbereiche sind im Format `YYYY-MM` oder `YYYY-MM-DD`. `to: null` heißt „weiterhin aktiv".

## Deployment

Da nichts gebaut werden muss, ist das gesamte Repository das Deployment-Artefakt. Auf einen beliebigen statischen Host kopieren reicht.

- **GitHub Pages**: aktuell aktiv, Quelle ist `main` / `/`.
- **Netlify / Cloudflare Pages / Vercel**: Repo verbinden, kein Build Command, Publish Directory `/`.
- **Eigener Webserver (nginx, Apache, Caddy…)**: Dateien servieren, korrekte MIME-Types für `.js` und `.json` sicherstellen.
- **Offline-Betrieb**: Die vier CDN-Skripte (React, ReactDOM, Babel Standalone, Tailwind Play, JSZip) müssen dann lokal mitgeliefert und die `<script src>`-Pfade in [index.html](index.html) angepasst werden.
- **Unterpfad-Hosting** (z. B. `example.com/langenbach/`) funktioniert unverändert, weil alle internen Pfade relativ sind.

Für eine "richtige" Produktionspipeline würde man Tailwind via CLI vorab kompilieren und JSX mit esbuild/Vite transpilieren — beides aktuell nicht nötig.

## Lokale Entwicklung

Jeder beliebige statische Server tut es, weil `members.json` per `fetch` geladen wird (und damit über `file://` an CORS scheitert):

```bash
# Python 3
python -m http.server 8000

# Node
npx serve .
```

Dann <http://localhost:8000> öffnen.

## Lizenz / Herkunft

Fork von `council-voting-tool` (ursprünglich für Moosburg) als Startpunkt für Langenbach.
