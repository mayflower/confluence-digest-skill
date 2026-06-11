# Confluence-Digest – Design

**Datum:** 2026-06-11
**Status:** Umgesetzt bis **Stufe 2a** inkl. Blogpost-Erfassung & Subagenten-Fan-out (> 72h).
Dieses Dokument ist der ursprüngliche Design-Entwurf; **verbindlich für das aktuelle Verhalten sind
`SKILL.md` + `README.md`**. Die technischen Eckdaten unten wurden an den umgesetzten Stand angeglichen.

## Zweck

Ein personalisierter Überblick über für mich relevante, kürzlich geänderte Confluence-Inhalte
(mayflowergmbh), um trotz hoher täglicher Aktivität informiert zu bleiben – ohne manuell viele
Spaces durchzusehen.

## Relevanz-Signale

Cross-space (kein fixer Space-Filter), Inhaltstypen **Seiten und Blogposts** (`type in (page, blogpost)`),
getrieben von vier Signalen:

- **Mich betreffend** – Seiten/Blogposts, in denen ich erwähnt werde (Mention) oder die ich mit-bearbeitet habe.
- **Themen/Keywords** – Treffer zu konfigurierten Begriffen: Volltext (`signals.keywords`, `text ~`) **oder**
  nur Titel (`signals.titleKeywords`, `title ~` – schmal, gegen Footer-/Adress-Rauschen).
- **Verfolgte Personen** – Aktivität bestimmter Kolleg:innen (`signals.people`, `{name, id}`).

Stufe 1 aktivierte nur „mich betreffend"; Keywords kamen in Stufe 1.5, Personen in Stufe 2a.
**Teams gestrichen** (werden bei Mayflower nicht in Confluence gepflegt).

## Architektur

Drei klar getrennte Bausteine (Trennung ist Voraussetzung für die spätere Verteilung):

1. **Skill/Command (Logik)** – generisch, enthält keine persönlichen Daten. Das verteilbare Artefakt.
2. **Nutzer-Config (Daten)** – pro Person lokal, gitignored. Keywords, Personen, Feintuning.
3. **Identität** – Atlassian-Account-ID, beim ersten Lauf automatisch via `atlassianUserInfo` ermittelt.

### Ausbaustufen

1. **Stufe 1 (jetzt):** Slash-Command `/confluence-digest`, Ausgabe im Chat, Relevanz = „mich betreffend"
   (Mentions + eigene Bearbeitungen). Mini-Onboarding: Identität automatisch erkennen, Config schreiben.
2. **Stufe 1.5:** Erststart-Hybrid-Interview + Keyword-Signal. Aktivitäts-Analyse schlägt Keywords vor,
   Nutzer:in bestätigt/ergänzt; Keywords fließen in den Digest. `setup`-Befehl zum Re-Konfigurieren.
3. **Stufe 2:** Personen-Signal (inkl. Personen-Teil des Interviews), geplanter Lauf (Cloud-Agent),
   Ausgabe als Datei/Confluence-Seite. (Teams gestrichen – werden bei Mayflower nicht in Confluence gepflegt.)
   - **Subagenten-Fan-out für große Fenster (> 72h): ✅ umgesetzt** – Such-Queries und
     Seiten-Zusammenfassungen werden je Query bzw. je Seite an einen Subagenten ausgelagert (kompakte
     Rückgabe, schlanker Hauptkontext, nebenläufig); ≤ 72h und `--dry-run` bleiben inline. Merge/Dedupe/
     Ranking bleiben zentral. Plan: `docs/plans/2026-06-11-confluence-digest-subagent-fanout-plan.md`.
     Der geplante Lauf (Cloud-Agent) bleibt zurückgestellt (Stufe 2b).
4. **Stufe 3 (Vision):** macOS-Desktop-App auf Basis derselben Logik.

## Konfiguration

Nutzer-Config (z.B. `config.local.yaml`, gitignored), beim ersten Lauf aus Vorlage erzeugt:

```yaml
cloudId: "<cloud-id>"           # eigene Confluence-cloudId (Platzhalter in der Vorlage)
accountId: auto                 # beim 1. Lauf via atlassianUserInfo gesetzt
signals:
  mentions: true                # Seiten/Blogposts, in denen ich erwähnt werde
  ownEdits: true                # Seiten/Blogposts, die ich mit-bearbeitet habe
  keywords: []                  # Volltext-Keywords (text ~), per setup befüllt
  titleKeywords: []             # nur-Titel-Keywords (title ~) – schmal
  people:   []                  # verfolgte Personen als {name, id}-Einträge
limits:
  highlights: 5                 # Anzahl ausführlicher Highlights (2–4 Sätze)
  groupSummaries: 8             # max Kurz-Summaries (2–3 Sätze) je Restgruppe
onboarding:
  hintShown: false              # einmaliger setup-Hinweis bei leeren Keywords
```

### Signal → CQL Mapping

