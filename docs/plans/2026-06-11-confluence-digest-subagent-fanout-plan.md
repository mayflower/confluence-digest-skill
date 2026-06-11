# Confluence-Digest – Subagent-Fan-out für große Fenster (> 72h)

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Bei Zeitfenstern **> 72h** (also länger als übers Wochenende) führt die Skill die
CQL-Such-Queries **und** die Seiten-Zusammenfassungen über **Subagenten** aus (je ein Subagent pro
Query bzw. pro zu summender Seite, mit kompakter Rückgabe). So bleibt der Hauptkontext schlank
(die großen Rohantworten werden im Subagent-Kontext verarbeitet) und die Läufe sind nebenläufig.
Bei **≤ 72h** (Standard 24h und Montags-72h) bleibt alles **inline** wie bisher.

**Architecture:** Reine `SKILL.md`-Erweiterung (Instruktionen) – kein Code. Der ausführende Agent
nutzt das Agent/Task-Tool zum Fan-out. Merge/Dedupe/Ranking bleiben zentral im Haupt-Agenten.

**Tech Stack:** Markdown-Skill, Agent/Task-Subagenten, Atlassian-MCP (über `ToolSearch` im Subagent
erreichbar – live bestätigt 2026-06-11), CQL.

**Out of Scope:** geplanter Lauf/Cloud-Agent (Stufe 2b, zurückgestellt), Label-Signal (Backlog),
„zuletzt von"-Backlog. Keine Änderung an Signalen/Ranking/Config.

---

## Verifizierte Fakten (Spike 2026-06-11)

- Ein **general-purpose-Subagent** kann den Atlassian-MCP nutzen (`ToolSearch` →
  `mcp__atlassian-mayflower__searchConfluenceUsingCql`) und eine kompakte Trefferliste
  zurückgeben. Im Test verarbeitete er die ~110k-Zeichen-Rohantwort intern (~54k Tokens) und gab
  nur 10 Zeilen zurück → Kontext-Isolation bestätigt.

## Designentscheidungen

- **Modus-Schwelle:** Fenstergröße in Stunden bestimmen (24h→24; Montags-Default→72; Override `Nh`→N;
  `Nd`→N·24). **> 72h → Fan-out-Modus**, sonst **Inline-Modus**. `--dry-run` läuft **immer inline**
  (braucht nur CQL + `totalSize`, keine Inhalte).
- **Phase 1 (Such-Fan-out):** je aktiver Signal-Query (mentions, ownEdits, je `keywords`-Eintrag,
  je `titleKeywords`-Eintrag, je `people`-Eintrag) **ein** Subagent. Rückgabe-Schema **fix**:
  - Kopf: `signal` (mentions|ownEdits|keyword:<kw>|titleKeyword:<kw>|person:<name>), `totalSize`
  - je Treffer eine Zeile: `id | title | space (resultGlobalContainer.title) | author
    (content.history.createdBy.displayName) | friendlyLastModified | absolute URL (_links.base+url)
    | type (page|blogpost)`
  - **kein** rohes JSON zurückgeben.
- **Merge/Dedupe/Ranking:** unverändert im Haupt-Agenten, auf Basis der kompakten Rückgaben
  (Schlüssel = `id`; Signale aggregieren; Scores wie gehabt Mention=4/ownEdit=2/keyword=1/person=1).
- **Phase 2 (Summary-Fan-out):** je zu summender Seite (H Highlights + je Gruppe bis
  `limits.groupSummaries`) **ein** Subagent: holt `getConfluencePage` (per `id`, markdown) und gibt
  **nur** die Zusammenfassung zurück (Highlights 2–4 Sätze, Gruppen-Einträge 2–3 Sätze) + die
  „Inhalt nicht abrufbar"-Notiz bei Fehlern. Kein Roh-Body zurück.
- **Fehler:** Subagent-Query fehlgeschlagen/leer → meldet das kompakt; Haupt-Agent verarbeitet die
  übrigen weiter (analog bestehender „einzelne CQL-Abfrage fehlgeschlagen"-Regel).
- **Inline-Modus (≤ 72h):** unverändert – Queries als parallele Inline-Calls, Summaries zentral.

---

## Task 1: Modus-Bestimmung + Fan-out-Schema in §2/§3 (SKILL.md)

**Step 1:** In §2 (Zeitfenster) die **Fenstergröße in Stunden** + **Modus** ergänzen
(> 72h → Fan-out, sonst Inline; `--dry-run` immer inline).
**Step 2:** In §3 einen Block „**Ausführungsmodus**" einfügen:
- **Inline (≤ 72h):** Queries als parallele `searchConfluenceUsingCql`-Calls direkt (wie bisher).
- **Fan-out (> 72h):** je Signal-Query ein Subagent (Agent/Task-Tool, general-purpose) mit dem
  **fixen Rückgabe-Schema** (siehe Designentscheidung) – Subagent lädt via `ToolSearch` das
  MCP-Tool, führt die CQL aus, gibt nur die kompakte Liste + `totalSize` zurück.
- Die konkreten CQL-Strings je Signal bleiben identisch (nur die Ausführung unterscheidet sich).
**Step 3 (Akzeptanz):** §3 beschreibt beide Modi eindeutig; Schema ist vollständig.
**Step 4: Commit** `feat: add execution-mode selection + search fan-out (>72h)`

---

## Task 2: Summary-Fan-out in §6/§7 (SKILL.md)

**Step 1:** §7 ergänzen: Im **Fan-out-Modus** wird jede zu summende Seite (H Highlights + je Gruppe
bis `limits.groupSummaries`) von **einem Subagenten** geholt+zusammengefasst (Rückgabe nur die
2–4- bzw. 2–3-Satz-Summary); im **Inline-Modus** wie bisher zentral. §6 unverändert (Auswahl,
welche Seiten gefetcht werden, bleibt gleich – nur das *Wie* wechselt).
**Step 2 (Akzeptanz):** §7 nennt beide Modi; Cap-/Highlight-Logik unverändert konsistent.
**Step 3: Commit** `feat: add summary fan-out (>72h)`

---

## Task 3: --dry-run, Doku, Verteilung

**Step 1:** `--dry-run`-Abschnitt: explizit „immer inline" (nur CQL + `totalSize`, kein Fan-out).
**Step 2:** README: kurzen Abschnitt „Ausführungsmodus" – ab > 72h Subagenten-Fan-out
(Begründung: schlanker Kontext + parallel); Voraussetzung: Agent/Task-Fähigkeit im Claude Code.
**Step 3:** Design-Doc-Backlog-Eintrag entfernen/auf „umgesetzt" setzen, falls vorhanden.
**Step 4: Commit** `docs: dry-run inline note + execution-mode in README`

---

## Task 4: Live-Akzeptanz (Controller + Nutzer:in)

- `/confluence-digest 7d` → läuft im Fan-out-Modus; Ergebnis konsistent mit einem Inline-7d-Lauf
  (Stichprobe), Hauptkontext bleibt schlank.
- `/confluence-digest` (24h) → bleibt inline.
- `/confluence-digest --dry-run 7d` → inline, nur CQL + Trefferzahlen.
(Optionaler/teurer Schritt – kann auch von der Nutzer:in im Alltag verifiziert werden.)
