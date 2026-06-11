---
name: confluence-digest
description: Use when the user runs /confluence-digest (optionally with a time window `Nh`/`Nd`, z.B. 24h, 72h, 7d) to get a prioritized overview of recently changed Confluence pages (mayflowergmbh) relevant to them — pages they are mentioned on or have contributed to — with AI summaries. Triggers: /confluence-digest, Confluence-Überblick, was ist neu in Confluence, Confluence-Digest.
---

# Confluence-Digest (Stufe 1)

Priorisierter Überblick über kürzlich geänderte, für die Nutzer:in relevante Confluence-Seiten
auf `mayflowergmbh`. Stufe 1 = Signal „mich betreffend" (Mentions + eigene Bearbeitungen).

**Datenquelle:** MCP-Server `atlassian-mayflower` (Tools `mcp__atlassian-mayflower__*`).
**Konfiguration:** `config.local.yaml` im Skill-Verzeichnis (nutzer-lokal, gitignored).
**Design:** `docs/plans/2026-06-11-confluence-digest-design.md`

## Aufruf

```
/confluence-digest            → Standardfenster (24h; montags 72h)
/confluence-digest <Fenster>  → Override-Fenster `Nh`/`Nd` (z.B. 24h, 72h, 7d – etwa nach Urlaub)
/confluence-digest --dry-run  → nur CQL + Trefferzahlen, ohne Inhalte/Zusammenfassungen
```

## Ablauf

### 1. Config laden / Mini-Onboarding
- Das Skill-Verzeichnis wird über einen Symlink erreicht
  (`~/.claude/skills/confluence-digest` → echtes Repo). Lies und schreibe `config.local.yaml`
  **neben `SKILL.md` im aufgelösten echten Verzeichnis** (z.B. via `realpath`), damit die Datei
  im gitignored Repo landet und nicht im Symlink-Pfad.
- Prüfe, ob `config.local.yaml` dort existiert.
- **Fehlt sie:** Begrüße kurz, rufe `mcp__atlassian-mayflower__atlassianUserInfo` auf,
  lies `account_id` + `name`. Erzeuge `config.local.yaml` aus `config.example.yaml`
  mit `accountId: <account_id>`. Bestätige der Nutzer:in: „Eingerichtet als <name>."
  (KEIN Themen-Interview – das ist Stufe 1.5.)
- **Existiert sie:** lade Werte. Steht `accountId: auto`, einmalig wie oben auflösen und
  zurückschreiben.
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

Aus jedem `content.nodes[]`-Eintrag liest du: `id`, `title`, `space.name`, `author.displayName`,
`lastModified`, `webUrl`; zusätzlich je Query `content.totalCount`.

### 4. Mergen & dedupen
- Sammle alle `content.nodes[]`. Schlüssel = `id`.
- Pro Seite merke die Menge der Treffer-Signale (eine Seite kann mention UND ownEdit sein).
- Merke je Query `content.totalCount`; bei `totalCount > limit` für die Volumen-Notiz vormerken
  (`N = totalCount - limit`, also `totalCount - 50`).

### 5. Ranken
Score je Seite: Mention = 2, ownEdit = 1, summiert über die Treffer-Signale
(Mention+ownEdit = 3 > nur Mention = 2 > nur ownEdit = 1). Sekundärsortierung: Aktualität
(Reihenfolge aus `ORDER BY lastmodified DESC`).

### 6. Highlights bestimmen
- Bestimme die Zahl der Highlights `H`: die Top 5 (oder alle, falls weniger als 5 Treffer),
  zusätzlich gedeckelt durch `maxSummaries` (Default 8). Formal:
  `H = min(5, Anzahl gerankter Seiten, maxSummaries)`.
- `maxSummaries` ist eine **Sicherheits-Obergrenze** dafür, wie viele Seiten pro Lauf überhaupt
  geholt/zusammengefasst werden dürfen.
- **Nur diese H Seiten** werden via `getConfluencePage` geholt und in §7 zusammengefasst – keine
  überflüssigen Fetches, keine verwaisten Zusammenfassungen.
- Alle übrigen gerankten Seiten (Rang > H) kommen in die Gruppenliste (§8) als **Titel + Link +
  relatives Datum**, OHNE Zusammenfassung und OHNE Fetch.

### 7. Zusammenfassen
Zusammenfassungen entstehen ausschließlich für die H Highlight-Seiten aus §6.
Für jede dieser H Seiten `mcp__atlassian-mayflower__getConfluencePage` (per `id`)
holen und 2–4 Sätze zusammenfassen: worum geht es / was ist der aktuelle Stand. Kein echter
Versions-Diff verfügbar → Stand beschreiben, nicht „diese Zeilen kamen dazu".
Schlägt der Fetch fehl (Rechte/gelöscht) → Seite ohne Zusammenfassung, mit Notiz „Inhalt nicht abrufbar".
Ergänze je Seite die „Warum für dich"-Begründung aus den Treffer-Signalen
(Mention → „Du wirst erwähnt", ownEdit → „Von dir mitbearbeitet").

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
- <Titel> · <space.name> · <lastModified> 🔗 <webUrl>
### ✏️ Von dir mitbearbeitet
- ...
```

Regeln:
- Jede Seite NUR EINMAL: als Highlight ODER in genau einer Gruppe (höchste Priorität: Mentions vor ownEdits).
  In der Gruppenliste zählt nur das höchstpriore Signal einer Seite (Mentions vor ownEdits);
  die vollständige Mehrfach-Begründung erscheint nur bei den Highlights.
- Die Gruppenliste enthält **nie** Zusammenfassungen – nur Titel + Link + relatives Datum.
- Leere Gruppen weglassen. Gar nichts → „🟢 Nichts Neues im Zeitraum (<Label>)."
- Volumen-Notiz anhängen, wo `totalCount > limit`: „(+N weitere – Fenster ggf. zu groß)".

## Fehlerbehandlung
- MCP `atlassian-mayflower` nicht verbunden / Auth-Fehler → klare Meldung:
  „Bitte `/mcp` öffnen und `atlassian-mayflower` authentifizieren." → Abbruch.
- `atlassianUserInfo` ohne `account_id` → „mich betreffend"-Signale überspringen, Hinweis ausgeben.
- Einzelne CQL-Abfrage schlägt fehl → überspringen, Hinweis, restliche Signale weiterverarbeiten.

## --dry-run
Schritte 1–6 normal, aber statt Schritt 7/8: gib pro Signal das CQL und `totalCount` aus,
plus die gerankte Trefferliste (Titel + Signale), ohne Seiten zu holen oder zusammenzufassen.
