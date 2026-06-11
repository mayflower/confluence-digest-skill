# confluence-digest

Personalisierter Confluence-Digest für Mayflower: ein Claude-Code-Slash-Command, der
org-weit kürzlich geänderte, für dich relevante Seiten findet und einen priorisierten
Überblick mit KI-Zusammenfassungen ausgibt.

**Status:** Stufe 2a implementiert (`/confluence-digest` + `setup`-Befehl, Keyword-Signal „Deine
Themen" und Personen-Signal „Verfolgte Personen"). Design & Plan siehe [`docs/plans/`](docs/plans/).

## Inbetriebnahme (Schnellstart)

Drei Dinge machen das Tool für dich nutzbar: ein **Atlassian-MCP-Server**, sein Name (**`mcpServer`**)
und deine **`cloudId`**. In vier Schritten:

1. **Atlassian-MCP einrichten & authentifizieren:** `/mcp` → Server → authenticate. Funktioniert mit
   einem lokalen Server oder dem claude.ai-Connector. (Bei eigenem lokalen Server beim OAuth nur die
   gewünschte Site freigeben.)
2. **Skill installieren** (Symlink ins Claude-Code-Skills-Verzeichnis), danach Claude Code neu laden:
   ```bash
   ln -s ~/Code/confluence-digest ~/.claude/skills/confluence-digest
   ```
3. **Config setzen:** Beim ersten Aufruf legt die Skill `config.local.yaml` aus der Vorlage an
   (gitignored, wird nie geteilt). Trage dort zwei Werte ein:
   - **`mcpServer`** = Name deines Atlassian-MCP-Servers (Default `atlassian-mayflower`; z.B.
     `atlassian` oder der Connector `claude_ai_Atlassian`). Die Skill ruft dann `mcp__<mcpServer>__*` auf.
   - **`cloudId`** = cloudId deiner Confluence-Instanz (siehe unten „cloudId ermitteln").
   - Die Account-ID wird automatisch via MCP ermittelt – nichts einzutragen.
4. **Loslegen:** `/confluence-digest setup` (Themen/Personen pflegen, optional) und dann
   `/confluence-digest`.

Mehr als `mcpServer` + `cloudId` musst du nicht anfassen – die Logik in `SKILL.md` bleibt unberührt.

### cloudId ermitteln

`config.example.yaml` enthält `cloudId: "<cloud-id>"` als Platzhalter – trage hier die cloudId
deiner Confluence-Instanz ein (in `config.local.yaml`). So findest du sie:

- Frag Claude im Projekt: *„Welche cloudId hat mein Confluence?"* – nutzt das MCP-Tool
  `getAccessibleAtlassianResources` und gibt die `id` deiner Site zurück, oder
- rufe `https://<deine-site>.atlassian.net/_edge/tenant_info` im Browser auf – das JSON-Feld
  `cloudId` ist der Wert.

Die cloudId ist kein Geheimnis, aber instanzspezifisch – daher als Platzhalter und nicht
fest im öffentlichen Repo.

## Benutzung

```
/confluence-digest            → Standardfenster (24h; montags 72h = Fr–Mo)
/confluence-digest 7d         → Override-Fenster (Nh/Nd, z.B. nach Urlaub)
/confluence-digest --dry-run  → nur CQL + Trefferzahlen, ohne Inhalte/Zusammenfassungen
/confluence-digest setup      → Onboarding-Interview: eigene Themen (Keywords) + Personen pflegen
```

**Inhaltstypen:** Alle Signale erfassen sowohl **Seiten** als auch **Blogposts**
(`type in (page, blogpost)`) – Blogposts sind ein eigener Confluence-Typ, unabhängig von Labels.

**Zusammenfassungen:** Die Top-Treffer (Highlights) bekommen eine ausführliche Summary (2–4 Sätze),
jeder weitere Gruppen-Eintrag eine kurze (2–3 Sätze). Pro Gruppe ist die Zahl der Summaries durch
`limits.groupSummaries` gedeckelt (Default 8); darüber hinaus nur Titel + Link. `limits.highlights`
(Default 5) steuert die Zahl der ausführlichen Highlights.

### Ausführungsmodus

Die Skill wählt den Ausführungsmodus automatisch anhand der Fenstergröße:

- **≤ 72h (Standard 24h, montags 72h) → Inline:** Such-Queries und Zusammenfassungen laufen direkt
  im Haupt-Agenten – wie gehabt.
- **> 72h (z.B. `7d` nach Urlaub) → Subagenten-Fan-out:** Jede Such-Query und jede zu summende Seite
  wird an einen eigenen Subagenten ausgelagert, der nur ein kompaktes Ergebnis zurückgibt. So bleibt
  der Hauptkontext schlank (die großen Rohantworten bleiben im Subagent-Kontext) und die Läufe sind
  nebenläufig. Merge, Dedupe und Ranking bleiben zentral im Haupt-Agenten.

`--dry-run` läuft **immer inline** (nur CQL + Trefferzahlen, kein Fan-out).

**Voraussetzung für den Fan-out:** Die Agent/Task-Fähigkeit (Subagenten) muss in deinem Claude Code
verfügbar sein.

### Eigene Themen (Keyword-Signal)

Neben „mich betreffend" (Mentions + eigene Bearbeitungen) kannst du **eigene Themen** als
Keywords pflegen. Sie erscheinen im Digest als dritte Gruppe **„🏷️ Deine Themen"**. Das Ranking
gewichtet Mentions am stärksten, dann eigene Bearbeitungen, dann Keyword- bzw. Personen-Treffer
(**Mentions > ownEdits > Themen > Personen**).

Zwei Match-Arten:
- **`signals.keywords`** – Volltext (`text ~`), findet das Wort auch im Seiteninhalt (breit).
- **`signals.titleKeywords`** – nur Titel (`title ~`), schmal. Ideal für breite Orts-/Adressbegriffe
  (z.B. den eigenen Standort), die sonst über Footer/Impressum überall im Volltext auftauchen.

### Verfolgte Personen (Personen-Signal)

Als viertes Signal kannst du **Kolleg:innen verfolgen**: Seiten, die eine verfolgte Person
bearbeitet hat, erscheinen im Digest als vierte Gruppe **„👥 Verfolgte Personen"** (jeweils mit
*(von &lt;name&gt;)*). Personen werden in `signals.people` als `{name, id}` gespeichert (`id` =
Atlassian-`accountId`); pro Person läuft eine eigene CQL-Abfrage (`contributor = "<id>"`) für die
Attribution „von wem". In der Rangfolge stehen Personen-Treffer gleichauf mit Keyword-Treffern
(**Mentions > ownEdits > Themen > Personen**); eine Seite landet nur dann in „👥 Verfolgte
Personen", wenn `person` ihr höchstpriores Signal ist.

