# LEARNING-PATH.md — HR Interview Cockpit

Ziel dieses Dokuments: Marco soll dieses Projekt in einem Bewerbungsgespräch
nicht nur vorzeigen, sondern *verteidigen* können — inklusive kritischer
Rückfragen zu Architektur, Sicherheit und Herkunft. Alles hier ist gegen den
tatsächlichen Code in `index.html` (1961 Zeilen, ~140 KB, Stand des einzigen
Commits `e35f511 feat: initial sanitized HR interview cockpit`) geprüft.
Wo etwas im Code nicht eindeutig war, steht das explizit dabei.

---

## 1. Elevator-Pitch (auswendig lernen)

> "Das HR Interview Cockpit ist ein strukturiertes Tool für den kompletten
> Interview-Prozess — von der Stellenanzeige über einen importierbaren
> Fragenpool mit Verhaltensankern bis zum Live-Interview mit Timer und
> Radar-Chart-Auswertung. Die Kerninnovation ist, dass es **zu 100 % im
> Browser läuft, ohne Backend und ohne Build-Schritt**: eine einzige
> `index.html`-Datei, alle Daten bleiben lokal, und selbst der optionale
> KI-Copilot ruft `api.anthropic.com` direkt aus dem Browser auf, mit einem
> Key, den man selbst einträgt. Zielgruppe ist HR/Fachbereich, die ein
> DSGVO-konformes, sofort einsatzbereites Tool ohne IT-Beschaffungsprozess
> brauchen."

---

## 2. Herkunfts-Kontext ehrlich erklären

**Fakten (aus README.md, wörtlich bestätigt):** Das Tool ist als privates
Projekt während eines Bewerbungsprozesses bei Festo entstanden — es gab
**kein Anstellungsverhältnis**. Die veröffentlichte Version ist bereinigt und
umbenannt, enthält nur synthetische Beispieldaten (fiktive Stellenanzeige,
fiktiver Beispielkandidat, selbst verfasste generische Fragenbank). Es sind
keine echten Kandidatendaten, keine echten Stellenanzeigen und keine
Kompetenzmodelle Dritter enthalten. Der Git-Verlauf besteht aus einem
einzigen sauberen Commit — es gibt keine Historie, die versehentlich
sensible Zwischenstände offenlegen könnte.

**Wie man das im Interview sagt (Formulierungsvorschlag):**

> "Das Tool ist ursprünglich in einem Bewerbungsprozess bei Festo entstanden,
> als ich mir überlegt habe, wie ich mein eigenes Interview strukturierter
> vorbereiten könnte — es gab dort kein Anstellungsverhältnis, das Projekt
> ist komplett privat und in meiner Freizeit entstanden. Für mein Portfolio
> habe ich es bereinigt: umbenannt, alle Bezüge zu Festo entfernt, und mit
> ausschließlich synthetischen Beispieldaten versehen — fiktive
> Stellenanzeige, fiktiver Kandidat. Nichts Reales ist enthalten."

**Warum das eine seriöse Vorgehensweise ist statt ein Problem:**
- Es zeigt **Eigeninitiative und Selbstlernbereitschaft** außerhalb eines
  Auftrags — nicht "ich habe firmeneigenen Code mitgenommen".
- Die aktive Bereinigung (Umbenennung, synthetische Daten, keine
  Kompetenzmodelle Dritter) zeigt ein **Bewusstsein für Vertraulichkeit,
  IP und DSGVO** — genau die Sorgfalt, die man von jemandem erwartet, der
  mit sensiblen HR-Daten arbeitet.
