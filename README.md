# confluence-digest

Personalisierter Confluence-Digest für Mayflower: ein Claude-Code-Slash-Command, der
org-weit kürzlich geänderte, für dich relevante Seiten findet und einen priorisierten
Überblick mit KI-Zusammenfassungen ausgibt.

**Status:** in Entwurf. Aktuelles Design siehe [`docs/plans/`](docs/plans/).

## Geplante Ausbaustufen

1. **Stufe 1:** Slash-Command, Ausgabe im Chat, Relevanz = „mich betreffend" (Mentions + eigene
   Bearbeitungen) + Mini-Onboarding (Identität automatisch erkennen).
2. **Stufe 1.5:** Erststart-Hybrid-Interview + Keyword-Signal, `setup`-Befehl zum Re-Konfigurieren.
3. **Stufe 2:** Personen/Teams-Signal, geplanter Lauf, Ausgabe als Datei/Confluence-Seite.
4. **Stufe 3:** macOS-Desktop-App auf Basis derselben Logik.

## Design-Eckpunkte (Stand Brainstorming)

- **Relevanz-Signale:** Themen/Keywords · mich betreffend (Mentions/eigene Mitarbeit) · Personen/Teams.
- **Zeitfenster:** Standard 24h, montags 72h (Fr–Mo), Override (z.B. 7d nach Urlaub).
- **Tiefe:** KI-Zusammenfassung pro relevanter Seite.
- **Struktur:** Top-Highlights oben (ausführlich) + nach Signal gruppierte Restliste (kürzer).
- **Verteilbarkeit:** Logik generisch, persönliche Daten in nutzer-lokaler Config (gitignored),
  Account-ID automatisch via Atlassian-MCP ermittelt.
