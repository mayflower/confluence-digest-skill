# Confluence-Digest Stufe 1 – Implementierungsplan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Ein prompt-basierter Slash-Command `/confluence-digest`, der org-weit (mayflowergmbh) die für mich relevanten, kürzlich geänderten Confluence-Seiten findet (Signal „mich betreffend": Mentions + eigene Bearbeitungen) und einen priorisierten Digest mit KI-Zusammenfassungen im Chat ausgibt.

**Architecture:** Reine Claude-Code-Skill (`SKILL.md`, kein Code/keine Runtime-Abhängigkeit), die die `atlassian-mayflower`-MCP-Tools orchestriert. Deterministische Logik (Zeitfenster) wird auf native CQL-`now(...)`-Ausdrücke abgebildet – keine eigene Datumsarithmetik. Persönliche Daten liegen in einer nutzer-lokalen, gitignored Config; die Account-ID wird beim Erststart automatisch ermittelt. Quelle der Wahrheit ist das Repo `~/Code/confluence-digest`; installiert wird per Symlink ins Skills-Verzeichnis.

**Tech Stack:** Markdown-Skill (`SKILL.md` mit YAML-Frontmatter), Atlassian Remote MCP (`atlassian-mayflower`), CQL. Config als YAML.

---

## Verifizierte Fakten (Spike vom 2026-06-11)

Gegen `mayflowergmbh.atlassian.net` (cloudId `<cloud-id>`) bestätigt:

