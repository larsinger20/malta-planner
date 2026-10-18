# Malta Planner

Gemeinsame Reiseplanungs-Webseite für eine Urlaubsgruppe (Malta, 31.10.–07.11.2026). Die Gruppe schlägt Aktivitäten und Essensideen vor, trägt sich ein, stimmt ab, führt Einkaufslisten und teilt Kosten. Alles in Echtzeit synchron. Die Seite wird live von der Gruppe benutzt.

## Über den Nutzer
- Lars (GitHub: larsinger20) betreut die Seite. **Kein Programmierer**: Erklärungen Schritt für Schritt, einfach und **auf Deutsch**, ohne Fachjargon.
- Nutzt meist das **iPhone (Safari)**, teilweise Windows-Desktop. Mobile-first denken (getestet bei 375 px Breite).
- Alle Texte auf der Seite sind **Deutsch** und in der **Wir-Form** („unsere Villa“, „unsere Flüge“). Lars gehört selbst zur Gruppe.

## Goldene Regeln (immer einhalten)
1. **Erst testen, dann zeigen, dann nach Lars' ausdrücklichem OK hochladen.** Nie ungefragt pushen.
2. **Nie Testdaten in die echte Datenbank schreiben.** Die Gruppe nutzt sie live. Getestet wird nur in einer Testkopie mit Beispieldaten (siehe „Testen“).
3. **Änderungen an echten Daten** (Korrekturen, Migrationen) nur mit Lars' ausdrücklichem OK, danach nachlesen und bestätigen.
4. **Vor jedem Push `git fetch`**, bei Bedarf `git pull --rebase`. Lars ändert manchmal direkt auf GitHub; fremde Änderungen ansehen und erhalten.
5. **Bestehende Daten nicht brechen**: Alte Dokumente haben evtl. neue Felder nicht. Gespeicherte Werte nie für die Optik ändern (z. B. bleibt `time` = „Morgens / Vormittags“, angezeigt wird „Vormittags“).
6. Änderungen klein halten. Lars entscheidet über Design-Geschmack; bei Unsicherheit kurz Varianten zeigen statt raten.
7. Commit-Nachrichten auf Deutsch, kurz und verständlich.

## Architektur
- **Eine einzige Datei: `index.html`** (HTML + CSS + ein JS-Modul). Kein Framework, keine Build-Tools, kein npm. Deployment = `index.html` committen und pushen.
- **Hosting**: GitHub Pages aus `larsinger20/malta-planner`, Branch `main`, Root. Nach dem Push ist die Seite nach ca. 1–2 Minuten live.
- **Datenbank**: Firebase Firestore per CDN-Import (`firebasejs/10.12.2`, modulares SDK). Die Web-Config steht in `index.html` (bei Web-Apps üblich).
- **Weitere Dienste**: Google Maps JavaScript API + Places API (New) für Karte und Ortsvorschläge (`MAPS_KEY`, auf die Domain beschränkt, liefert lokal nichts), Open-Meteo (Wetter + Marine, ohne Schlüssel).
- **Identität**: zufällige Geräte-ID in `localStorage.uid`. Anmeldung mit Name + PIN; mit gleichem Namen + PIN lässt sich das Profil auf einem neuen Gerät wiederherstellen.
- **Symbole**: `malta-32.png`, `malta-180.png`, `malta-touch-180.png`, `favicon.ico` (intern BMP-Format), `flag.png` (echte Flagge, gemeinfrei). Bei neuen Icons neue Dateinamen statt `?v=`-Anhänge (Safari-Cache). `rel="icon"` nie auf das Touch-Icon zeigen lassen (Firefox nimmt sonst den Sand-Hintergrund).

## Datenmodell (Firestore)
Drei Sammlungen: `activities`, `people`, `signups`. **Neue Sammlungen funktionieren nicht ohne Freigabe in der Firebase-Konsole** (macht Lars), neue Felder schon.
- `people/{uid}`: `{ name, ph, seen?, col?, tour?, tips? }`. Profil-Schreibvorgänge **immer mit `{ merge: true }`**. `col` = Profilfarbe aus `PALETTE`. Namen sind eindeutig (ohne Groß/Klein).
- `activities/{id}`: `{ title, desc, day, time, cat: 'act'|'food', kind: 'cook'|'rest'|'', place?, pid?, lat?, lng?, link?, res?: {t,p}|null, ts, by: uid, shop?, cm? }`
  - Alte Dokumente ohne `cat` gelten als `'act'`. `link` ist nur Altbestand (wird angezeigt, nicht mehr eingegeben).
  - `day`: Montag–Sonntag, `time`: Ganztags / „Morgens / Vormittags“ / Mittags / Nachmittags / Abends / Nachts. Beides optional.
