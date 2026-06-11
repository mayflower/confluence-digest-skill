---
name: confluence-digest
description: Use when the user runs /confluence-digest (optionally with a time window `Nh`/`Nd`, z.B. 24h, 72h, 7d) to get a prioritized overview of recently changed Confluence pages (mayflowergmbh) relevant to them — pages they are mentioned on or have contributed to, plus optional topic keywords — with AI summaries; `/confluence-digest setup` runs an interview to maintain those topics/keywords. Triggers: /confluence-digest, Confluence-Überblick, was ist neu in Confluence, Confluence-Digest, confluence-digest setup, eigene Themen/Keywords pflegen.
---

# Confluence-Digest (Stufe 1.5)

Priorisierter Überblick über kürzlich geänderte, für die Nutzer:in relevante Confluence-Seiten
auf `mayflowergmbh`. Signal „mich betreffend" (Mentions + eigene Bearbeitungen) plus optionale
Themen-Keywords („Deine Themen").

**Datenquelle:** MCP-Server `atlassian-mayflower` (Tools `mcp__atlassian-mayflower__*`).
**Konfiguration:** `config.local.yaml` im Skill-Verzeichnis (nutzer-lokal, gitignored).
**Design:** `docs/plans/2026-06-11-confluence-digest-design.md`

## Aufruf

```
/confluence-digest            → Standardfenster (24h; montags 72h)
/confluence-digest <Fenster>  → Override-Fenster `Nh`/`Nd` (z.B. 24h, 72h, 7d – etwa nach Urlaub)
/confluence-digest --dry-run  → nur CQL + Trefferzahlen, ohne Inhalte/Zusammenfassungen
/confluence-digest setup      → Onboarding-Interview: eigene Themen (Keywords) pflegen
```

**Argument-Routing:** Lautet das Argument `setup`, springe direkt in den Abschnitt
„## setup / Onboarding-Interview" und führe **nicht** den normalen Digest-Ablauf aus.
Alle anderen Argumente (`<Fenster>`, `--dry-run`, kein Argument) laufen wie unten beschrieben.

## Ablauf

### 1. Config laden / Onboarding
- Das Skill-Verzeichnis wird über einen Symlink erreicht
  (`~/.claude/skills/confluence-digest` → echtes Repo). Lies und schreibe `config.local.yaml`
  **neben `SKILL.md` im aufgelösten echten Verzeichnis** (z.B. via `realpath`), damit die Datei
  im gitignored Repo landet und nicht im Symlink-Pfad.
- Prüfe, ob `config.local.yaml` dort existiert.
- **Fehlt sie (NEUE Nutzer:in):** Begrüße kurz, erzeuge `config.local.yaml` aus
  `config.example.yaml`. Führe dann **das volle Onboarding-Interview** aus dem Abschnitt
  „## setup / Onboarding-Interview" durch (Identitätsbestätigung via `atlassianUserInfo`
  inkl. Rückschreiben von `accountId: <account_id>` **und** der Keyword-Schritt). Das ersetzt
  das frühere Mini-Onboarding der Stufe 1. Im Anschluss bietet das Interview an, direkt einen
  Digest zu laufen.
- **Existiert sie:** lade Werte. Steht `accountId: auto`, einmalig via `atlassianUserInfo`
  auflösen und zurückschreiben. Fehlt der Schlüssel `onboarding` (Bestandsconfig aus Stufe 1),
  behandle `onboarding.hintShown` als `false`.
- Der aufgelöste `accountId` wird **für spätere Stufen** gespeichert; Stufe 1 nutzt in der CQL
  `currentUser()` und verbraucht ihn nicht.
- Schlägt `atlassianUserInfo` fehl → siehe Abschnitt Fehlerbehandlung.

