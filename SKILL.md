---
name: confluence-digest
description: Use when the user runs /confluence-digest (optionally with a time window like 24h/72h/7d) to get a prioritized overview of recently changed Confluence pages (mayflowergmbh) relevant to them — pages they are mentioned on or have contributed to — with AI summaries. Triggers: /confluence-digest, Confluence-Überblick, was ist neu in Confluence, Confluence-Digest.
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
/confluence-digest 7d         → Override-Fenster (z.B. nach Urlaub)
/confluence-digest --dry-run  → nur CQL + Trefferzahlen, ohne Inhalte/Zusammenfassungen
```

## Ablauf

### 1. Config laden / Mini-Onboarding
- Prüfe, ob `config.local.yaml` im Skill-Verzeichnis existiert.
- **Fehlt sie:** Begrüße kurz, rufe `mcp__atlassian-mayflower__atlassianUserInfo` auf,
  lies `account_id` + `name`. Erzeuge `config.local.yaml` aus `config.example.yaml`
  mit `accountId: <account_id>`. Bestätige der Nutzer:in: „Eingerichtet als <name>."
  (KEIN Themen-Interview – das ist Stufe 1.5.)
- **Existiert sie:** lade Werte. Steht `accountId: auto`, einmalig wie oben auflösen und
  zurückschreiben.
- Schlägt `atlassianUserInfo` fehl → siehe Abschnitt Fehlerbehandlung.

### 2. Zeitfenster bestimmen
- Lies das aktuelle Datum aus dem Harness-Kontext (Feld „Today's date").
- Optionales Argument hat Vorrang: `24h`/`48h`/`72h` → `now("-Xh")`; `Nd` (z.B. `7d`) → `now("-Nd")`.
- Kein Argument: ist heute **Montag** → `now("-72h")` (Fr–Mo), sonst → `now("-24h")`.
- Merke dir das gewählte Fenster als Label für die Ausgabe (z.B. „letzte 24h", „Fr–Mo", „letzte 7 Tage").
- Bei `--dry-run`: Argument-Parsing identisch, nur spätere Schritte unterscheiden sich.

### 3. Relevante Seiten holen
Für jedes aktive Signal eine CQL-Abfrage via `mcp__atlassian-mayflower__searchConfluenceUsingCql`
(cloudId aus Config), `limit: 50`, jeweils mit dem Fenster aus Schritt 2:

- mentions:  `mention = currentUser() AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`
- ownEdits:  `contributor = currentUser() AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`

(Nur Signale ausführen, die in der Config `true` sind.)

### 4. Mergen & dedupen
- Sammle alle `content.nodes[]`. Schlüssel = `id`.
- Pro Seite merke die Menge der Treffer-Signale (eine Seite kann mention UND ownEdit sein).
- Merke je Query `content.totalCount`; bei `totalCount > limit` für die Volumen-Notiz vormerken.

### 5. Ranken
Score je Seite: Mention = 2, ownEdit = 1, summiert über die Treffer-Signale
(Mention+ownEdit = 3 > nur Mention = 2 > nur ownEdit = 1). Sekundärsortierung: Aktualität
(Reihenfolge aus `ORDER BY lastmodified DESC`).

### 6. Top-N für Zusammenfassung
- Die obersten `min(maxSummaries, Anzahl)` Seiten werden zusammengefasst (Highlights-Kandidaten).
- Davon werden die Top 3–5 als „Highlights" ausführlich dargestellt; der Rest kommt in die Gruppenliste.
- Seiten jenseits von `maxSummaries` erscheinen nur als Titel+Link in der Gruppenliste (ohne Fetch).

### 7. Zusammenfassen
Für jede Highlights-Kandidaten-Seite `mcp__atlassian-mayflower__getConfluencePage` (per `id`)
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
