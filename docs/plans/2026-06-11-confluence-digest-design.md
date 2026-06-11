# Confluence-Digest – Design

**Datum:** 2026-06-11
**Status:** Entwurf validiert (Brainstorming abgeschlossen), bereit für Implementierungsplan Stufe 1

## Zweck

Ein personalisierter Überblick über für mich relevante, kürzlich geänderte Confluence-Inhalte
(mayflowergmbh), um trotz hoher täglicher Aktivität informiert zu bleiben – ohne manuell viele
Spaces durchzusehen.

## Relevanz-Signale

Cross-space (kein fixer Space-Filter), getrieben von drei Signalen:

- **Mich betreffend** – Seiten, in denen ich erwähnt werde (Mention) oder die ich mit-bearbeitet habe.
- **Themen/Keywords** – Volltext-Treffer zu konfigurierten Begriffen, egal in welchem Space.
- **Personen/Teams** – Aktivität bestimmter verfolgter Kolleg:innen.

Stufe 1 aktiviert nur „mich betreffend" (braucht keine Pflege, funktioniert sofort).

## Architektur

Drei klar getrennte Bausteine (Trennung ist Voraussetzung für die spätere Verteilung):

1. **Skill/Command (Logik)** – generisch, enthält keine persönlichen Daten. Das verteilbare Artefakt.
2. **Nutzer-Config (Daten)** – pro Person lokal, gitignored. Keywords, Personen, Feintuning.
3. **Identität** – Atlassian-Account-ID, beim ersten Lauf automatisch via `atlassianUserInfo` ermittelt.

### Ausbaustufen

1. **Jetzt:** Slash-Command `/confluence-digest`, Ausgabe im Chat, Relevanz = „mich betreffend".
2. **Später:** Keywords-/Personen-Signale, geplanter Lauf (Cloud-Agent), Ausgabe als Datei/Confluence-Seite.
3. **Vision:** macOS-Desktop-App auf Basis derselben Logik.

## Konfiguration

Nutzer-Config (z.B. `config.local.yaml`, gitignored), beim ersten Lauf aus Vorlage erzeugt:

```yaml
cloudId: mayflowergmbh          # feste Site
accountId: auto                 # beim 1. Lauf via atlassianUserInfo gesetzt
signals:
  mentions: true                # Seiten, in denen ich erwähnt werde
  ownEdits: true                # Seiten, die ich mit-bearbeitet habe
  keywords: []                  # z.B. ["Symfony", "DSGVO"] – Stufe 2, leer startend
  people:   []                  # Namen/Account-IDs verfolgter Personen – Stufe 2
limits:
  maxSummaries: 8               # Obergrenze KI-zusammengefasster Seiten pro Lauf
```

### Signal → CQL Mapping

Jeweils kombiniert mit `lastmodified >= <Fensterstart>` und `type = page`:

| Signal               | CQL-Kern                              |
|----------------------|---------------------------------------|
| Mentions             | `mention = "<accountId>"`             |
| Eigene Bearbeitungen | `contributor = "<accountId>"`         |
| Keywords             | `text ~ "<keyword>"` (1 Query/Keyword)|
| Personen             | `contributor in ("<id1>","<id2>")`    |

**Teams** sind in CQL nicht direkt filterbar → später zu Mitglieds-Account-IDs auflösen (Stufe 2).

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
6. **Top-N Inhalte holen** (bis `maxSummaries`) via `getConfluencePage`, zusammenfassen. Rest ohne Volltext-Fetch in die Kurzliste.
7. **Rendern** (siehe Output).

### Volumen-Schutz

Liefert eine Query auffällig viele Treffer (zu breites Keyword), auf die neuesten kappen und im
Output hinweisen („+12 weitere zu ‚Symfony' – Keyword evtl. zu breit"). Hält Lauf bezahlbar und
Digest lesbar.

## Output-Format

Markdown, Ausgabe im Chat (Stufe 1):

```
# Confluence-Digest · <Zeitraum>
<Kopfzeile: X relevante Seiten aus Y Spaces>

## ⭐ Highlights (Top 3–5)
### <Seitentitel> · <Space> · <Autor>, <wann>
<2–4 Sätze KI-Zusammenfassung>
**Warum für dich:** <z.B. "Du wirst erwähnt" / "Keyword ‚DSGVO'">
🔗 <Link>

## Nach Signal gruppiert
### 🔔 Dich betreffend (Mentions)
- <Titel> · <Space> · <Autor>, <wann> — <1 Satz> 🔗
### ✏️ Von dir mitbearbeitet
- ...
### 🏷️ Deine Themen
- ... (+12 weitere zu ‚Symfony' – Keyword evtl. zu breit)
### 👥 Verfolgte Personen
- ...
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
- Keine State-Datei in Stufe 1 (Fenster aus Datum).
- Keine Teams-Auflösung, keine Keywords/Personen-Pflege-UI – Stufe 2.

## Test

- **Trockenlauf (`--dry-run`):** zeigt generierte CQL-Queries + Treffer*zahl* pro Signal, ohne
  Inhalte zu holen/zusammenzufassen. Schnell, billig, gut zum Justieren von Keywords.
- Manuelle Verifikation gegen bekanntes Beispiel (Seite, in der ich sicher erwähnt wurde).

## Verteilung an Kolleg:innen

- Command/Skill enthält null persönliche Daten → einfach weitergeben (Plugin/Skill-Ordner oder Git-Repo).
- Jede:r bekommt beim ersten Lauf eigene Config + auto-erkannte Account-ID.
- Voraussetzung dokumentieren: eigener `atlassian-mayflower`-MCP eingerichtet & authentifiziert.

## Offene Punkte für die Implementierung

- Genaue CQL-Unterstützung von `mention` und `contributor` gegen die reale Instanz verifizieren
  (ggf. Alternativen via `searchConfluenceUsingCql` testen).
- Format der relativen Zeitangaben.
- Ablageort der Nutzer-Config (Projektordner vs. `~/.config`).