`/confluence-digest setup` startet ein kurzes Interview: Es bestätigt deine Identität, schlägt
Themen aus Labels und Titeln deiner jüngsten Aktivität vor und fragt anschließend, welche
Kolleg:innen du verfolgen möchtest (Namen werden via Atlassian zur `accountId` aufgelöst; häufige
Mit-Autor:innen werden optional vorgeschlagen). Du wählst aus, streichst oder ergänzt frei und die
Auswahl landet in deiner `config.local.yaml`. **Neue Nutzer:innen** durchlaufen dieses Interview
automatisch beim ersten Aufruf. Das Interview deckt damit **Keywords und Personen** ab. Der
einmalige dezente Hinweis auf den `setup`-Befehl betrifft nur Bestandsnutzer:innen aus Stufe 1,
die noch keine Keywords gesetzt haben.

## An Kolleg:innen verteilen

Den Ordner **ohne** `config.local.yaml` weitergeben (z.B. Repo klonen oder Ordner kopieren),
dann denselben Symlink setzen. Jede:r bekommt beim ersten Lauf die eigene Config + Account-ID.
Voraussetzung: ein eigener authentifizierter Atlassian-MCP-Server (Name in `config.local.yaml` unter `mcpServer`).

## Geplante Ausbaustufen

1. **Stufe 1:** Slash-Command, Ausgabe im Chat, Relevanz = „mich betreffend" (Mentions + eigene
   Bearbeitungen) + Mini-Onboarding (Identität automatisch erkennen).
2. **Stufe 1.5:** Erststart-Hybrid-Interview + Keyword-Signal, `setup`-Befehl zum Re-Konfigurieren.
   ✅ implementiert.
3. **Stufe 2a:** Personen-Signal (verfolgte Kolleg:innen) + Personen-Schritt im Interview. ✅ implementiert.
   **Stufe 2b (geplant):** geplanter Lauf, Ausgabe als Datei/Confluence-Seite.
4. **Stufe 3:** macOS-Desktop-App auf Basis derselben Logik.

## Design-Eckpunkte (Stand Brainstorming)

- **Relevanz-Signale:** Themen/Keywords · mich betreffend (Mentions/eigene Mitarbeit) · Personen/Teams.
- **Zeitfenster:** Standard 24h, montags 72h (Fr–Mo), Override (z.B. 7d nach Urlaub).
- **Tiefe:** KI-Zusammenfassung pro relevanter Seite.
- **Struktur:** Top-Highlights oben (ausführlich) + nach Signal gruppierte Restliste (kürzer).
- **Verteilbarkeit:** Logik generisch, persönliche Daten in nutzer-lokaler Config (gitignored),
  Account-ID automatisch via Atlassian-MCP ermittelt.

## Lizenz

MIT – siehe [LICENSE](LICENSE). © 2026 Mayflower GmbH.