- **Account-ID** via `atlassianUserInfo` → Feld `account_id` (hier: `<account-id>`, name „Beispiel").
- **`contributor = currentUser()`** funktioniert → eigene Bearbeitungen.
- **`mention = currentUser()`** funktioniert → Erwähnungen. `currentUser()` macht eine hartcodierte Account-ID im CQL unnötig.
- **Relative Fenster nativ:** `lastmodified >= now("-24h")` / `now("-72h")` / `now("-7d")`. **Keine eigene Datumsrechnung nötig.**
- **`ORDER BY lastmodified DESC`** funktioniert.
- **Dedup nötig:** dieselbe Seite kann in beiden Abfragen auftauchen (im Spike: „2026 Reisekosten WÜRZBURG" in Mentions *und* Contributor).
- **Response je Treffer** (`searchConfluenceUsingCql` → `content.nodes[]`): `id`, `type`, `status`, `title`, `lastModified` (bereits relativer Text, z.B. „vor etwa 2 Stunden"), `summary` (Content-Snippet mit HTML-Entities), `space.{key,name}`, `author.displayName` (Autor/letzter Bearbeiter der Seite – **nicht** zwingend ich), `webUrl`. Plus `content.totalCount`.

**Konsequenz fürs Rendern:** `author.displayName` ist nicht „wer mich erwähnt hat", sondern der Seitenautor. Für Stufe 1 wird er als „zuletzt von" dargestellt, nicht als „erwähnt durch".

**Korrektur aus der Live-Akzeptanz (2026-06-11):** Die Spike-Antworten oben zeigten ein
vereinfachtes Format (`content.nodes[]`/`totalCount`/`space.name`/`author.displayName`/`webUrl`).
Im echten Skill-Lauf liefert `searchConfluenceUsingCql` jedoch das **rohe Confluence-Search-Format**:
`results[]`, `totalSize`, `<item>.content.id`, `<item>.resultGlobalContainer.title` (Space-Name),
`<item>.content.history.createdBy.displayName` (Autor), `<item>.friendlyLastModified` (relativ),
`<item>.url` (relativ; absolut via `_links.base + url`). Die `SKILL.md` mappt Felder daher nach
Bedeutung und toleriert beide Formen (siehe SKILL.md §3 Tabelle).

---

## Wichtige Designentscheidungen für Stufe 1

- **Nur Signal „mich betreffend":** `mentions` + `ownEdits`. Keine Keywords/Personen (Stufe 1.5/2).
- **Mini-Onboarding:** Bei fehlender Config nur Identität ermitteln + Config schreiben (kein Themen-Interview – das kommt in 1.5).
- **Ausgabe:** nur Chat (Markdown). Keine Datei/Confluence-Ausgabe (Stufe 2).
- **Zeitfenster:** Standard `now("-24h")`; ist heute Montag `now("-72h")`; Override-Argument (`24h`/`72h`/`7d`/`Nd`/`Nh`) erlaubt. Wochentag aus dem vom Harness gelieferten aktuellen Datum.
- **`maxSummaries`:** Default 8.

---

## Repo- & Installationsstruktur

```
~/Code/confluence-digest/
├── SKILL.md                 # die Skill (Quelle der Wahrheit)
├── config.example.yaml      # Vorlage (committet)
├── config.local.yaml        # pro Nutzer, GITIGNORED (zur Laufzeit erzeugt)
├── README.md
└── docs/plans/...
```

Installation (Stufe 1, lokal): Symlink `~/.claude/skills/confluence-digest` → `~/Code/confluence-digest`.
Verteilung an Kolleg:innen später: Ordner kopieren (ohne `config.local.yaml`).

---

## Task 1: Repo-Gerüst & Installation

**Files:**
- Create: `~/Code/confluence-digest/config.example.yaml`
- Create (Symlink): `~/.claude/skills/confluence-digest` → `~/Code/confluence-digest`

**Step 1: Config-Vorlage anlegen**

`config.example.yaml`:
```yaml
# confluence-digest – Vorlage. Beim ersten Lauf wird daraus config.local.yaml erzeugt.
cloudId: <cloud-id>   # mayflowergmbh
accountId: auto            # beim 1. Lauf via atlassianUserInfo gesetzt
signals:
  mentions: true
  ownEdits: true
  keywords: []             # Stufe 1.5
  people:   []             # Stufe 2
limits:
  maxSummaries: 8
```

**Step 2: Symlink anlegen**

Run: `ln -s ~/Code/confluence-digest ~/.claude/skills/confluence-digest`
Expected: `ls -la ~/.claude/skills/confluence-digest` zeigt den Symlink.

**Step 3: Commit**
```bash
git add config.example.yaml
git commit -m "feat: add config template and skill install scaffold"
```

---

## Task 2: SKILL.md-Grundgerüst (Frontmatter + Modus-Routing)

**Files:**
- Create: `~/Code/confluence-digest/SKILL.md`

**Step 1: Frontmatter + Übersicht schreiben**

```markdown
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

\`\`\`
/confluence-digest            → Standardfenster (24h; montags 72h)
/confluence-digest 7d         → Override-Fenster (z.B. nach Urlaub)
/confluence-digest --dry-run  → nur CQL + Trefferzahlen, ohne Inhalte/Zusammenfassungen
\`\`\`
```

**Step 2 (Akzeptanz):** Skill erscheint nach Reload in der Skill-Liste; `/confluence-digest` wird erkannt.

**Step 3: Commit**
```bash
git add SKILL.md
git commit -m "feat: add confluence-digest skill skeleton (frontmatter + invocation)"
```

---

## Task 3: Mini-Onboarding & Config-Laden

**Files:** Modify `SKILL.md` (Abschnitt „## Ablauf → 1. Config")

**Step 1: Abschnitt schreiben**

```markdown
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
```

**Step 2 (Akzeptanz):** Bei fehlender Config wird sie mit echter `accountId` erzeugt; zweiter Lauf erzeugt sie nicht erneut.

**Step 3: Commit** `git commit -am "feat: config loading + mini-onboarding"`

---

## Task 4: Zeitfenster bestimmen

**Files:** Modify `SKILL.md`

**Step 1: Abschnitt schreiben**

```markdown
### 2. Zeitfenster bestimmen
- Lies das aktuelle Datum aus dem Harness-Kontext (Feld „Today's date").
- Optionales Argument hat Vorrang: `24h`/`48h`/`72h` → `now("-Xh")`; `Nd` (z.B. `7d`) → `now("-Nd")`.
- Kein Argument: ist heute **Montag** → `now("-72h")` (Fr–Mo), sonst → `now("-24h")`.
- Merke dir das gewählte Fenster als Label für die Ausgabe (z.B. „letzte 24h", „Fr–Mo", „letzte 7 Tage").
- Bei `--dry-run`: Argument-Parsing identisch, nur spätere Schritte unterscheiden sich.
```

**Step 2 (Akzeptanz):** `/confluence-digest 7d` ergibt `now("-7d")`; ohne Argument an einem Montag `now("-72h")`, sonst `now("-24h")`.

**Step 3: Commit** `git commit -am "feat: time-window mapping to CQL now()"`

---

## Task 5: CQL-Abfragen, Merge & Dedup

**Files:** Modify `SKILL.md`

**Step 1: Abschnitt schreiben**

```markdown
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
```

**Step 2 (Akzeptanz, `--dry-run`):** Beide CQL-Strings werden ausgegeben, je Signal eine Trefferzahl; eine in beiden Listen vorkommende Seite erscheint nur einmal mit zwei Signalen.

**Step 3: Commit** `git commit -am "feat: per-signal CQL queries + merge/dedup"`

---

## Task 6: Ranking & Top-N-Auswahl

**Files:** Modify `SKILL.md`

**Step 1: Abschnitt schreiben**

```markdown
### 5. Ranken
Score je Seite: Mention = 2, ownEdit = 1, summiert über die Treffer-Signale
(Mention+ownEdit = 3 > nur Mention = 2 > nur ownEdit = 1). Sekundärsortierung: Aktualität
(Reihenfolge aus `ORDER BY lastmodified DESC`).

### 6. Top-N für Zusammenfassung
- Die obersten `min(maxSummaries, Anzahl)` Seiten werden zusammengefasst (Highlights-Kandidaten).
- Davon werden die Top 3–5 als „Highlights" ausführlich dargestellt; der Rest kommt in die Gruppenliste.
- Seiten jenseits von `maxSummaries` erscheinen nur als Titel+Link in der Gruppenliste (ohne Fetch).
```

**Step 2 (Akzeptanz):** Eine Mention+ownEdit-Seite rankt über einer reinen ownEdit-Seite; nie mehr als `maxSummaries` Seiten werden für Zusammenfassung geholt.

**Step 3: Commit** `git commit -am "feat: ranking + top-N selection"`

---

## Task 7: Inhalte holen & zusammenfassen

**Files:** Modify `SKILL.md`

**Step 1: Abschnitt schreiben**

```markdown
### 7. Zusammenfassen
Für jede Highlights-Kandidaten-Seite `mcp__atlassian-mayflower__getConfluencePage` (per `id`)
holen und 2–4 Sätze zusammenfassen: worum geht es / was ist der aktuelle Stand. Kein echter
Versions-Diff verfügbar → Stand beschreiben, nicht „diese Zeilen kamen dazu".
Schlägt der Fetch fehl (Rechte/gelöscht) → Seite ohne Zusammenfassung, mit Notiz „Inhalt nicht abrufbar".
Ergänze je Seite die „Warum für dich"-Begründung aus den Treffer-Signalen
(Mention → „Du wirst erwähnt", ownEdit → „Von dir mitbearbeitet").
```

**Step 2 (Akzeptanz):** Top-Seiten haben eine 2–4-Satz-Zusammenfassung + korrekte „Warum für dich"-Zeile.

**Step 3: Commit** `git commit -am "feat: page fetch + AI summary + reason line"`

---

## Task 8: Rendering

**Files:** Modify `SKILL.md`

**Step 1: Abschnitt schreiben** (exaktes Ausgabeformat)

```markdown
### 8. Rendern (Markdown im Chat)

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

Regeln:
- Jede Seite NUR EINMAL: als Highlight ODER in genau einer Gruppe (höchste Priorität: Mentions vor ownEdits).
- Leere Gruppen weglassen. Gar nichts → „🟢 Nichts Neues im Zeitraum (<Label>)."
- Volumen-Notiz anhängen, wo `totalCount > limit`: „(+N weitere – Fenster ggf. zu groß)".
```

**Step 2 (Akzeptanz):** Ausgabe entspricht dem Format; keine Seite doppelt; leere Gruppen fehlen.

**Step 3: Commit** `git commit -am "feat: digest rendering"`

---

## Task 9: Fehlerbehandlung & --dry-run

**Files:** Modify `SKILL.md`

**Step 1: Abschnitt schreiben**

```markdown
## Fehlerbehandlung
- MCP `atlassian-mayflower` nicht verbunden / Auth-Fehler → klare Meldung:
  „Bitte `/mcp` öffnen und `atlassian-mayflower` authentifizieren." → Abbruch.
- `atlassianUserInfo` ohne `account_id` → „mich betreffend"-Signale überspringen, Hinweis ausgeben.
- Einzelne CQL-Abfrage schlägt fehl → überspringen, Hinweis, restliche Signale weiterverarbeiten.

## --dry-run
Schritte 1–6 normal, aber statt Schritt 7/8: gib pro Signal das CQL und `totalCount` aus,
plus die gerankte Trefferliste (Titel + Signale), ohne Seiten zu holen oder zusammenzufassen.
```

**Step 2 (Akzeptanz):** Bei getrenntem MCP saubere Anleitung statt Stacktrace; `--dry-run` macht keine `getConfluencePage`-Calls.

**Step 3: Commit** `git commit -am "feat: error handling + dry-run mode"`

---

## Task 10: Live-Akzeptanz & Distributionshinweis

**Files:** Modify `README.md`

**Step 1: End-to-End gegen Live-Instanz** (manuell, durch die Nutzer:in bestätigt)
- `/confluence-digest --dry-run` → beide CQL korrekt, plausible Trefferzahlen.
- `/confluence-digest 7d` → Digest mit Highlights + Gruppen, korrekte Links, keine Dopplungen.
- `/confluence-digest` ohne Argument → korrektes Standardfenster (Wochentagslogik).
- Erststart-Verhalten: temporär `config.local.yaml` umbenennen, Lauf → Config wird mit echter accountId neu erzeugt.

**Step 2: README ergänzen** um „Installation (Symlink)", „Voraussetzung: `atlassian-mayflower`-MCP authentifiziert", „An Kolleg:innen verteilen: Ordner ohne `config.local.yaml` kopieren".

**Step 3: Commit** `git commit -am "docs: install + distribution notes; mark Stufe 1 done"`

---

## Out of Scope (spätere Stufen)
- Keywords-Signal + Hybrid-Onboarding-Interview + `setup`-Befehl (Stufe 1.5)
- Personen/Teams-Signal, geplanter Lauf, Datei-/Confluence-Ausgabe (Stufe 2)
- macOS-Desktop-App (Stufe 3)
```