- `signups/{activityId}_{uid}`: `{ act, uid }` = „dabei“ (Aktivität) bzw. Stimme 👍 (Essen).
- Sonder-Dokumente in `activities`: `tripinfo` (`cat:'info'`, Reise-Info), `shoplist` (`cat:'list'`, allgemeine Einkaufsliste), `stats` (`cat:'stats'`, Aufrufzähler), Ausgaben `x…` (`cat:'cost'`, `amt` in Cent, `by`, `for[]`, `kind:'pay'` = Ausgleich). **Beim Filtern nie `(a.cat || 'act')` ohne Kategorieprüfung**, sonst tauchen Sonder-Dokumente als Vorschläge auf.
- `shop` und `cm` (Einkaufsliste, Kommentare) sind Maps `{ id: {...} }`: **Einträge immer einzeln** per `setDoc(..., { shop: { [id]: {...} } }, { merge: true })` schreiben bzw. per `updateDoc` + `deleteField()` löschen, nie die ganze Map überschreiben.

## Funktionen (Überblick)
- Countdown-Kachel mit drei Phasen (vor Abflug / „Wir sind auf Malta“ / „Wieder zu Hause“), aufklappbare **Reise-Info** (Villa, Check-in/-out, Flüge, Wetter-Streifen). Bearbeiten der Reise-Info nur für den Admin. Keine Tür- oder Schlüsselcodes eintragen.
- Vier Reiter: **Aktivitäten**, **Essen** (Kochen/Restaurant, Abstimmen mit 👍, Favorit), **Einkauf** (Listen + Umschalter „Liste | Kosten“ mit Ausgleichsrechnung), **Mein Plan** (Kalender: eigene Aktivitäten + alle Essen mit festem Tag; Tipp auf Eintrag springt zur Kachel und lässt sie aufleuchten).
- Karten mit ⋯-Menü (Bearbeiten, Ort, Kommentar, Ausgabe, Löschen). Ersteller wird automatisch eingetragen.
- Ortsvorschläge per Google, Karte aller Orte (Google Maps, eigener Hell/Dunkel-Schalter), Einkaufs-Vorschläge aus eingebauter Rezeptliste `RECIPES`.
- Profilfarben (jede nur einmal), Tour für neue Nutzer (7 Schritte, `TOUR`) + kleine Tipps beim ersten Besuch (`TIPS`), Statistik für den Admin, Easter Egg (Tipp auf Flagge).
- Bewusst **nicht** gewollt bzw. verworfen: Detail-Blatt/Overlay pro Karte („zu overengineered“), Preisstufen bei Restaurants, Malta-Sperre bei Ortsvorschlägen, WLAN/Mietwagen/Notruf in der Reise-Info, Chips statt Dropdowns für Tag/Tageszeit, Push-Benachrichtigungen, Home-Bildschirm-Werbung.

## Design-Regeln
- Stil: **Apple-Formen + Urlaubsfarben**. Sand `#f6f4ef` / dunkel `#0b0f10`, Karten `#fff` / `#20282b` mit feinem Rahmen, Radius 20 px, Akzent **Meerestürkis** (`--acb`/`--ac`). Alle Farben nur über CSS-Variablen in `:root`; **Dunkelmodus immer mitpflegen und prüfen**.
- **Einheitlichkeit ist wichtig**: Alle Kacheln (`.c`) gleich (Padding 18/18/16, Titel 20 px/600, Infotext 15 px `--mu`). Neue Elemente an Bestehendes angleichen.
- **Liquid Glass nur für schwebende Bedienelemente** (Reiter, Umschalter, Karte-Knopf, Menüs, Dialog, Hinweise, Ortsvorschläge, Karten-Bedienung), nie für Inhalte.
- **Linien-Symbole (SVG) in der Bedienung**, Emojis nur in Inhalten. Kein 🇲🇹-Emoji (Windows zeigt „MT“), stattdessen `flag.png`.
- Knöpfe sind Pillen; Hauptknopf vorher hell mit Rahmen, danach grün gefüllt. Seltene Aktionen gehören ins ⋯-Menü.
- Rückmeldungen per `toast()`, **kein `alert()`/`confirm()`**; Sicherheitsfragen über den Glas-Dialog `await ask({ t, m, ok, danger })`.
- Animationen dezent (Lars mag Wobbeln/Glas, aber nicht übertrieben), immer `prefers-reduced-motion` beachten. Animationen nur beim auslösenden Element abspielen, nicht bei jedem Neuaufbau.
- In Texten für Nutzer die Vorschlagskacheln nie „Karte“ nennen (Verwechslung mit der Landkarte).

