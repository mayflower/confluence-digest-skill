# Confluence-Digest Stufe 2a – Implementierungsplan (Personen-Signal)

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Ein viertes Signal **„verfolgte Personen"**: Seiten, die bestimmte Kolleg:innen bearbeitet
haben, erscheinen im Digest als neue Gruppe „👥 Verfolgte Personen" (mit Summary). Das `setup`-Interview
bekommt einen Personen-Schritt (Name → accountId auflösen).

**Architecture:** Reine prompt-basierte `SKILL.md`-Erweiterung + nutzer-lokale Config. Personen werden
als Name+accountId gespeichert; CQL nutzt `contributor = "<id>"` pro Person (Attribution „von wem").
Live-Akzeptanz statt Unit-Tests.

**Tech Stack:** Markdown-Skill, Atlassian Remote MCP (`atlassian-mayflower`), CQL
(`contributor`), `lookupJiraAccountId`.

**Out of Scope:** geplanter Lauf (Cloud-Agent), Datei-/Confluence-Ausgabe (Stufe 2b),
„zuletzt von"-Backlog, macOS-App.
**Teams/Gruppen:** laut Nutzer werden bei Mayflower **keine Teams in Confluence gepflegt** →
Team-Verfolgung vorerst gestrichen (nicht nur vertagt). Nur Einzelpersonen.

---

## Verifizierte Fakten (Spike 2026-06-11)

- **`lookupJiraAccountId`** (cloudId, searchString) löst einen Namen auf →
  `data.users.users[]` mit `accountId`, `displayName`, E-Mail (im Test „Max Mustermann" → genau
  1 Treffer `<account-id>`). Liefert auch `data.groups` (hier leer)
  – wäre für Teams nutzbar, aber Teams werden laut Nutzer in Confluence nicht gepflegt → nicht verfolgt.
- **`contributor = "<id>"`** (Einzelperson) und **`contributor in ("<id1>", ...)`** funktionieren in
  CQL; im Test 8 von Max in 7d bearbeitete Seiten. Antwortformat = rohes Search-Format wie gehabt.
- Es gibt **keinen** Hinweis darauf, *welche* Person eine Seite via `contributor in (...)` ausgelöst
  hat → für Attribution „von wem" pro Person eine eigene `contributor = "<id>"`-Query (wie bei Keywords).

---

## Designentscheidungen Stufe 2a

- **Config `signals.people`:** Liste von Einträgen `{name, id}`, z.B.
  `people: [{name: "Max Mustermann", id: "<account-id>"}]`. Nicht-leer = Signal aktiv.
- **Abfrage:** pro Person eine `contributor = "<id>" AND type = page AND lastmodified >= now("<FENSTER>")
  ORDER BY lastmodified DESC`, `limit (= 50)`. (Pro Person → Attribution „von <name>".)
- **Runtime-Auflösung:** Hat ein Eintrag nur `name` (manuell ergänzt, `id` fehlt/leer), löse einmalig
  via `lookupJiraAccountId` auf, nimm den ersten Treffer, nutze dessen `accountId` und schreibe ihn in
  die Config zurück. Keine Auflösung möglich → Eintrag überspringen, Hinweis.
- **Signal/Score:** neues Signal `person`, Score **1** (gleiche Stufe wie keyword). Score-Summe wie
  gehabt (Mention=4, ownEdit=2, keyword=1, person=1).
- **Gruppen-Priorität (für „nur einmal"-Einsortierung):**
  **Mentions > ownEdits > Themen (keyword) > Verfolgte Personen (person)**.
- **Neue Render-Gruppe „👥 Verfolgte Personen"** nach „Deine Themen". Eintrag zeigt zusätzlich
  *(von <name>)*. Summaries je `limits.groupSummaries` wie andere Gruppen.
- **Interview:** Personen-Schritt nach dem Keyword-Schritt (Name eingeben → auflösen → bestätigen →
  speichern). Vorschläge optional aus häufigen Mit-Autor:innen (siehe Task 4, „nice to have").

---

## Task 1: Spike ✅ ERLEDIGT (2026-06-11)
`lookupJiraAccountId` + `contributor =`/`in` live bestätigt (siehe „Verifizierte Fakten"). Keine Anpassung nötig.

---

## Task 2: Personen-Signal in der Suche (§3/§4)

**Files:** Modify `SKILL.md`

**Step 1:** §3 erweitern – ist `signals.people` nicht leer, führe **pro Person** (mit aufgelöstem
`id`) eine Abfrage aus:
`contributor = "<id>" AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`,
`limit (= 50)`. Vorab Runtime-Auflösung fehlender IDs (siehe Designentscheidung).
**Step 2:** §4 – Treffer mit Signal `person` vermerken **plus, welche Person** (Name). Mehrere
verfolgte Personen können dieselbe Seite bearbeitet haben → alle merken. Gesamtzahl je Person für
Volumen-Notiz.
**Step 3 (Akzeptanz, --dry-run):** je Person CQL + Trefferzahl.
**Step 4: Commit** `feat: add followed-people (contributor) search signal`

---

## Task 3: Ranking + Gruppe „👥 Verfolgte Personen" (§5/§6/§7/§8)

**Files:** Modify `SKILL.md`

**Step 1:** §5 – Score `person = 1` ergänzen; Beispiele aktualisieren (z.B. keyword+person = 2;
ownEdit+person = 3).
**Step 2:** §6/§7 – Personen-Treffer wie die anderen Signale behandeln (Highlights möglich;
Gruppen-Summary bis `groupSummaries`). „Warum für dich" für person → „Von <name> bearbeitet".
**Step 3:** §8 – neue Gruppe `### 👥 Verfolgte Personen` **nach** „🏷️ Deine Themen". Gruppen-Priorität
aktualisieren: Mentions > ownEdits > Themen > Verfolgte Personen. Eine Seite landet nur dann hier,
wenn ihr höchstpriores Signal `person` ist. Eintrag zeigt *(von <name>)*.
**Step 4 (Akzeptanz):** Eine nur von einer verfolgten Person bearbeitete Seite landet in „Verfolgte
Personen"; eine Seite, die auch Mention ist, bleibt unter Mentions.
**Step 5: Commit** `feat: rank person signal + 'Verfolgte Personen' group`

---

## Task 4: Personen-Schritt im setup-Interview (§setup)

**Files:** Modify `SKILL.md`

**Step 1:** Setup-Abschnitt: nach dem Keyword-Schritt einen **Personen-Schritt** ergänzen:
- Frage, welche Kolleg:innen verfolgt werden sollen (frei, optional; leere Auswahl erlaubt).
- **Optionaler Vorschlag:** häufige Mit-Autor:innen aus der Aktivitäts-Analyse (Schritt 2 holt bereits
  mention/ownEdit-Seiten; aus `content.history.createdBy.displayName` der Treffer die häufigsten
  Personen ≠ man selbst ableiten und als Kandidaten anbieten).
- Pro genannter Person `lookupJiraAccountId` aufrufen; bei mehreren Treffern den/die passende(n)
  bestätigen lassen; `{name, id}` sammeln.
**Step 2:** Speichern: gewählte Personen nach `signals.people` schreiben (als `{name, id}`).
`onboarding.hintShown` bleibt/auf `true`.
**Step 3:** §1 (neue Nutzer:in) und §setup-Intro erwähnen, dass das Interview jetzt Keywords **und**
Personen abdeckt.
**Step 4 (Akzeptanz):** `/confluence-digest setup` fragt Personen ab, löst Namen auf, schreibt
`{name,id}` in die Config.
**Step 5: Commit** `feat: add people step to setup interview`

---

## Task 5: dry-run, Config-Vorlage, README

**Files:** Modify `SKILL.md`, `config.example.yaml`, `README.md`

**Step 1:** `--dry-run` um Personen-Queries ergänzen (je Person CQL + Gesamtzahl).
**Step 2:** `config.example.yaml`: `people: []` Kommentar aktualisieren (Format `{name, id}`,
nicht-leer = aktiv, per `setup` befüllt).
**Step 3:** `README`: „Verfolgte Personen" als viertes Signal dokumentieren; Ranking-Reihenfolge
ergänzen (Mentions > ownEdits > Themen > Personen); setup deckt jetzt Keywords + Personen ab.
**Step 4: Commit** `docs: document people signal; dry-run + config + README`

---

## Task 6: Live-Akzeptanz

**Files:** ggf. `config.local.yaml` (Test) – nicht committen.

**Step 1 (Controller + Nutzer:in):**
- `/confluence-digest setup` → Personen-Schritt funktioniert, Namen werden aufgelöst, `{name,id}` landet in Config.
- `/confluence-digest --dry-run` → je Person CQL + Trefferzahl.
- `/confluence-digest 7d` → Gruppe „👥 Verfolgte Personen" erscheint, *(von <name>)*, Ranking korrekt, Dedup intakt.
**Step 2:** kurzes Fazit; offene Punkte/Verfeinerungen notieren.
