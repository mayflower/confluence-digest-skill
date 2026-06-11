# Confluence-Digest Stufe 1.5 – Implementierungsplan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Stufe 1 um ein **Keyword-Signal** (Volltext) und ein **Erststart-Hybrid-Interview** mit `setup`-Befehl erweitern: Nutzer:innen pflegen interessante Themen (Keywords), die org-weit per `text ~` gefunden und im Digest als drittes Signal „Deine Themen" dargestellt werden. Vorschläge im Interview entstehen aus Labels + Titeln der eigenen jüngsten Aktivität.

**Architecture:** Weiterhin reine prompt-basierte `SKILL.md` (kein Code, keine Tests) + nutzer-lokale `config.local.yaml`. Erweiterung der bestehenden Skill, keine neue. Live-Akzeptanz statt Unit-Tests.

**Tech Stack:** Markdown-Skill, Atlassian Remote MCP (`atlassian-mayflower`), CQL (`text ~`, `mention`/`contributor = currentUser()`), `searchConfluenceUsingCql` mit `expand=content.metadata.labels`.

---

## Verifizierte Fakten (Spike vom 2026-06-11)

- **Labels auslesbar** über den `expand`-Parameter der Suche: `expand: "content.metadata.labels"` →
  Labels unter `content.metadata.labels.results[].name`. Labels sind oft leer/sparse (eine von 3
  Testseiten hatte „blog") → Kombination mit Titel-Termen ist berechtigt.
- **Korpusgröße** plausibel: 55 eigene Seiten in 365d (Feld `totalSize`).
- **`text ~ "<kw>"` live bestätigt** (2026-06-11): `text ~ "Symfony" … now("-90d")` → 8 Treffer,
  gleiches `results[]`-Format. Matcht den **Volltext** (auch Body), nicht nur Titel → Rauschen
  möglich (Treffer in alten Seiten, die das Wort nur im Inhalt führen) → Volumen-Notiz greift.
- Antwort-Felder & relative Datums-/URL-Behandlung: wie in Stufe-1-Plan dokumentiert (rohes
  Search-Format `results[]`/`totalSize`/`resultGlobalContainer.title`/`content.history.createdBy`/
  `friendlyLastModified`/`_links.base + url`).

---

## Designentscheidungen Stufe 1.5

- **Keyword-Match:** Volltext `text ~ "<kw>"` (eine CQL-Query pro Keyword).
- **Keyword-Vorschläge:** Labels + häufige Titel-Terme aus eigenen jüngsten mention/ownEdit-Seiten.
- **Einstieg für Bestandsnutzer:innen:** `/confluence-digest setup` startet das Interview; zusätzlich
  zeigt der Digest bei leeren Keywords **einmalig** einen dezenten Hinweis.
- **Ranking neu:** Mention=4, ownEdit=2, **keyword=1** (summiert; Mention > ownEdit > keyword,
  Mehrfach-Treffer steigen). Ersetzt die Stufe-1-Gewichte 2/1.
- **Neue Render-Gruppe:** „🏷️ Deine Themen" für Seiten, deren höchstpriores Signal ein Keyword ist.
- **Interview deckt in 1.5 nur Keywords ab** (+ Identitätsbestätigung). Personen-Teil = Stufe 2.

---

## Config-Erweiterung

`config.example.yaml` / `config.local.yaml` bekommt einen Onboarding-Marker:

```yaml
onboarding:
  hintShown: false     # wird true, sobald der Keyword-Hinweis einmal gezeigt ODER setup gelaufen ist
```

`keywords: []` existiert bereits. `signals.keywords` schaltet das Signal an/aus (Default true, greift
aber nur, wenn `keywords` nicht leer ist).

---

## Task 1: Spike – `text ~` bestätigen ✅ ERLEDIGT (2026-06-11)

Live bestätigt: `text ~ "Symfony" AND type = page AND lastmodified >= now("-90d")` → 8 Treffer,
`results[]`-Format. Volltext-Match (auch Body). Siehe „Verifizierte Fakten". Keine Anpassung nötig.

---

## Task 2: Keyword-Signal in der Suche (§3/§4)

**Files:** Modify `SKILL.md`

**Step 1:** §3 erweitern – wenn `signals.keywords` true UND `keywords` nicht leer:
pro Keyword eine Abfrage
`text ~ "<kw>" AND type = page AND lastmodified >= now("<FENSTER>") ORDER BY lastmodified DESC`,
`limit (= 50)`, `expand` hier nicht nötig.
**Step 2:** §4 Merge/Dedup – Keyword-Treffer mit Signal `keyword` (plus welches Keyword) vermerken;
Gesamtzahl je Keyword-Query wie gehabt für die Volumen-Notiz.
**Step 3 (Akzeptanz, --dry-run):** Mit einem Test-Keyword erscheinen pro Keyword CQL + Trefferzahl.
**Step 4: Commit** `feat: add keyword (text~) search signal`

---

## Task 3: Ranking & neue Render-Gruppe (§5/§8)

**Files:** Modify `SKILL.md`

**Step 1:** §5 Scores ändern: Mention=4, ownEdit=2, keyword=1 (summiert). Beispiele aktualisieren.
**Step 2:** §8 neue Gruppe „### 🏷️ Deine Themen" nach den bestehenden Gruppen; enthält Seiten, deren
höchstpriores Signal ein Keyword ist (Format wie andere Gruppen: Titel · Space · Datum 🔗).
„Warum für dich" bei Keyword-Highlights: „Thema ‚<kw>'".
Prioritätsreihenfolge der Gruppen: Mentions > ownEdits > Themen.
**Step 3 (Akzeptanz):** Eine reine Keyword-Seite landet in „Deine Themen"; eine Mention+Keyword-Seite
nur unter Mentions (Dedup-Regel unverändert).
**Step 4: Commit** `feat: rank keyword signal + 'Deine Themen' render group`

---

## Task 4: `setup`-Befehl + Hybrid-Interview (Keywords)

**Files:** Modify `SKILL.md` (neuer Abschnitt „## setup / Onboarding-Interview"), `config.example.yaml`

**Step 1:** `config.example.yaml` um `onboarding.hintShown: false` ergänzen.
**Step 2:** Argument-Routing in §Aufruf/§1: `/confluence-digest setup` → Interview-Modus.
**Step 3:** Interview-Abschnitt schreiben:
1. Identität via `atlassianUserInfo` bestätigen.
2. **Vorschläge sammeln:** mention- und ownEdit-Suche über `now("-90d")`, limit 50, mit
   `expand: "content.metadata.labels"`. Aus den Treffern:
   - Labels: alle `content.metadata.labels.results[].name`.
   - Titel-Terme: Titel tokenisieren, Stoppwörter/Zahlen/Jahreszahlen entfernen, nach Häufigkeit zählen.
   - Kandidatenliste = häufigste Labels + Titel-Terme (Top ~10, dedupliziert, case-insensitive).
3. **Dialog:** Kandidaten zeigen, Nutzer:in wählt aus / streicht / ergänzt frei weitere Keywords.
4. Ergebnis nach `keywords:` in `config.local.yaml` schreiben; `onboarding.hintShown: true` setzen.
5. Anschließend anbieten, direkt einen Digest zu laufen.
**Step 4:** §1 Erststart (keine Config): NEUE Nutzer:innen bekommen direkt das volle Interview
(Identität + Keyword-Schritt), nicht nur das Mini-Onboarding.
**Step 5 (Akzeptanz):** `/confluence-digest setup` zeigt Kandidaten aus echter Aktivität, schreibt
gewählte Keywords in die Config.
**Step 6: Commit** `feat: add setup command + hybrid keyword onboarding interview`

---

## Task 5: Dezenter Hinweis für Bestandsnutzer:innen (§8)

**Files:** Modify `SKILL.md`

**Step 1:** §8 Render-Regel: ist `keywords` leer UND `onboarding.hintShown` false → am Ende des
Digests einmalig eine dezente Zeile: „💡 Tipp: `/confluence-digest setup`, um eigene Themen
hinzuzufügen." Danach `onboarding.hintShown: true` in die Config schreiben.
**Step 2 (Akzeptanz):** Erster Lauf mit leerer Keyword-Liste zeigt den Hinweis genau einmal;
zweiter Lauf nicht mehr.
**Step 3: Commit** `feat: one-time setup hint when no keywords configured`

---

## Task 6: README + Live-Akzeptanz

**Files:** Modify `README.md`

**Step 1:** README: `setup`-Befehl + Keyword-Feature dokumentieren; Status auf „Stufe 1.5" heben.
**Step 2: Live-Akzeptanz** (Controller + Nutzer:in):
- `/confluence-digest setup` → Kandidaten plausibel, Auswahl landet in Config.
- `/confluence-digest --dry-run` → Keyword-CQL je Keyword + Trefferzahlen.
- `/confluence-digest` → „Deine Themen"-Gruppe erscheint; Ranking korrekt; Hinweis weg, sobald Keywords gesetzt.
**Step 3: Commit** `docs: stufe 1.5 usage; mark done`

---

## Out of Scope (Stufe 2+)
- Personen/Teams-Signal + Personen-Teil des Interviews
- Geplanter Lauf (Cloud-Agent), Datei-/Confluence-Ausgabe
- „zuletzt von" = echte letzte bearbeitende Person (Backlog, siehe Design-Doc)
- macOS-App (Stufe 3)
```