- **Aufklapp-Zeilen** (Kommentare, Vorschläge, Einkaufsliste bei Kochvorschlägen, Reise-Info): ganze Zeile antippbar (volle Breite, min. 44 px), Text bleibt in beiden Zuständen gleich, nur ein Linien-Pfeil dreht sich (`chev(key, open)`, Klasse `.op` am Knopf, `chevMark(key)` vor dem Neuaufbau; zum Zuklappen mit Gleiten `.cl`). Alle gleiten per `slideHeight` auf/zu; auch zugeklappt steckt der Knopf in seiner Hülle (`.cmts`/`.sug`/`.shop`), sonst springt der Abstand. Vorschläge: Zusatz („Urlaubs-Basics …“ / „Passt zu …“) klein im Kasten (`.sgl`).
- **Einkauf-Animationen**: Vorschlag antippen → Chip schrumpft weg, neuer Eintrag gleitet herein (`justAdded`, `.si.new`, auch bei getippten Einträgen); Löschen → Zeile schrumpft weg, ein zurückkehrender Vorschlags-Chip poppt auf (`justBack`, `.chip.back`). Grundsatz: Animationen nur für das gerade angetippte Element über kurzlebige Merker, nie bei jedem Neuaufbau.
- **Wetter** (Reise-Info): ruhige, gleich hohe Wisch-Kacheln (Tag, Symbol, max/min, Regen nur ab 20 %), Tag antippbar (`wxSel`, `.wxd.sel`, ohne Neuaufbau), darunter Detailkasten `#wxdet` mit Datum + Sonnenauf-/-untergang (`sunrise`/`sunset` von Open-Meteo) und vier festen Spalten Regen · Wind · Wellen · Wasser („folgt“, solange die Meeresvorhersage nicht reicht). Angetippte Kachel scrollt ganz ins Bild, Wisch-Position bleibt bei Live-Updates erhalten. Lars fand eine Liste pro Tag zu lang.

- **Robustheit (Paket 1, 18.10.2026, nach E2E-Test mit Agenten)**:
  - Beträge über `parseAmt()`: Tausenderpunkt („2.000“, „1.234,56“) richtig, Unklares abgelehnt statt geraten.
  - Kein Doppelspeichern bei schlechtem Netz: Formulare schließen/leeren **sofort** (nicht erst nach `await`), feste IDs (`costForm.id`), Sperren (`costBusy`, `goBusy`), Chips/„Alle“ `disabled`. Ausgleich prüft nach der Frage per `settle(balances())`, ob er noch offen ist.
  - Eingaben überleben Live-Updates: `render()` sichert alle Felder in `#list` (`keepFields`/`restoreFields`, inkl. `dataset` und `#ek`), `freshForm = true` beim Öffnen eines Formulars, damit keine alten Werte übernommen werden. Reise-Info-Bearbeiten wird nur durch eigene Aktionen neu gebaut.
  - Keine Geister-Vorschläge: vor Schreiben in bestehende Vorschläge `alive(id)` prüfen (sonst Hinweis `GONE`); Dokumente ohne `title` werden nie als Vorschlag gezeigt.
  - Kosten: Stand-Zeile „Bezahlt: … · Dein Anteil: … · Ausgleich erhalten/Überwiesen: …“ (`eur0` = ganze Euro ohne „,00“), freie Überweisung („Andere Überweisung eintragen“, `costForm.pay`), Empfänger darf Ausgleichsbuchungen löschen.
  - „Name ändern“ mit Abbrechen und Prüfung des bisherigen PINs; leerer Name gibt Hinweis.
  - Fußzeile: Knopf rutscht nur bei Platzmangel unter die Namen; ab 5 Leuten 3 Kreise + „+N“ (`.stack i.more`).
  - Schmale Handys (`@media(max-width:360px)`), `color-scheme`, Rot-Flächen über `--rdb`, `.kseg .on` mit `--segon`.
  - Einkauf: neue Zeile öffnet sich einmal weich in der Höhe (`growNew()`, Merker `justAdded` wird beim ersten Zeichnen verbraucht), Leuchten klingt aus (`siGlow`). Vorschlagsformular gleitet auf/zu (`closeAdd()`, `addAnim`).
  - Noch offen (Paket 2/3): Abflug-Text bis zur Landung, Vorschlags-Teilwort-Treffer (Flammkuchen → Backpulver), Tippflächen < 44 px, Kachel-Kontrast, Google-Logo unter „Ohne Kartenpunkt“, Profilmenü schließt bei Snapshots, Tour aus Profilmenü während Farbwahl, VoiceOver-Angaben, CSS-Aufräumen.