Jeweils kombiniert mit `lastmodified >= now("-<Fenster>")` und `type in (page, blogpost)`,
`ORDER BY lastmodified DESC`. Mentions/eigene nutzen `currentUser()`:

| Signal               | CQL-Kern                                    |
|----------------------|---------------------------------------------|
| Mentions             | `mention = currentUser()`                   |
| Eigene Bearbeitungen | `contributor = currentUser()`               |
| Keywords (Volltext)  | `text ~ "<keyword>"` (1 Query je Keyword)   |
| Keywords (nur Titel) | `title ~ "<keyword>"` (1 Query je Keyword)  |
| Verfolgte Personen   | `contributor = "<id>"` (1 Query je Person)  |

(Pro Person eine eigene Query → Attribution „von wem". Teams gestrichen.)

## Erststart-Onboarding (Interview)

Statt einer leeren Config wird beim Erststart ein geführtes Interview die Config befüllen – senkt
die Einstiegshürde, besonders für die spätere Verteilung an Kolleg:innen. Manuelles Editieren bleibt
parallel möglich.

**Trigger:** Kein Config-File vorhanden → vor dem ersten Digest läuft das Interview.

**Ablauf (hybrid – Vorschläge aus Aktivität + freie Ergänzung):**

1. Begrüßung; `accountId` via `atlassianUserInfo` ermitteln und bestätigen lassen („Du bist \<Name\>?").
2. **Aktivitäts-Analyse** (MCP): Mentions + eigene Bearbeitungen der letzten ~30–90 Tage abrufen →
   daraus ableiten: häufige Mit-Autor:innen (Personen-Vorschläge) und häufige Begriffe aus
   Titeln/Labels (Keyword-Vorschläge).
3. **Dialog:** „Vorgeschlagene Themen: […] – welche übernehmen? Weitere ergänzen?"; analog für Personen.
   Nutzer:in bestätigt/korrigiert/ergänzt frei.
4. Defaults: `mentions`/`ownEdits` = an, `maxSummaries` = Standard.
5. Config schreiben, dann **direkt ersten Digest anbieten**.

**Re-Konfiguration:** `/confluence-digest setup` führt das Interview erneut (aktuelle Werte als
Default); die Config-Datei bleibt frei editierbar.

**Edge Cases:** Aktivitäts-Analyse leer (neue/wenig aktive Person) → elegant auf „nur fragen"
zurückfallen, Vorschläge entfallen. Abbruch → Minimal-Config (nur `mentions`/`ownEdits`), später
per `setup` ergänzbar.

**Staffelung:** Stufe 1 enthält nur das Mini-Onboarding (Identität erkennen + Config schreiben).
Das volle Hybrid-Interview liefert Mehrwert erst mit den Keyword-Signalen → kommt in **Stufe 1.5**
(Keywords), der Personen-Teil des Interviews folgt mit **Stufe 2**.

## Ablauf

### Zeitfenster (aus „heute" + optionalem Argument, keine State-Datei)

- Standard: letzte **24h**
- Heute = Montag: letzte **72h** (Fr–Mo automatisch)
- Override: `/confluence-digest 7d` → letzte 7 Tage (z.B. nach Urlaub)
- CQL: `lastmodified >= "YYYY-MM-DD HH:mm"`

### Schritte

1. **Config laden.** Fehlt sie → aus Vorlage erzeugen, `accountId` via `atlassianUserInfo` ermitteln und schreiben (einmalig).
2. **Fenster berechnen** (Wochentag + Override).
3. **CQL-Queries bauen** (eine pro aktivem Signal) und über `searchConfluenceUsingCql` (cloudId = mayflower) ausführen.
4. **Mergen & dedupen** nach Page-ID; pro Seite merken, *welche* Signale getroffen haben (für Ranking + Gruppierung).
5. **Ranken** nach Signal-Priorität:
   - P1: Mentions
   - P2: eigene Bearbeitungen
   - P3: Keyword-Volltreffer
   - P4: verfolgte Personen
   - Mehrfach-Treffer steigen (Mention + Keyword > nur Keyword).
6. **Zusammenfassen** via `getConfluencePage`: die `limits.highlights` Highlights ausführlich (2–4 Sätze),
   je Restgruppe bis `limits.groupSummaries` Einträge kurz (2–3 Sätze); darüber nur Titel+Link.
7. **Rendern** (siehe Output).

**Ausführung:** Fenster ≤ 72h → alles inline; **> 72h → Subagenten-Fan-out** (je Such-Query und je
zu summender Seite ein Subagent, kompakte Rückgabe); `--dry-run` immer inline.

### Volumen-Schutz

Liefert eine Query auffällig viele Treffer (zu breites Keyword), auf die neuesten kappen und im
Output hinweisen („+12 weitere zu ‚Symfony' – Keyword evtl. zu breit"). Hält Lauf bezahlbar und
Digest lesbar.

## Output-Format

Markdown, Ausgabe im Chat (Stufe 1):

```
# Confluence-Digest · <Zeitraum>
<Kopfzeile: X relevante Seiten aus Y Spaces>

## ⭐ Highlights (Top `limits.highlights`, Default 5)
### <Seitentitel> · <Space> · <Autor>, <wann>
<2–4 Sätze KI-Zusammenfassung>
**Warum für dich:** <z.B. "Du wirst erwähnt" / "Keyword ‚DSGVO'">
🔗 <Link>

## Nach Signal gruppiert  (Priorität: Mentions > ownEdits > Themen > Personen)
### 🔔 Dich betreffend (Mentions)
- <Titel> · <Space> · <wann> — <2–3 Sätze Summary, bis groupSummaries> 🔗
### ✏️ Von dir mitbearbeitet
- ...
### 🏷️ Deine Themen
- ... (+N weitere ohne Zusammenfassung; bei zu breitem Keyword zusätzlich Volumen-Notiz)
### 👥 Verfolgte Personen
- <Titel> · <Space> · <wann> *(von <name>)* — <2–3 Sätze> 🔗
```

### Regeln

- Jede Seite **nur einmal** – als Highlight *oder* in genau einer Gruppe (höchste Priorität).
- **Highlights** = Top-gerankte Treffer mit ausführlicher Zusammenfassung; Gruppenliste knapp.
- **Leere Gruppen** weglassen. Nichts Neues → freundliche „Nichts Neues im Zeitraum"-Zeile.
- **Zeit-/Autorangaben** relativ („vor 3 Std.", „gestern").

Spätere Stufen: dasselbe Markdown zusätzlich in Datei (`digests/YYYY-MM-DD.md`) oder Confluence-Seite –
Render-Funktion bleibt, nur Ausgabeziel wechselt.

## Fehlerbehandlung

- **MCP nicht verbunden / Auth abgelaufen:** klare Meldung mit Hinweis „`/mcp` → `atlassian-mayflower` authentifizieren", sauberer Abbruch.
- **`accountId` nicht ermittelbar:** keyword-/personenbasierte Signale laufen weiter; „mich betreffend" wird mit Hinweis übersprungen.
- **Einzelne CQL-Query schlägt fehl:** überspringen, Rest läuft, Hinweis im Output.
- **Seiteninhalt nicht ladbar** (Rechte/gelöscht): Eintrag ohne Zusammenfassung, nur Titel+Link mit Notiz.

## Bewusste Grenzen (YAGNI)

- Kein echter Versions-Diff (MCP-Limit) – Zusammenfassung beschreibt aktuellen Stand.
- Keine State-Datei (Fenster aus Datum).
- Keine Teams (gestrichen). Keywords/Personen werden per `setup`-Interview gepflegt (umgesetzt).
- **Ausschluss-Filter (`exclude`): ✅ umgesetzt** – konfigurierbarer `exclude.spaces` (`space not in (...)`)
  + `exclude.titlePatterns` (`title !~`), an alle Signal-Queries angehängt; Seiten-Kopien („Kopie von"/„Copy of") immer per Default ausgeschlossen.

## Test

- **Trockenlauf (`--dry-run`):** zeigt generierte CQL-Queries + Treffer*zahl* pro Signal, ohne
  Inhalte zu holen/zusammenzufassen. Schnell, billig, gut zum Justieren von Keywords.
- Manuelle Verifikation gegen bekanntes Beispiel (Seite, in der ich sicher erwähnt wurde).

## Verteilung an Kolleg:innen

- Command/Skill enthält null persönliche Daten → einfach weitergeben (Plugin/Skill-Ordner oder Git-Repo).
- Jede:r bekommt beim ersten Lauf eigene Config + auto-erkannte Account-ID.
- Voraussetzung dokumentieren: eigener `atlassian-mayflower`-MCP eingerichtet & authentifiziert.

## Bekannte Einschränkungen / Backlog

- **Label-/Tag-Signal (Backlog):** Ein eigenes Signal für verfolgte Confluence-Labels
  (`label in ("tag1","tag2")`) wäre denkbar – kuratierte Tags als Relevanzquelle. Vertagt; greift
  nur, wo konsequent gelabelt wird. (Blogposts werden bereits über `type in (page, blogpost)`
  abgedeckt, unabhängig von Labels.)

- **„zuletzt von" zeigt die Ersteller:in, nicht die letzte bearbeitende Person.** Stufe 1 nutzt
  `history.createdBy.displayName`. Die letzte Änderung steckt in `version.authorId` (nur ID, kein
  Name in der Such-Antwort). Verbesserung (Stufe 1.5/2): `version.authorId` zu einem Namen auflösen
  und als echtes „zuletzt geändert von" anzeigen. Live bestätigt am 2026-06-11.

## Geklärte Punkte (ehem. offen)

- CQL-Unterstützung von `mention`/`contributor`/`text ~`/`title ~` gegen die reale Instanz **bestätigt**.
- Relative Zeitangaben kommen fertig aus `friendlyLastModified`.
- Nutzer-Config liegt **neben `SKILL.md`** (gitignored `config.local.yaml`).
