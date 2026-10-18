# Malta Planner

Gemeinsame Reiseplanungs-Seite einer Urlaubsgruppe (Malta, 31.10.–07.11.2026, ~10 Leute). Vorschläge für Aktivitäten und Essen, eintragen/abstimmen, Kommentare, Einkaufslisten, Kosten teilen, alles in Echtzeit. **Wird live benutzt.** Die Gruppe plant meist spontan: Die meisten Vorschläge haben keinen Tag/keine Uhrzeit. Die Seite ist Ideensammlung + Abstimmung, kein Terminkalender.

## Über Lars
- Betreut die Seite (GitHub `larsinger20`), **kein Programmierer**: auf Deutsch, einfach, Schritt für Schritt, ohne Fachjargon.
- Meist **iPhone (Safari, auch als Web-App vom Home-Bildschirm)**, teils Windows-PC. Mobile-first (375 px).
- Seitentexte Deutsch in der **Wir-Form** („unsere Villa“). Vorschlagskacheln nie „Karte“ nennen (Verwechslung mit der Landkarte).
- Entscheidet Design-Geschmack selbst; bei Unsicherheit kurz Varianten zeigen. Mag dezentes Wobbeln/Glas, nichts Übertriebenes.

## Goldene Regeln
1. **Erst gründlich testen, dann zeigen, dann nach Lars' ausdrücklichem OK hochladen.** Nie ungefragt pushen. **Beim Testen nie sparen.**
2. **Nie Testdaten in die echte Datenbank.** Getestet wird nur in einer Testkopie (siehe „Testen“). Lesen der echten Daten zur Fehlersuche ist ok.
3. Änderungen an echten Daten nur mit Lars' OK, danach nachlesen.
4. Vor jedem Push `git fetch` (Lars ändert manchmal direkt auf GitHub), fremde Änderungen erhalten.
5. Bestehende Daten nie brechen (alte Dokumente haben neue Felder evtl. nicht); gespeicherte Werte nie für die Optik ändern (`time` bleibt „Morgens / Vormittags“, angezeigt „Vormittags“).
6. Änderungen klein halten. Commit-Nachrichten Deutsch, kurz.
7. Bei nennenswerten Änderungen diese Datei mitpflegen (Lars nutzt auch Cloud-Sitzungen vom iPhone).

## Architektur
- **Eine Datei `index.html`** (HTML + CSS + ein JS-Modul), kein Framework/Build/npm. Deploy = committen + pushen (GitHub Pages, `main`, Root, live nach 1–2 Min.; GitHub cacht 10 Min.).
- Firestore per CDN (`firebasejs/10.12.2`, modular), Web-Config in `index.html`. Google Maps JS + Places API (New) (`MAPS_KEY`, nur für die Domain, lokal ohne Ergebnis). Open-Meteo (Wetter + Marine, ohne Schlüssel).
- Identität: Geräte-ID `localStorage.uid`; Anmeldung Name + PIN, gleiche Daten stellen das Profil auf neuem Gerät wieder her (Web-App vom Home-Bildschirm hat eigenen Speicher → dort einmal anmelden).
- Icons: `malta-32.png`, `malta-180.png`, `malta-touch-180.png`, `favicon.ico` (BMP), `flag.png`. Neue Icons = neue Dateinamen. `rel="icon"` nie aufs Touch-Icon.

## Datenmodell (Firestore)
Nur drei Sammlungen: `activities`, `people`, `signups` (neue Sammlungen bräuchten Freigabe in der Konsole durch Lars; neue Felder gehen).
- `people/{uid}`: `{ name, ph, col, seen, tour, tips{key}, trip, sn{id}, cmr{id}, nbase }`. **Immer `{ merge: true }`.** Namen eindeutig (ohne Groß/Klein). `col` aus `PALETTE`, jede Farbe nur einmal.
- `activities/{id}`: `{ title, desc, day, time, cat:'act'|'food', kind:'cook'|'rest'|'', place, pid, lat, lng, link(Altbestand), res{t,p}, ts, by, shop{}, cm{} }`. Ohne `cat` = `'act'`.
- `signups/{actId}_{uid}`: `{ act, uid }` = dabei bzw. 👍-Stimme.
- Sonder-Dokumente in `activities`: `tripinfo` (`cat:'info'`), `shoplist` (`cat:'list'`), `stats` (`cat:'stats'`), Ausgaben `x…` (`cat:'cost'`, `amt` in Cent, `by`, `for[]`, `kind:'pay'` = Ausgleich). **Nie `(a.cat || 'act')` ohne Kategorieprüfung filtern**, Dokumente ohne `title` nie als Vorschlag zeigen.
- Maps `shop`/`cm`/`sn`/`cmr`: Einträge **einzeln** per `setDoc(..., {shop:{[id]:…}}, {merge:true})`, Löschen per `updateDoc` + `deleteField()`. Nie ganze Maps überschreiben.