### 2. Zeitfenster bestimmen
- Lies das aktuelle Datum aus dem Harness-Kontext (Feld „Today's date").
- Optionales Argument hat Vorrang, Form `Nh`/`Nd` (Beispiele `24h`, `72h`, `7d`):
  `Nh` → `now("-Nh")`; `Nd` → `now("-Nd")`.
- Kein Argument: ist heute **Montag** → `now("-72h")` (Fr–Mo), sonst → `now("-24h")`.
- Merke dir das gewählte Fenster als Label für die Ausgabe. Labeling-Regel:
  - Standard 24h → „letzte 24h"; Montags-Standard 72h → „Fr–Mo".
  - Sonst ohne Spezial-Label: bei `Nh` → „letzte N Stunden", bei `Nd` → „letzte N Tage".
- Bei `--dry-run`: Argument-Parsing identisch, nur spätere Schritte unterscheiden sich.

### 3. Relevante Seiten holen
Für jedes aktive Signal eine CQL-Abfrage via `mcp__atlassian-mayflower__searchConfluenceUsingCql`
(cloudId aus Config), `limit (= 50)`, jeweils mit dem Fenster aus Schritt 2:

- mentions:  `mention = currentUser() AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`
- ownEdits:  `contributor = currentUser() AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`

(Nur Signale ausführen, die in der Config `true` sind.)

**Keyword-Signal (Stufe 1.5):** Es gibt zwei Keyword-Listen, beide erzeugen dasselbe Signal
`keyword` (Gruppe „Deine Themen"); sie unterscheiden sich nur im CQL-Match:

- **`signals.keywords`** (Volltext, auch im Body): pro Eintrag
  `text ~ "<kw>" AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`
- **`signals.titleKeywords`** (nur Titel – schmal, gegen Footer-/Adress-Rauschen): pro Eintrag
  `title ~ "<kw>" AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`

Beide jeweils `limit (= 50)`, gleiches Fenster aus Schritt 2; `expand` nicht nötig. Führe pro
Eintrag in **beiden** nicht-leeren Listen je eine Abfrage aus. Sind beide Listen leer, entfällt
dieser Schritt. (Tipp: breite Begriffe, die als Ortsname/Adresse überall im Body vorkommen –
z.B. der eigene Standort – gehören in `titleKeywords`, nicht in `keywords`.)

**Antwort-Format (wichtig – nach Bedeutung mappen, nicht auf einen festen Pfad verlassen):**
`searchConfluenceUsingCql` liefert je nach Fall eine von zwei Formen. Lies die Felder nach
folgender Bedeutung, und nutze pro Feld den ersten vorhandenen Pfad:

| Bedeutung | Pfad (rohes Search-Format) | Pfad (vereinfachtes Format) |
|-----------|----------------------------|------------------------------|
| Trefferliste | `results[]` | `content.nodes[]` |
| Gesamtzahl | `totalSize` | `content.totalCount` |
| Seiten-ID | `<item>.content.id` | `<item>.id` |
| Titel | `<item>.title` | `<item>.title` |
| Space-Name | `<item>.resultGlobalContainer.title` | `<item>.space.name` |
| Autor/letzte:r Bearbeiter:in | `<item>.content.history.createdBy.displayName` | `<item>.author.displayName` |
| relatives Datum | `<item>.friendlyLastModified` | `<item>.lastModified` |
| URL | `_links.base` + `<item>.url` (relativ → absolut) | `<item>.webUrl` |

Die absolute URL im rohen Format = `_links.base` (z.B. `https://mayflowergmbh.atlassian.net/wiki`)
+ `<item>.url`.

### 4. Mergen & dedupen
- Sammle alle Treffer aller Abfragen (mentions, ownEdits, je Keyword). Schlüssel = Seiten-ID.
- Pro Seite merke die Menge der Treffer-Signale (eine Seite kann mehrere Signale tragen, z.B.
  mention UND ownEdit UND keyword).
- Keyword-Treffer mit dem Signal `keyword` vermerken **plus, welches Keyword** sie ausgelöst hat
  (aus `signals.keywords` ODER `signals.titleKeywords`; eine Seite kann von mehreren Keywords
  getroffen werden → alle merken).
- Merke je Query die Gesamtzahl (`totalSize` bzw. `content.totalCount`); ist sie `> limit`,
  für die Volumen-Notiz vormerken (`N = Gesamtzahl - limit`, also `Gesamtzahl - 50`). Das gilt
  je mention-/ownEdit-Query **und je Query pro Keyword** (aus beiden Listen; Gesamtzahl pro
  Keyword separat führen).

### 5. Ranken
Score je Seite: Mention = 4, ownEdit = 2, keyword = 1, summiert über die Treffer-Signale.
Mehrere Keyword-Treffer auf derselben Seite zählen weiterhin als **ein** keyword-Beitrag (= 1).
Beispiele:
- Mention + ownEdit = 6
- nur Mention = 4
- ownEdit + keyword = 3
- nur ownEdit = 2
- nur keyword = 1

Sekundärsortierung: Aktualität (Reihenfolge aus `ORDER BY lastmodified DESC`).

### 6. Highlights & Gruppen aufteilen
- **Highlights:** `H = min(limits.highlights (Default 5), Anzahl gerankter Seiten)` – die
  Top-Seiten nach Score/Aktualität. Diese bekommen in §7 eine **ausführliche** Summary (2–4 Sätze).
- **Gruppen:** Alle übrigen gerankten Seiten (Rang > H) werden nach §8 in ihre Gruppe einsortiert
  (höchstpriores Signal). Pro Gruppe bekommen die ersten `limits.groupSummaries` (Default 8)
  Einträge (nach Ranking/Aktualität) in §7 eine **kürzere** Summary (2–3 Sätze); weitere Einträge
  derselben Gruppe nur Titel + Link + Datum (siehe §8).
- Es werden also nur die Seiten via `getConfluencePage` geholt, die auch eine Summary bekommen
  (Highlights + die je Gruppe bis zur Obergrenze) – keine überflüssigen Fetches.

### 7. Zusammenfassen
Zusammengefasst werden (a) die H **Highlights** und (b) je Gruppe die ersten
`limits.groupSummaries` Einträge (siehe §6). Für jede dieser Seiten
`mcp__atlassian-mayflower__getConfluencePage` (per `id`) holen und zusammenfassen:
- **Highlights:** 2–4 Sätze (ausführlich) – worum geht es / was ist der aktuelle Stand.
- **Gruppen-Einträge (bis zur Obergrenze):** 2–3 Sätze (kürzer) – worum geht es / was ist neu.

Kein echter Versions-Diff verfügbar → Stand beschreiben, nicht „diese Zeilen kamen dazu".
Schlägt der Fetch fehl (Rechte/gelöscht) → Eintrag ohne Summary, Notiz „Inhalt nicht abrufbar".
Einträge **jenseits** der Gruppen-Obergrenze werden NICHT geholt (nur Titel + Link + Datum).
Ergänze je Seite die „Warum für dich"-Begründung aus den Treffer-Signalen
(Mention → „Du wirst erwähnt", ownEdit → „Von dir mitbearbeitet",
keyword → „Thema ‚<kw>'" – bei mehreren getroffenen Keywords diese aufzählen). In den Gruppen
genügt das jeweilige Gruppensignal (bei „Deine Themen" das/die Keyword(s) angeben).

### 8. Rendern (Markdown im Chat)

```markdown
# Confluence-Digest · <Fenster-Label>
<X relevante Seiten>

## ⭐ Highlights
### <Titel> · <space.name> · zuletzt von <author.displayName>, <lastModified>
<2–4 Sätze Zusammenfassung>
**Warum für dich:** <Begründung>
🔗 <webUrl>

## Nach Signal gruppiert
### 🔔 Dich betreffend (Mentions)
- **<Titel>** · <space.name> · <lastModified>
  <2–3 Sätze Summary>
  🔗 <webUrl>
### ✏️ Von dir mitbearbeitet
- ...
### 🏷️ Deine Themen
- **<Titel>** · <space.name> · <lastModified> *(‚<kw>')*
  <2–3 Sätze Summary>
  🔗 <webUrl>
```

(Einträge jenseits der Gruppen-Obergrenze ohne Summary: `- <Titel> · <space.name> · <lastModified> 🔗 <webUrl>`.)

Regeln:
- Jede Seite NUR EINMAL: als Highlight ODER in genau einer Gruppe.
  In der Gruppenliste zählt nur das **höchstpriore** Signal einer Seite.
  Gruppen-Prioritätsreihenfolge: **Mentions > ownEdits > Themen (keyword)**.
  Eine Seite landet also nur dann in „🏷️ Deine Themen", wenn ihr höchstpriores Signal ein
  Keyword ist (sie also weder Mention noch ownEdit ist). Eine Mention+Keyword-Seite erscheint
  ausschließlich unter Mentions. Die vollständige Mehrfach-Begründung erscheint nur bei den Highlights.
- Pro Gruppe bekommen die ersten `limits.groupSummaries` (Default 8) Einträge eine 2–3-Satz-Summary
  (Reihenfolge nach Ranking/Aktualität); weitere Einträge derselben Gruppe nur Titel + Link + Datum.
  Sind in einer Gruppe Einträge ohne Summary übrig, hänge ans Gruppenende
  „(+N weitere ohne Zusammenfassung)" an.
- Leere Gruppen weglassen. Gar nichts → „🟢 Nichts Neues im Zeitraum (<Label>)."
- Volumen-Notiz anhängen, wo die Gesamtzahl `> limit`: „(+N weitere – Fenster ggf. zu groß)".
  Das gilt auch je Keyword-Query (z.B. „(+N weitere zu ‚<kw>' – Fenster ggf. zu groß)").
- **Setup-Hinweis (Stufe 1.5):** Sind **beide** Listen `signals.keywords` UND
  `signals.titleKeywords` leer UND `onboarding.hintShown`
  false (ein **fehlender** `onboarding`-Schlüssel gilt als `false`), hänge **einmalig** am Ende
  des Digests eine dezente Zeile an:
  „💡 Tipp: `/confluence-digest setup`, um eigene Themen hinzuzufügen."
  Setze danach `onboarding.hintShown: true` in `config.local.yaml` (Schlüssel anlegen, falls er
  fehlt), damit der Hinweis bei künftigen Läufen nicht erneut erscheint. Sind in einer der beiden
  Listen Keywords gesetzt, entfällt der Hinweis. Wurde in DIESEM Lauf das Interview durchlaufen oder wurden
  gerade Keywords gesetzt, gilt `onboarding.hintShown` bereits als true — der Hinweis entfällt.

## Fehlerbehandlung
- MCP `atlassian-mayflower` nicht verbunden / Auth-Fehler → klare Meldung:
  „Bitte `/mcp` öffnen und `atlassian-mayflower` authentifizieren." → Abbruch.
- `atlassianUserInfo` ohne `account_id` → „mich betreffend"-Signale überspringen, Hinweis ausgeben.
- Einzelne CQL-Abfrage schlägt fehl → überspringen, Hinweis, restliche Signale weiterverarbeiten.

## --dry-run
Schritte 1–6 normal, aber statt Schritt 7/8: gib pro Signal das CQL und die Gesamtzahl aus,
plus die gerankte Trefferliste (Titel + Signale), ohne Seiten zu holen oder zusammenzufassen.
Bei nicht-leerer `signals.keywords`: je Eintrag das `text ~`-CQL plus Gesamtzahl ausgeben;
bei nicht-leerer `signals.titleKeywords`: je Eintrag das `title ~`-CQL plus Gesamtzahl.

## setup / Onboarding-Interview
Wird ausgelöst durch `/confluence-digest setup` **oder** beim Erststart einer neuen Nutzer:in
(§1, fehlende `config.local.yaml`). Ziel: die Keyword-Listen `signals.keywords` (Volltext) und
`signals.titleKeywords` (nur Titel) in `config.local.yaml` pflegen.
Stufe 1.5 deckt im Interview **nur Keywords** ab (+ Identitätsbestätigung); der Personen-Teil
ist Stufe 2.

**1. Identität bestätigen.** Rufe `mcp__atlassian-mayflower__atlassianUserInfo` auf, lies
`account_id` + `name`. Schreibe `accountId: <account_id>` in `config.local.yaml` (ersetzt `auto`)
und bestätige: „Eingerichtet als <name>." Schlägt der Aufruf fehl → siehe Fehlerbehandlung.

**2. Vorschläge sammeln.** Führe zwei Suchen via `searchConfluenceUsingCql` aus – Fenster fest
`now("-90d")`, `limit (= 50)`, jeweils **mit** `expand: "content.metadata.labels"`:
- `mention = currentUser() AND type = page AND lastmodified >= now("-90d") ORDER BY lastmodified DESC`
- `contributor = currentUser() AND type = page AND lastmodified >= now("-90d") ORDER BY lastmodified DESC`

Aus den Treffern beider Abfragen Kandidaten ableiten:
- **Labels:** alle `content.metadata.labels.results[].name` einsammeln (Labels sind oft
  sparse/leer – das ist ok, sie sind nur ein Teil der Quelle).
- **Titel-Terme:** Tokenisiere die Titel an Whitespace/Satzzeichen. Verwirf: Tokens < 3 Zeichen,
  reine Zahlen, Jahreszahlen (19xx/20xx), gängige de/en-Stoppwörter sowie generische
  Confluence-Begriffe (z.B. Meeting, Notes, Protokoll, Doku, Seite, Page, Übersicht). Zähle die
  verbleibenden Tokens case-insensitiv. Bilde KEINE Mehrwort-Phrasen (nur Einzeltoken).
- **Kandidatenliste** = zuerst Labels (nach Häufigkeit), dann Titel-Terme (nach Häufigkeit); bei
  Gleichstand alphabetisch. Nimm nach case-insensitivem Dedup die ersten ~10.

**3. Dialog.** Zeige die Kandidaten nummeriert. Die Nutzer:in kann frei auswählen, einzelne
streichen und beliebige eigene Keywords ergänzen. Es gibt kein Minimum/Maximum; eine leere
Auswahl ist erlaubt. Weise kurz auf die zwei Match-Arten hin und lass pro Keyword wählen:
**Volltext** (Standard, findet das Wort auch im Body) → `signals.keywords`; **nur Titel**
(schmal, gut für breite Orts-/Adressbegriffe wie den eigenen Standort) → `signals.titleKeywords`.
Im Zweifel Volltext.

**4. Ergebnis speichern.** Schreibe die gewählten Keywords je nach Match-Art nach
`signals.keywords` (Volltext) bzw. `signals.titleKeywords` (nur Titel) in `config.local.yaml`
und setze `onboarding.hintShown: true` (Schlüssel `onboarding` anlegen, falls er fehlt). Damit
erscheint der Setup-Hinweis aus §8 künftig nicht mehr.

**5. Direkt loslaufen.** Biete an, sofort einen Digest zu laufen
(normaler Ablauf ab §2 mit Standardfenster). Bei Zustimmung ausführen, sonst beenden.
Der unmittelbar danach angebotene Digest-Lauf verwendet die soeben geschriebenen Config-Werte
(Keywords in `signals.keywords`/`signals.titleKeywords` + `onboarding.hintShown: true`).