## Fallstricke im Code
- `render()` wird übersprungen, solange ein Eingabefeld Fokus hat (`pending`), und beim `focusout` nachgeholt. Eigene Aktionen rufen `render(true)`.
- `$('list').onclick` ist ein großer Verteiler: **neue Knöpfe in der Liste brauchen einen eigenen Fall** (oder früh `return`), sonst landen sie beim Mitmachen-Code (der ist mit `if (!a) return;` abgesichert).
- **CSS-Klassennamen vor dem Anlegen suchen** (z. B. `.bar` gehört der Statistik). Breite Selektoren wie `.pdh>div>small` können ungewollt andere Teile treffen.
- `[hidden]{display:none!important}` steht im CSS, damit `hidden` auch bei Flex-Elementen wirkt.

- **Eigene Einträge mit Fokus im Feld** (Kommentar, Einkauf): Die echte Firestore meldet die lokale Änderung erst kurz nach `setDoc`, und `render()` ist bei Fokus gesperrt. Deshalb nach dem Schreiben `await Promise.race([w, 80 ms])` und dann `render(true)` (+ Sicherheitsnetz). Test-Stub dafür mit asynchronen Snapshots (20 ms) und Server-Antwort (500 ms) bauen, sonst fällt so ein Fehler nicht auf (Bug 18.10.: Kommentar erschien erst nach Neuladen).

## Testen
- Node.js ist auf Lars' PC nicht installiert, **Python ist da**. Testkopie: `index.html` in einen Arbeitsordner kopieren und dort die Firebase-Imports durch einen kleinen Stub mit Beispieldaten im Speicher ersetzen (gleiche Funktionsnamen: `collection`, `doc`, `setDoc`, `updateDoc`, `deleteDoc`, `deleteField`, `increment`, `onSnapshot`), dazu Stubs für `google.maps` und Open-Meteo. Nie die echte Datenbank ansprechen.
- Ausliefern mit einem Python-Server **ohne Cache** (`Cache-Control: no-store`), sonst zeigt der Browser alte Fassungen.
- Prüfen in 375 px Breite, hell **und** dunkel. Die Testumgebung spielt Animationen oft nicht ab: Werte und Abläufe per JavaScript nachmessen statt nur Screenshots zu vertrauen.
- **Cloud-Sitzung (z. B. vom iPhone)**: Dort gibt es kein Browser-Fenster für Lars. Kleine Änderungen (Texte, Farben, kleine Korrekturen) sind ok, mit Test per Skript und kurzer Beschreibung für Lars. Größere Umbauten lieber am PC mit sichtbarer Testversion. Auch in der Cloud gilt: erst nach Lars' OK pushen.

## Mögliche nächste Features
- „Wer kocht?“ / Koch-Team bei Kochvorschlägen
- „Wer bringt was mit?“-Liste, Umfragen, Heute-Ansicht während des Urlaubs
- Ernährungsangaben/Allergien im Profil
- Rückblick nach dem Urlaub
