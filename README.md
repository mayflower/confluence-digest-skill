# confluence-digest

Personalisierter Confluence-Digest für Mayflower: ein Claude-Code-Slash-Command, der
org-weit kürzlich geänderte, für dich relevante Seiten findet und einen priorisierten
Überblick mit KI-Zusammenfassungen ausgibt.

**Status:** Stufe 1.5 implementiert (`/confluence-digest` + `setup`-Befehl, Keyword-Signal „Deine
Themen"). Design & Plan siehe [`docs/plans/`](docs/plans/).

## Installation

Voraussetzung: Der MCP-Server **`atlassian-mayflower`** muss eingerichtet und authentifiziert sein
(`/mcp` → `atlassian-mayflower` → authenticate, beim OAuth nur die `mayflowergmbh`-Site freigeben).

Skill installieren (Symlink ins Claude-Code-Skills-Verzeichnis):

```bash
ln -s ~/Code/confluence-digest ~/.claude/skills/confluence-digest
```

Danach Claude Code neu laden. Beim ersten Aufruf erzeugt die Skill automatisch eine
nutzer-lokale `config.local.yaml` (mit deiner via MCP ermittelten Account-ID) – die ist
gitignored und wird nie geteilt.

## Benutzung

```
/confluence-digest            → Standardfenster (24h; montags 72h = Fr–Mo)
/confluence-digest 7d         → Override-Fenster (Nh/Nd, z.B. nach Urlaub)
/confluence-digest --dry-run  → nur CQL + Trefferzahlen, ohne Inhalte/Zusammenfassungen
/confluence-digest setup      → Onboarding-Interview: eigene Themen (Keywords) pflegen
```

**Zusammenfassungen:** Die Top-Treffer (Highlights) bekommen eine ausführliche Summary (2–4 Sätze),
jeder weitere Gruppen-Eintrag eine kurze (2–3 Sätze). Pro Gruppe ist die Zahl der Summaries durch
`limits.groupSummaries` gedeckelt (Default 8); darüber hinaus nur Titel + Link. `limits.highlights`
(Default 5) steuert die Zahl der ausführlichen Highlights.

### Eigene Themen (Keyword-Signal)

Neben „mich betreffend" (Mentions + eigene Bearbeitungen) kannst du **eigene Themen** als
Keywords pflegen. Sie erscheinen im Digest als dritte Gruppe **„🏷️ Deine Themen"**. Das Ranking
gewichtet Mentions am stärksten, dann eigene Bearbeitungen, dann Keyword-Treffer
(Mention > ownEdit > keyword).

Zwei Match-Arten:
- **`signals.keywords`** – Volltext (`text ~`), findet das Wort auch im Seiteninhalt (breit).
- **`signals.titleKeywords`** – nur Titel (`title ~`), schmal. Ideal für breite Orts-/Adressbegriffe
  (z.B. den eigenen Standort), die sonst über Footer/Impressum überall im Volltext auftauchen.

`/confluence-digest setup` startet ein kurzes Interview: Es bestätigt deine Identität und
schlägt Themen aus Labels und Titeln deiner jüngsten Aktivität vor. Du wählst aus, streichst
oder ergänzt frei und die Auswahl landet in deiner `config.local.yaml`. **Neue Nutzer:innen**
durchlaufen dieses Interview automatisch beim ersten Aufruf. Der einmalige dezente Hinweis auf
den `setup`-Befehl betrifft nur Bestandsnutzer:innen aus Stufe 1, die noch keine Keywords gesetzt
haben.

## An Kolleg:innen verteilen

Den Ordner **ohne** `config.local.yaml` weitergeben (z.B. Repo klonen oder Ordner kopieren),
dann denselben Symlink setzen. Jede:r bekommt beim ersten Lauf die eigene Config + Account-ID.
Voraussetzung: eigener authentifizierter `atlassian-mayflower`-MCP.

## Geplante Ausbaustufen

1. **Stufe 1:** Slash-Command, Ausgabe im Chat, Relevanz = „mich betreffend" (Mentions + eigene
   Bearbeitungen) + Mini-Onboarding (Identität automatisch erkennen).
2. **Stufe 1.5:** Erststart-Hybrid-Interview + Keyword-Signal, `setup`-Befehl zum Re-Konfigurieren.
   ✅ implementiert.
3. **Stufe 2:** Personen-Signal (verfolgte Kolleg:innen), geplanter Lauf, Ausgabe als Datei/Confluence-Seite.
4. **Stufe 3:** macOS-Desktop-App auf Basis derselben Logik.

## Design-Eckpunkte (Stand Brainstorming)

- **Relevanz-Signale:** Themen/Keywords · mich betreffend (Mentions/eigene Mitarbeit) · Personen/Teams.
- **Zeitfenster:** Standard 24h, montags 72h (Fr–Mo), Override (z.B. 7d nach Urlaub).
- **Tiefe:** KI-Zusammenfassung pro relevanter Seite.
- **Struktur:** Top-Highlights oben (ausführlich) + nach Signal gruppierte Restliste (kürzer).
- **Verteilbarkeit:** Logik generisch, persönliche Daten in nutzer-lokaler Config (gitignored),
  Account-ID automatisch via Atlassian-MCP ermittelt.