## Funktionen (Kurzüberblick)
- **Countdown-Kachel** (`countdown()`): vor Abflug Tage/Std./Min. → „✈️ Wir fliegen“ bis `LAND` → „Wir sind auf Malta · Tag X von 8“ → nach `RETURN` Rückblick (Aktivitäten, Essen, „€ geteilt“). Antippen öffnet die **Reise-Info** in der Kachel: Glas-Umschalter Wetter · Villa · An-/Abreise (öffnet immer mit Wetter), Wetter-Wischkacheln mit Detailkasten, Villa mit Airbnb-Link, Flüge als Bordkarten, Autos zum Flughafen mit persönlichem Hinweis. Nur Admin bearbeitet. **Keine Tür-/Schlüsselcodes eintragen** (Datenbank offen).
- **Reiter** Aktivitäten · Essen · Einkauf · Mein Plan. Reiterwechsel springt nach oben und klappt die Reise-Info zu.
- **Kacheln** (`.card`): Titel + ⋯-Menü (Bearbeiten, Ort, Reservierung, Kommentar, Ausgabe, Löschen), Infozeile ohne Emojis, Beschreibung, Ort/Website/Reservierung, Einkaufsliste (Kochen), Kommentare, Fußzeile mit Kreisen + Namen + Hauptknopf („🙋 Ich bin dabei“ bzw. 👍; vorher hell mit Rahmen, danach grün). Ersteller wird automatisch eingetragen. Essen nach Stimmen sortiert, „🏆 Favorit“ nur bei alleiniger Führung. Kochen ohne Ort zeigt die Villa.
- **„Neu“**: Vorschlag anderer bleibt „Neu“, bis er wirklich im Bild war (≥ halb sichtbar, 0,8 s; `IntersectionObserver`, `people.sn`); Pille bleibt bis zum Reiterwechsel. Kommentare „N neu“ bis zum Aufklappen (`people.cmr`). Oranger Punkt am Reiter bei Ungesehenem **oder** Ungelesenem (Lars: so lassen). Älter als `NB0` bzw. `nbase` (Registrierung) = gesehen.
- **Einkauf**: allgemeine Liste + Listen aller Kochvorschläge, Vorschläge aus `RECIPES` (Abgleich über den Titel; Treffer, die nur Teil eines längeren sind, fallen weg). Umschalter Liste | Kosten: Stand, Balken, wenige Überweisungen (`balances`/`settle`), Beträge über `parseAmt()` („1.234,56“).
- **Mein Plan**: eigene Aktivitäten + alle Essen mit festem Tag, nach Tag/Tageszeit; Tipp springt zur Kachel (`goToCard`, Aufleuchten).
- **Karte**: Google Maps Vollbild (Knopf in Aktivitäten, Essen, Mein Plan; Filter startet passend: Aktivitäten/Essen/Meine), eigener Hell/Dunkel-Schalter, Googles Orts-Symbole aus. Pins = eigene HTML-Ebene (`pinLayer`, eine `OverlayView`, keine Google-Marker): farbige Perle mit weißem Symbol (`GP_IC`), Größe nach Zoom (`pinSize`), Namen ab Zoom 13 nur wo Platz (`pinLabels`), Überlappende → Perle mit Zahl, Tipp zoomt (`pinCluster`), was die Villa verdeckt, rückt daneben; eigene mit goldenem Rand; angetippt hebt sich die Perle als Stecknadel (`.sel`). Fenster: Luftlinie zur Villa, „Zum Vorschlag“ (`mapGo` → `goToCard`, erst nach dem Zurück im Verlauf), „Route ab Villa ↗“. „N ohne Kartenpunkt“ aufklappbar, Namen springen zur Kachel. **Ortsvorschläge** per Google beim Tippen (Malta bevorzugt, nicht gesperrt).
- Profil: Farbe, Name ändern (mit PIN), Tour (`TOUR`, 7 Schritte) + Tipps beim ersten Besuch (`TIPS`), Statistik (Admin). Easter Egg: Tipp auf Flagge.
- **Update-Hinweis**: prüft ETag von `index.html` beim Zurückkommen/alle 15 Min. → „✨ Neue Version · tippen zum Aktualisieren“ (lädt mit `?v=`). Web-App: `html.pwa` → Verlauf hinter der Reiter-Leiste.
- **Bewusst nicht gewollt**: Detail-Blatt/Overlay pro Kachel, Preisstufen, Malta-Sperre bei Orten, WLAN/Mietwagen/Notruf in der Reise-Info, Chips statt Dropdowns für Tag/Tageszeit, Push-Benachrichtigungen, Home-Bildschirm-Werbung, „Heute“-Ansicht, „Wer kocht?“, Funktionen, die auf festen Tagen/Uhrzeiten aufbauen.