- Transparenz von sich aus ("das ist der Kontext, das ist das, was ich
  bereinigt habe") wirkt vertrauenswürdiger als es zu verschweigen und bei
  Nachfrage in Erklärungsnot zu geraten.

---

## 3. Architektur-Überblick

**Ein einziges Dokument, drei Ebenen, alle über HTML-Kommentar-Header
navigierbar:**

- **`<head>` (Zeilen 1–9):** Drei CDN-Skripte, keine lokale Kopie, kein
  Bundler:
  - `xlsx.full.min.js` (SheetJS, Version 0.18.5) — Excel-Import/-Export.
  - `chart.umd.min.js` (Chart.js 4.4.1) — Radar-Charts.
  - `pdf.min.js` (pdf.js 3.11.174) — Text-Extraktion aus CV-PDFs.
- **`<style>` (Zeilen 10–~200):** Ein CSS-Block mit `:root`-Variablen
  (Farbpalette) und thematischen Kommentar-Abschnitten (`/* Kalender */`,
  `/* Auswahl */`, `/* Interview */`, `/* Zusammenfassung */`,
  `/* Vergleich */`). Kein CSS-Framework.
- **`<body>` (Zeilen ~200–483):** Mehrere `<div class="view">`-Container,
  einer pro Schritt (`view-select`, `view-interview`, `view-summary`,
  `view-stellen`, `view-decide`, `view-zweitgespraech` …), per JS
  ein-/ausgeblendet statt Routing/Framework.
- **`<script>` (Zeilen 484–1961):** Ein einziges globales Skript, **kein**
  IIFE/Modul-Wrapper — alle Funktionen und State-Variablen liegen im
  globalen Scope. Navigierbar über durchgehende
  `/* ================= Abschnittsname ================= */`-Kommentare,
  u. a.: Sprache/i18n, Phasen & Daten, State & Helpers, Navigation,
  Kalender, Excel-Import, Auswahl, Vorlagen, KI-Copilot, Kurzanker,
  Interview, Zusammenfassung, Ergebnis & Exporte, KI: Bericht & Mail,
  Auto-Save/Team/DSGVO, IndexedDB, Stellen (Vakanzen), Entscheidung,
  Zweitgespräch, Sync (Team-Ordner), Draft (Absturzsicherung),
  Beispiel-Kandidat (Demo), Reset & Init.

**State-Management-Ansatz:** kein Redux/Signals/Framework — einfache
`let`-Variablen im globalen Scope (`pool`, `guide`, `answers`, `items`,
`stellen`, `ctx`, …, Zeilen 587–598), die von den Render-Funktionen direkt
gelesen und nach jeder Änderung per expliziten `render*()`-Aufrufen neu ins
DOM geschrieben werden (kein virtuelles DOM, kein reaktives Binding —
klassisches "State ändern → komplettes Teilstück neu rendern" per
`innerHTML`-Zuweisung).

**Persistenz-Mechanismen (mehrschichtig, alles clientseitig):**
- `localStorage` für: API-Key, Fragen-Vorlagen (`TPL_KEY`), Kurzanker-Cache
  (`KA_KEY`), Anker-Anzeigemodus, Interview-Archiv (`LS_KEY`), Vakanzen
  (`LS_STELLEN`), Interview-Entwurf/Absturzsicherung (`LS_DRAFT`).
- `IndexedDB` (Zeile 1477, Datenbank `hrcockpit`) — **nur** um den
  `FileSystemDirectoryHandle` eines gewählten lokalen Ordners persistent zu
  halten (File System Access API, `showDirectoryPicker`), damit der
  Browser nach einem Reload wieder in denselben Ordner schreiben kann.
- Optional: direktes Schreiben von JSON-Dateien in einen lokal gewählten
  Ordner (`writeToFolder`, `chooseFolder`) — funktioniert nur in
  Chrome/Edge, mit explizitem Hinweis im Code (Zeile 1404) für andere
  Browser.

---

## 4. Stationen (Kernkonzepte mit Codeverweis)

### Station 1 — xlsx-Fragenpool-Import (zwei Formate, ein Parser-Fallback)
**Codeverweis:** `parseOfficial()` Zeilen 685–724, `handleFile()` Zeilen
731–764, Library: SheetJS `XLSX.read()` / `XLSX.utils.sheet_to_json()`.
**Erklärung:** Der Import versucht zuerst ein "offizielles" Format zu
erkennen (Sheet-Name enthält "matrix", Spalten wie "Underlaying Concept",
"Question in English/German", "Anker 1–4" per Regex auf die Kopfzeile
gesucht — spaltenreihenfolge-unabhängig). Schlägt das fehl, fällt der Code
auf ein generisches Format zurück (Sheet mit "fragen" im Namen oder erstes
Sheet, Spalten "Cluster"/"Frage" per Regex gefunden). Es gibt sogar eine
Heuristik `isRef()`, die erkennt, wenn in einer Zelle nur eine
ID-Referenznummer statt eines echten Fragetexts steht, und löst das über
eine `byId`-Lookup-Tabelle auf. Alles läuft synchron im Browser über
`FileReader.readAsArrayBuffer` — keine Serverkommunikation.
**Selbstkontrollfrage:** Was passiert, wenn eine hochgeladene xlsx-Datei
weder die Spalte "Cluster" noch "Frage" enthält? *(Antwort: `handleFile`
wirft `new Error("Spalten 'Cluster' und 'Frage' nicht gefunden.")`, die in
`setPoolStatus` als Fehlermeldung im UI angezeigt wird — kein stiller
Absturz.)*

### Station 2 — Cluster- & Verhaltensanker-Bewertung (BARS-Prinzip)
**Codeverweis:** `ANKER_TPL` Zeilen 563–568 (Default-Anker für 5 Cluster),
Rendering in `renderItem()` Zeilen 1156–1168, Kurzfassungs-Cache
`fmtAnker()`/`ankerMode` Zeilen 1052–1059.
**Erklärung:** Jede Frage gehört zu einem Cluster (z. B.
"Führungsstärke", "Kommunikationsfähigkeit") und kann bis zu 4
Verhaltensanker haben — je einen Ankertext pro Bewertungsstufe (1–4), im
Stil von Behaviorally Anchored Rating Scales (BARS): konkrete beobachtbare
Verhaltensbeschreibungen statt abstrakter Skalenpunkte. Zusätzlich gibt es
einen "Kurzanker"-Modus: eine KI kann lange Ankertexte auf eine prägnante
Zeile pro Stufe eindampfen (`simplifyAnchors()`, Ergebnis wird in
`localStorage` unter `KA_KEY` gecacht, Original bleibt per Klick erreichbar).
**Selbstkontrollfrage:** Warum wird die KI-Kurzfassung der Anker in
`localStorage` gecacht statt bei jedem Aufruf neu generiert? *(Antwort:
Kostenersparnis bei der API, Konsistenz der Bewertungsmaßstäbe über mehrere
Interviews hinweg, und Fallback ohne Netzwerk/Key — das Original bleibt
immer verfügbar.)*

### Station 3 — Timer & Phasen-Tracker im Live-Cockpit
**Codeverweis:** `PHASES`-Konstante Zeilen 554–560 (start/cv/fach/diag/
rahmen/ende mit geplanter Dauer je Phase), `buildPlan()` Zeile 659,
`startInterview()` Zeilen 1085–1110, `renderPhaseBar()` Zeilen 1113–1120,
laufender Timer via `setInterval` Zeile 1108, Zeitmessung pro Frage in
`saveCurrent()` Zeilen 1122–1130 (`a.sek += ...`).
**Erklärung:** Der Interview-Ablauf ist als feste Sequenz von "Items"
modelliert (`items` Array aus `{t:"check"}`, `{t:"q", q}`, `{t:"form"}`),
die beim Start aus dem gewählten Fragenkatalog zusammengebaut wird
(`startInterview`). Ein `setInterval`-Handle aktualisiert die
Gesamtlaufzeit-Anzeige jede Sekunde. Beim Verlassen einer Frage
(`saveCurrent`) wird die auf dieser Frage verbrachte Zeit in Sekunden
kumuliert. `renderPhaseBar()` zeigt eine Fortschrittsleiste mit
Soll-Zeitfenstern je Phase (aus `buildPlan()`) und markiert die aktuelle
sowie bereits abgeschlossene Phasen.
**Selbstkontrollfrage:** Wird die pro-Frage-Zeit weitergezählt, wenn man
zwischen Fragen vor- und zurücknavigiert? *(Antwort: `qStart` wird bei
jedem `renderItem()`-Aufruf für Fragen neu gesetzt und in `saveCurrent()`
aufaddiert — die Zeit wird also bei jedem Betreten/Verlassen der Frage neu
gemessen und kumulativ aufsummiert, nicht überschrieben.)*

### Station 4 — 4-stufige Bewertungsskala & Live-Erfassung
**Codeverweis:** `setRating()` Zeilen 1182–1190, `toggleSkip()` Zeilen
1191–1197, Keyboard-Shortcut Zeilen 1204–1208 (Tasten 1–4 setzen die
Bewertung direkt), Ratinglabels `rate1..rate4` in den i18n-Objekten
("unzureichend/grundlegend/gut/exzellent", Zeile 505).
**Erklärung:** Jede Antwort bekommt eine von 4 diskreten Stufen (kein
Freitext-Score, keine Kommastellen) plus optionale Notiz und einen
"Überspringen"-Zustand, der sich mit Rating gegenseitig ausschließt
(`a.rating=null` bei Skip, `a.skipped=false` bei Rating). Ein Klick auf die
bereits gewählte Stufe hebt die Bewertung wieder auf (Toggle-Verhalten,
Zeile 1185: `a.rating=a.rating===v?null:v`).
**Selbstkontrollfrage:** Warum wurde eine 4-stufige statt z. B. 5-stufige
Skala gewählt? *(Kein "Mittelwert/neutral"-Ausweichfeld möglich — der/die
Interviewer:in muss sich zwischen "eher schwach" und "eher stark"
entscheiden. Das ist eine bewusste Design-Entscheidung gegen
Zentraltendenz-Bias in Bewertungsskalen — im Code selbst nicht
kommentiert, aber aus der Struktur klar ableitbar.)*

### Station 5 — Radar-Chart-Zusammenfassung (Chart.js)
**Codeverweis:** `clusterAverages()` Zeilen 1211–1221,
`finishInterview()` Zeilen 1222–1270, Chart-Konstruktion Zeilen 1239–1243;
zweite Verwendung in `renderDecide()` (Kandidatenvergleich/Entscheidung,
um Zeile 1702) und im Kandidatenvergleich (`decChart`/`radarChart`
Canvas-Elemente Zeilen 247/421).
**Erklärung:** `clusterAverages()` gruppiert alle beantworteten Fragen nach
Cluster und bildet je Cluster den Mittelwert der Bewertungen (nicht
beantwortete/übersprungene Fragen werden ausgeschlossen, `null` falls
keine Antwort im Cluster existiert). Diese Werte werden direkt als
Chart.js-`radar`-Datensatz übergeben:
```js
new Chart($("radarChart"),{type:"radar",
  data:{labels:Object.keys(cavg),datasets:[{label:$("kName").value,
    data:Object.values(cavg).map(v=>v??0), ...}]},
  options:{responsive:true,maintainAspectRatio:false,animation:false,
    scales:{r:{min:0,max:4,ticks:{stepSize:1}}},plugins:{legend:{display:false}}}});
```
Die Skala ist bewusst auf `min:0,max:4` mit `stepSize:1` fixiert, damit sie
exakt der 4-Punkte-Bewertungsskala entspricht. Vor jedem Neuaufbau wird ein
eventuell vorhandenes altes Chart-Objekt zerstört (`radarObj.destroy()`),
um Speicherlecks/doppelte Canvas-Renderings bei wiederholtem Aufruf zu
vermeiden.
**Selbstkontrollfrage:** Was zeigt das Chart für ein Cluster, in dem gar
keine Frage beantwortet wurde? *(Antwort: `cavg` liefert `null` für dieses
Cluster, `Object.values(cavg).map(v=>v??0)` macht daraus eine `0` im Chart
— optisch nicht von "wirklich schlecht bewertet" unterscheidbar, das ist
eine bekannte Grenze, siehe Abschnitt 5.)*

### Station 6 — Direkter Browser-zu-Claude-API-Call
**Codeverweis:** `getKey()` Zeile 930, `callClaude()` Zeilen 931–944,
Aufrufer u. a. `analyzeAd()` Zeilen 973–999, `simplifyAnchors()` Zeilen
1060–1081, `generateReport()`/`generateMail()` (ab Zeile 1336/1359).
**Erklärung:** Der Call geht direkt gegen
`https://api.anthropic.com/v1/messages` (Model-Konstante `AI_MODEL =
"claude-sonnet-5"`, Zeile 486), mit Headern `x-api-key` (Wert aus
`getKey()`), `anthropic-version: 2023-06-01` und explizit
`anthropic-dangerous-direct-browser-access: true`. Dieser letzte Header ist
technisch notwendig, weil die Anthropic-API standardmäßig
Browser-Origin-Requests aus Sicherheitsgründen blockt (kein CORS für
Server-zu-Server-Auth-Header aus dem Browser) — der Header ist ein
bewusstes Opt-in von Anthropic für genau diesen "kein Backend"-Use-Case,
aber mit dem Namen "dangerous" auch eine ausdrückliche Warnung. Der Key
selbst wird nur in `localStorage` unter `"hr-cockpit-api-key"` gehalten
(`getKey()`/Zeile 1948), nie an einen eigenen Server geschickt — es gibt
schlicht keinen eigenen Server. Ohne Key läuft alles im Demo-Modus über
eine rein lokale Keyword-Heuristik (`localMatch()`, Zeilen 963–971), die
ganz ohne Netzwerk auskommt.
**Sicherheitsimplikation & Umgang im Code:** Ein API-Key, der im Browser
liegt (auch nur in `localStorage`), ist von jedem JS auf derselben Origin
und von jedem mit Gerätezugriff auslesbar — es gibt keine Server-seitige
Geheimhaltung. Der Code entschärft das, so gut es ohne Backend geht: Key
wird nie fest im Quellcode hinterlegt, sondern muss vom Nutzer selbst
eingegeben werden ("bring your own key"); README und UI-Hinweistext
(`keyHint`, Zeile 489: "Wird nur lokal im Browser gespeichert") sagen das
offen; Fehlerbehandlung bei 401 gibt einen expliziten Hinweis
("API-Key prüfen"). Das grundsätzliche Risiko (Keys sind im Client-Kontext
nie wirklich geheim) bleibt aber bestehen — das ist ein bewusster
Trade-off für "kein Backend", kein gelöstes Problem.
**Selbstkontrollfrage:** Was würde ein eigener Backend-Proxy hier lösen,
was der Code aktuell nicht löst? *(Ein Proxy könnte den Key serverseitig
verstecken, Rate-Limits/Kosten zentral steuern und Requests loggen/
validieren — kostet aber genau die Einfachheit ("kein Backend, kein
Hosting"), die der Kern des Projekts ist.)*

### Station 7 — Vakanzen/Stellen, Zweitgespräch & Entscheidungs-Workflow
**Codeverweis:** Abschnitte `/* Stellen (Vakanzen) */` ab Zeile 1481,
`/* Entscheidung */` ab Zeile 1575, `/* Zweitgespräch */` ab Zeile 1584,
inkl. `exportBRListe()` Zeile 1675 (XLSX-Export "BR-Anhörung", also
Betriebsrats-Anhörung nach §99 BetrVG) und `generateBRMail()` Zeile 1728.
**Erklärung:** Das Tool geht über ein Einzel-Interview hinaus: Man kann
"Stellen" (Vakanzen) mit mehreren Kandidat:innen anlegen, pro Kandidat:in
ein Erstgespräch und optional ein strukturiertes Zweitgespräch
durchführen, dann in einer Entscheidungsansicht die Kandidat:innen
vergleichen (Radar-Overlay) und am Ende eine Betriebsrats-Anhörungsliste
als xlsx sowie einen KI-gestützten Mail-Entwurf für die BR-Mitteilung
generieren. Die AGG-Dokumentation ("Sachliche Begründung", i18n-Key
`decBegrL`, Zeile 489) ist im UI-Text ausdrücklich benannt — das Tool
denkt den deutschen Antidiskriminierungs-/Mitbestimmungskontext mit,
nicht nur die reine Interviewlogik.
**Selbstkontrollfrage:** Warum bietet das Tool eine "sachliche
Begründung" als eigenes Pflichtfeld bei der Entscheidung an? *(Weil nach
AGG Ablehnungsentscheidungen im Streitfall sachlich begründbar sein
müssen — das Tool erzwingt/unterstützt Dokumentation genau dafür, nicht
nur "gut/schlecht".)*

---

## 5. Ehrliche Grenzen & Design-Trade-offs

- **"Kein Backend" heißt: Datenpersistenz ist rein clientseitig und damit
  fragil.** Bewertungen landen in `localStorage` (mehrere Keys:
  `LS_KEY`/Archiv, `LS_STELLEN`/Vakanzen, `LS_DRAFT`/Crash-Recovery) sowie
  optional in JSON-Dateien in einem lokal gewählten Ordner (File System
  Access API, nur Chrome/Edge). Es gibt **keine zentrale Datenbank** —
  wenn mehrere Personen im Team interviewen, müssen sie JSON-Dateien
  manuell teilen bzw. über denselben lokalen/Netzwerk-Ordner
  synchronisieren (`scanTeamFolder()`/`connectAndSync()`). `localStorage`
  ist zudem an Browser+Gerät gebunden, hat ein Größenlimit (~5–10 MB) und
  wird bei "Browserdaten löschen" ersatzlos entfernt, wenn kein Ordner
  gewählt wurde.
- **API-Key-Sicherheit:** Wie in Station 6 beschrieben, liegt der Key
  clientseitig in `localStorage` — nachweisbar auslesbar durch jedes Skript
  im selben Origin-Kontext oder durch physischen/Malware-Zugriff auf das
  Gerät. Für ein Einzel-Nutzer-Tool mit selbst eingetragenem Key ist das
  ein akzeptabler, transparent kommunizierter Trade-off; für ein
  Mehrbenutzer-Produktivsystem mit geteiltem/zentral verwaltetem Key wäre
  das **nicht** vertretbar (dann bräuchte es zwingend einen Backend-Proxy).
- **Single-File bei ~140 KB / ~1960 Zeilen:** Navigierbar über
  Kommentar-Header und konsequente Namenskonventionen, aber es gibt keine
  Modulgrenzen, keine Typprüfung, kein Linting/Tests im Code sichtbar, und
  aller State ist global — Namenskollisionen oder versehentliche
  Seiteneffekte sind bei weiterem Wachstum ein reales Risiko. Diff-Reviews
  in einer einzigen 1960-Zeilen-Datei sind unübersichtlicher als in klar
  getrennten Modulen. Der Ansatz skaliert bewusst nicht über "ein
  Werkzeug, ein Autor, kein Team-Codebase" hinaus — passend für ein
  Portfolio-/Solo-Tool, nicht für ein wachsendes Produkt mit mehreren
  Entwickler:innen.
- **Radar-Chart-Grenze:** Cluster ohne beantwortete Fragen werden als `0`
  dargestellt (`v??0`), optisch nicht unterscheidbar von "bewusst mit 0/1
  bewertet" (siehe Station 5) — eine im Code sichtbare, nicht behobene
  Ungenauigkeit.
- **Kein automatisierter Test-Code im Repo sichtbar** (kein `tests/`-
  Ordner, kein Test-Runner im Repo) — Qualitätssicherung erfolgt offenbar
  manuell/durch Nutzung, nicht durch automatisierte Tests. Das ist ein
  Unterschied zu Marcos anderem Portfolio-Projekt (marco-os), das
  `node --test` nutzt.

---

## 6. Recruiter-Simulation (8–10 Fragen mit Musterantworten)

**1. "Warum ein Single-File-Ansatz statt einer normalen Build-Toolchain
(React/Vite/etc.)?"**
> "Weil das Kernziel war: Jeder soll die Datei per Doppelklick oder
> `python -m http.server` sofort öffnen können, ohne `npm install`, ohne
> Build-Schritt, ohne Node-Version-Konflikte. Für ein HR-Tool, das im
> Zweifel auf einem Firmenrechner ohne Entwicklerumgebung laufen soll, ist
> das ein echter Vorteil. Der Preis dafür ist Wartbarkeit bei wachsendem
> Umfang — das ist mir bewusst, und bei einem größeren Team-Projekt würde
> ich das anders aufteilen."

**2. "Warum ruft die App die Anthropic-API direkt aus dem Browser auf statt
über einen eigenen Backend-Proxy?"**
> "Aus demselben Grund: kein Backend heißt kein Hosting, keine Serverkosten,
> keine Wartung eines zweiten Systems. Anthropic erlaubt das explizit über
> den Header `anthropic-dangerous-direct-browser-access` — der Name macht
> aber klar, dass es ein bewusstes Sicherheits-Trade-off ist: Der API-Key
> liegt im Browser-`localStorage` und ist damit nicht wirklich geheim. Für
> ein Tool mit selbst eingetragenem, persönlichem Key ist das vertretbar
> und transparent kommuniziert. Für ein Produkt mit einem zentral
> verwalteten, geteilten Key würde ich das nicht mehr so bauen — dann
> braucht es einen Proxy, der den Key serverseitig hält."

**3. "Wo werden die Bewertungen gespeichert? Was passiert, wenn ich den
Browser-Cache lösche?"**
> "Standardmäßig in `localStorage` — mehrere Keys für Archiv, Vakanzen und
> einen Auto-Save-Entwurf zur Absturzsicherung. Optional kann man einen
> lokalen Ordner wählen (File System Access API, aktuell nur
> Chrome/Edge), dann werden JSONs zusätzlich dort abgelegt. Ohne
> gewählten Ordner gehen die Daten bei einem Cache-Reset verloren — das
> ist eine bewusste Grenze eines reinen Client-Tools ohne Datenbank."

**4. "Wie stellt das Tool sicher, dass die Interview-Bewertung nicht
willkürlich ist?"**
> "Über Verhaltensanker nach BARS-Prinzip: Jede Bewertungsstufe (1–4) hat
> einen konkreten, beobachtbaren Verhaltenstext statt einer abstrakten
> Zahl. Zusätzlich gibt es eine feste 4-Punkte-Skala ohne neutrale
> Mitte, damit man sich entscheiden muss, und ein Pflichtfeld für die
> sachliche Begründung bei der finalen Entscheidung — relevant für
> AGG-Dokumentation."

**5. "Was macht der KI-Copilot genau, und was passiert ohne API-Key?"**
> "Er schlägt aus einer eingefügten Stellenanzeige (und optional einem
> CV) passende Fragen aus dem Pool sowie neue positionsspezifische Fragen
> vor. Ohne Key läuft eine rein lokale Keyword-Matching-Heuristik
> (`localMatch`) — das Tool bleibt also auch ganz ohne KI und Internet
> nutzbar, nur eben weniger intelligent in der Vorauswahl."

**6. "Wie kam es zu diesem Projekt — ist das aus einem früheren
Arbeitgeber-Kontext?"**
> "Es ist während eines Bewerbungsprozesses bei Festo als privates Projekt
> entstanden, ohne dass ein Anstellungsverhältnis bestand — ich habe es
> in meiner Freizeit gebaut, um meine eigene Interviewvorbereitung zu
> strukturieren. Für mein Portfolio habe ich es bereinigt: umbenannt,
> jeglicher Festo-Bezug entfernt, und mit rein synthetischen
> Beispieldaten versehen. Es sind keine echten Kandidatendaten oder
> Kompetenzmodelle enthalten."

**7. "Ist es rechtlich/ethisch unbedenklich, das zu zeigen?"**
> "Ja — es gab kein Anstellungsverhältnis, aus dem ein Verwertungsverbot
> entstehen würde, und die veröffentlichte Version enthält bewusst keine
> echten Daten oder proprietären Inhalte Dritter. Genau deshalb habe ich
> vor der Veröffentlichung aktiv bereinigt, statt es unverändert zu
> lassen."

**8. "Was würdest du an der Architektur ändern, wenn daraus ein
Team-Produkt werden sollte?"**
> "Aufteilen in Module/Komponenten mit eigenem State-Management statt
> globaler `let`-Variablen, ein echtes Backend für zentrale Datenhaltung
> statt `localStorage`/lokaler Ordner, einen Backend-Proxy für den
> API-Key, und automatisierte Tests — aktuell gibt es im Repo keine."

**9. "Warum Chart.js für den Radar-Chart und nicht etwas Eigenes?"**
> "Weil ein Radar-/Netzdiagramm mit mehreren Achsen (den Clustern) und
> fester Skala (0–4) ein Standard-Anwendungsfall ist, für den es keinen
> Sinn macht, das selbst zu bauen — Chart.js liefert das über wenige
> Konfigurationszeilen (`type:'radar'`, `scales.r.min/max/stepSize`).
> Eigenentwicklung hätte hier nur Zeit gekostet, ohne einen Mehrwert für
> das eigentliche Ziel des Tools zu liefern."

**10. "Wie robust ist der xlsx-Import gegenüber unterschiedlichen
Excel-Formaten?"**
> "Es gibt zwei Pfade: ein spezifisches Format mit erkennbaren Spalten wie
> 'Underlaying Concept' und einen generischen Fallback, der per Regex
> nach Spalten wie 'Cluster'/'Frage' sucht — spaltenreihenfolge-
> unabhängig. Fehlt eine Pflichtspalte, wirft der Code einen expliziten
> Fehler mit Meldung im UI statt eines stillen Fehlschlags. Komplett
> beliebige, unstrukturierte Excel-Dateien würde es aber nicht
> interpretieren können — das ist eine bewusste Grenze."

---

## 7. Checkliste — Bist du bereit?

- [ ] Ich kann den Elevator-Pitch (Abschnitt 1) frei und in unter 20
      Sekunden sprechen.
- [ ] Ich kann den Festo-Kontext transparent und ohne zu stocken erklären,
      inklusive "kein Anstellungsverhältnis" und "bereinigt/synthetisch".
- [ ] Ich kann erklären, warum es kein Backend gibt und was das für
      Datenpersistenz und API-Key-Sicherheit bedeutet — inklusive der
      Kehrseite.
- [ ] Ich kann den Weg eines xlsx-Fragenpools vom Upload bis zur
      Anzeige im Interview in eigenen Worten nachzeichnen.
- [ ] Ich kann erklären, wie ein Verhaltensanker (BARS) funktioniert und
      warum die Skala 4 statt z. B. 5 Stufen hat.
- [ ] Ich kann den `callClaude()`-Aufruf technisch erklären: Endpoint,
      Header (insbesondere `anthropic-dangerous-direct-browser-access`),
      wo der Key liegt, was ohne Key passiert.
- [ ] Ich kann erklären, wie der Radar-Chart aus den Cluster-Mittelwerten
      aufgebaut wird und wo seine Schwäche liegt (0 vs. unbeantwortet).
- [ ] Ich kann mindestens zwei "Warum X und nicht Y"-Fragen (Single-File
      vs. Build-Tool, direkter API-Call vs. Backend-Proxy) souverän mit
      Trade-off-Argumentation beantworten, ohne die Entscheidung zu
      verteidigen als sei sie alternativlos.
- [ ] Ich kann mindestens 3 ehrliche Grenzen des Tools benennen, ohne dass
      es wie Selbstsabotage klingt — sondern als Zeichen von technischer
      Reife und Selbstreflexion.