## Design
- **Apple-Formen + Urlaubsfarben**: Sand `#f6f4ef` / dunkel `#0b0f10`, Kacheln `#fff` / `#20282b` mit feinem Rahmen, Radius 20, Akzent Meerestürkis (`--acb`/`--ac`). Farben nur über CSS-Variablen; **Dunkelmodus immer mitpflegen**.
- **Einheitlichkeit**: alle Kacheln `.c` Padding 18/18/16, Titel 20 px/600, Infotext 15 px `--mu`. Neues an Bestehendes angleichen.
- **Liquid Glass nur für schwebende Bedienelemente** (Reiter, Umschalter, Karte-Knopf, Menüs, Dialog, Hinweise), nie für Inhalte. Umschalter = gleitende Perle mit dezentem Wobbeln, Position per CSS `translate` (nicht `transform`).
- Linien-Symbole (SVG) in der Bedienung, Emojis nur in Inhalten. Kein 🇲🇹-Emoji (Windows zeigt „MT“) → `flag.png`.
- Knöpfe = Pillen, seltene Aktionen ins ⋯-Menü. Rückmeldung per `toast()`, Sicherheitsfragen per `await ask({t,m,ok,danger})`, **nie `alert`/`confirm`**.
- Tippflächen mind. 44 px (kleine Knöpfe über unsichtbares `::after` bzw. transparenten Rand vergrößern, Optik gleich).
- **Animationen**: dezent, `prefers-reduced-motion` beachten, nur für das auslösende Element (kurzlebige Merker wie `justAdded`, `justTicked`, `cmNew`), nie bei jedem Neuaufbau. Einheitliche Dauern: großes Aufklappen 0,34/0,26 s, kleines (`slideHeight`) ab 0,32/0,24 s, wächst mit der Länge bis 0,56 s; alle Umschalter 0,42 s. Kurven: auf `cubic-bezier(.45,0,.15,1)`, zu `cubic-bezier(.4,0,.2,1)`.
- **Aufklapp-Zeilen** (Kommentare, Vorschläge, Einkaufsliste, Reise-Info): ganze Zeile antippbar, Text bleibt gleich, nur der Pfeil dreht sich (`chev`/`chevMark`); auch zugeklappt in ihrer Hülle (`.cmts`/`.sug`/`.shop`), sonst springt der Abstand. Kopfzeile bleibt beim Gleiten ruhig, Inhalt blendet gestaffelt ein; lange Kommentarlisten: Seite gleitet im selben Takt mit (`glideScroll`).

## Fallstricke im Code
- `render()` wartet, solange ein Feld in der Liste Fokus hat (`pending`) oder etwas gleitet (`slides`/`slideWait`), und holt es danach nach. Eigene Aktionen rufen `render(true)`. Eingaben überleben Neuaufbauten (`keepFields`/`restoreFields`; `freshForm` beim Öffnen).
- Eigene Einträge mit Fokus im Feld: Firestore meldet die lokale Änderung asynchron → nach dem Schreiben `await Promise.race([w, 80 ms])`, dann `render(true)`.
- Schreiben ins Profil (z. B. `readCm`) löst einen People-Snapshot + Neuaufbau aus; laufende Animationen nur dank `slideWait` geschützt.
- `$('list').onclick` ist ein großer Verteiler: **neue Knöpfe brauchen einen eigenen Fall** bzw. frühes `return` (sonst Mitmachen-Code; abgesichert mit `if (!a) return;`).
- Vor Schreiben in bestehende Vorschläge `alive(id)` prüfen. Formulare sofort schließen/leeren (kein Doppelspeichern bei schlechtem Netz).
- **CSS-Klassennamen vor dem Anlegen suchen** (`.bar` = Statistik, Kosten-Balken `.kbar`). Breite Selektoren treffen leicht Fremdes. `background`-Kurzform setzt `background-clip` zurück.
- `[hidden]{display:none!important}` steht im CSS. Profilmenü `#pm` wird bei Live-Updates nicht neu gebaut (`top()` frischt nur auf).

## Testen (gründlich, nie daran sparen)
- Node fehlt, **Python ist da**. Testkopie im Scratchpad: Firebase-Imports durch Stub mit Beispieldaten ersetzen (`collection`, `doc`, `setDoc`, `updateDoc`, `deleteDoc`, `deleteField`, `increment`, `onSnapshot`), Stubs für `google.maps` und Open-Meteo. **Stub realistisch asynchron** (Snapshot nach ~20 ms, Antwort nach ~500 ms), sonst fallen Timing-Fehler nicht auf. Zeit testen über `localStorage.fakeNow`.
- Server ohne Cache (`Cache-Control: no-store`). Prüfen bei 375 und 320 px, hell **und** dunkel, ohne seitliches Scrollen, ohne Konsolenfehler.
- Abläufe per JavaScript nachmessen (Höhen pro Bild, Zustände, gespeicherte Daten), Screenshots fürs Aussehen. Das Test-Fenster zeichnet ohne GPU: große Bewegungen ruckeln dort, Klick-Rechenzeit separat messen.
- Cloud-Sitzung (iPhone): kein Browser-Fenster für Lars → nur kleine Änderungen, per Skript testen, kurz beschreiben, erst nach OK pushen.

## Offene Kleinigkeiten / Ideen
- Paket 3: Kachel-Kontrast, Tour aus dem Profilmenü während der Farbwahl, VoiceOver-Angaben, CSS aufräumen.
- Ideen: „Wer bringt was mit?“, Umfragen, Ernährungsangaben/Allergien im Profil.
