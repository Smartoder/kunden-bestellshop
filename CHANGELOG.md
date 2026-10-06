# Changelog — Änderungen am Skill

Kurz und chronologisch. **Diese Datei ist die Notiz für jeden Agenten**, der den Skill
auf einem anderen Rechner lädt: hier steht, was zuletzt geändert wurde, in welchem
Commit — und was beim Weiterarbeiten zu beachten ist.

Immer **neueste Einträge oben**. Nach jeder Änderung am Skill **im selben Vorgang**
ergänzen und mit `git push` veröffentlichen (siehe `README.md` → „Aktualisieren").

---

## 2026-10-06 — Bon-Fehler behoben und am Bon bewiesen (Umlaute, Adresse, Zahlungsart)

**Anlass:** Drei Fehler auf einem echten Bon (Bestellung #43): Umlaute verstümmelt
(`BÃ¶lstedter StraÃŸe`), Hausnummer fehlte im Bon-Feld, und „Online bezahlt" stand auf
dem Bon, obwohl der Kunde **bar** gewählt hatte. Der Betreiber hat die drei Punkte auf
dem Papier markiert.

**Wichtigste Erkenntnis: die Kasse war nicht die Ursache.** Die Kassen-DB und der
Export lieferten sauberes UTF-8 (`C3 B6`); unsere Bon-Datei hatte doppelt kodierte
Bytes (`C3 83 C2 B6`). Die Kasse schreibt in ihren **eigenen** Ausgabedateien
korrektes UTF-8 (`C3 BC` in `Servicegebühr`). Der Fehler entstand **beim Lesen**.

| Fehler | Ursache | Behoben in |
|---|---|---|
| Umlaute doppelt kodiert | `Invoke-RestMethod` dekodiert unter PowerShell 5.1 mit dem System-ANSI-Zeichensatz, wenn der Server keinen `charset` mitschickt | `WebRequest` + `Encoding::UTF8` |
| Hausnummer fehlte (`Straße.1a`) | **Punkt** vor der Nr. galt nicht als Trenner — nicht der Umlaut | Zerlegung akzeptiert Punkt, Bindestrich, Slash, Bereiche (`11-13`) |
| Falsches Zahlungsmittel | `Bar` / `Kartenzahlung` sind **keine** Stammdatenwerte dieser Kasse | echte Werte: `Barzahlung` / `EC-Karte` |
| Bon zeigte trotz Fix das Alte | der laufende Prozess hatte den Skriptstand von **zwei Wochen** vorher | Neustart + Startzeit prüfen |

**Nachweis:** Rechnung **#6636** — `<PaymentType>Barzahlung</PaymentType>`,
`Street=Bölstedter Straße` (HEX `C3 B6`), `HouseNo=1a`, 0 doppelt kodierte Stellen.
Der Betreiber hat es auf dem Bon bestätigt: *„bon und strasse kommt alles sauber"*.

| Datei | Änderung |
|---|---|
| `references/winorder-kasse.md` | **Zahlungsarten auslesen statt raten** (neuer Abschnitt mit Beispiel-Tabelle und der Falle `Bar` → `Barzahlung`). **Nach jeder Format-Änderung den Prozess neu starten** (inkl. Wächter-Prüfung und **Einzelinstanz-Riegel für den Wächter**). Encoding-Abschnitt um die Bestelldaten-Falle erweitert. Diagnose-Checkliste von 7 auf 11 Punkte (Prozessstand, Umlaute, Zahlungsmittel, Hausnummer). |
| `references/regeln-und-fallen.md` | **Falle 8 erweitert** (zweite Hälfte der UTF-8-Falle: die Bestelldaten, mit Byte-Tabelle und Prüfbefehl). **Neu Falle 12a** (Wert im Code, aber die Kasse kennt ihn nicht). **Neu Falle 12b** (Code geändert, laufender Prozess hat ihn nie geladen — inkl. Wächter). |

### Nachtrag — der Wächter brauchte selbst einen Riegel

Beim Aufräumen zeigte sich ein Folgefehler: der **Wächter** (Aufpasser) hatte
**keinen** Einzelinstanz-Riegel, nur der Abhol-Prozess. Zwei Wächter starteten den
Prozess deshalb gegenseitig alle ~15 Sekunden neu — der Kassen-Anschluss **flackerte**.

- Ein Mutex (`…BridgeWatcher`) beendet den zweiten Wächter sofort
- **Ein einzelner Autostart-Eintrag ist kein Nachweis:** der Run-Eintrag war
  vorhanden, wurde aber nicht ausgeführt. Ergänzt um eine **Verknüpfung im
  Startordner** — erst der Riegel macht zwei Autostart-Wege gefahrlos

> **Merksatz:** Wer zwei Autostart-Wege anlegt, muss vorher den doppelten Start
> verhindern. Sonst bekämpfen sich die Instanzen.

**Für jeden Agenten:** Bei einem Bon-Fehler **zuerst die Bytes der eigenen Datei
prüfen** (`C3 83 C2 xx` = doppelt kodiert), **nicht** die Kasse verdächtigen. Und:
ein Fix am Formatter wirkt **erst nach einem Neustart** des Abhol-Prozesses — sonst
läuft die alte Logik weiter und der Bon bleibt falsch, obwohl der Code stimmt.

---

## 2026-10-05 — Git-Autor-Regel präzisiert (Deploy-Blockade `COMMIT_AUTHOR_REQUIRED`)

**Anlass:** Ein Produktions-Deploy des Musters blieb `BLOCKED` — **ohne Build (0 ms)**.
Der Skill nannte bisher nur die Erscheinungsform „not a member". Der tatsächliche Code
war `COMMIT_AUTHOR_REQUIRED`: die **E-Mail** des Autors gehörte zu keinem Konto.
Fünf vorherige Commits **desselben Autors** waren noch durchgelaufen — die Prüfung
greift ab einem Zeitpunkt, nicht rückwirkend.

**Kernaussage:** Der **Name** des Autors ist frei, die **E-Mail** nicht. Vercel ordnet
über `githubCommitAuthorEmail` zu. Bei Hobby + privatem Repo muss der Autor der
**Kontoinhaber** sein. „Commits nur als <markenname>" genügt **nicht**, wenn dessen
E-Mail eine Fantasie-Adresse ist.

| Datei | Änderung |
|---|---|
| `SKILL.md` | Kritikalität 5 neu gefasst: beide Codes in einer Tabelle, „Name frei, E-Mail nicht", Fix per `--amend --author` + `push --force-with-lease`, Diagnose über `vercel inspect --json` / `readyStateReason`. Diagnose-Tabelle: „Git-Autor = **Kontoinhaber**". |
| `references/regeln-und-fallen.md` | Falle 11 neu gefasst: `COMMIT_AUTHOR_REQUIRED` ↔ „not a member", Nachweis vom 05.10.2026, Diagnose zuerst lesen. |
| `references/infrastruktur.md` | Repo-Checkliste + Warnhinweis auf „Kontoinhaber" umgestellt. |
| `references/referenz-leckerbissen.md` | Repo-Zeile: Umzug `HofZeitV12/…` → `Smartoder/…` („This repository moved"). Werkzeug-Stand um **05.10.2026** ergänzt: Vercel-MCP-Plugin `403`, **Token-Variante** `HTTP 200` (Ersatzweg ohne OAuth). Neuer Abschnitt **8.2**: `BLOCKED` ist kein Code-Fehler. |
| `README.md` | Kosten-Tabelle: „Git-Autor muss **Kontoinhaber** sein". |

**Für den Agenten:** Bei einem `BLOCKED`-Deploy **zuerst `readyStateReason` lesen**
(`vercel inspect --json <url>`), **nicht** den Code durchsuchen. Ein grüner
Deploy-Verlauf beweist **nichts** — die Autorenprüfung kann jederzeit greifen.
Startseite danach live prüfen (`age: 0`), nicht nur HTTP 200.

---

## 2026-10-03 (abends) — Konsistenz nach der Muster-Umstellung

**Anlass:** Nach dem Umbenennen der Vorlage auf das konkrete Muster blieben drei
Stellen widersprüchlich. Korrigiert, damit die GitHub-Seite in sich stimmig ist.

| Commit | Inhalt |
|---|---|
| `af49077` | `README.md` „Herkunft": **nicht mehr** „neutralisiert" (widersprach dem offen benannten Muster). Klarstellung: das **Muster ist benannt**, Zugangsdaten bleiben draußen — deshalb darf das Repo öffentlich sein. `README.md` Ablauf **Schritt 0** auf „Muster „Leckerbissen" live kontrollieren" gezogen (vorher generisch). |

**Für den Agenten:** geprüft am 03.10.2026 — kein Widerspruch mehr zwischen
`SKILL.md`, `README.md`, `CHANGELOG.md` und `references/*`. Die Notizen für andere
Agenten stehen hier im Changelog; die ausführliche Analyse (Dateiliste,
Abweichungstabelle, MCP-Stand, Live-Nachweis) liegt als Canvas
`skill-audit-kunden-bestellshop` im Cursor-Projekt `c-Users-lecke-Downloads-WinOrder`.

---

## 2026-10-03 (abends) — Skill als GitHub-Quelle sichtbar gemacht

**Ziel:** Der Skill soll **auf GitHub sichtbar** sein — **nicht** auf einem lokalen
Rechner. Ein fremder Agent lädt ihn über die Roh-URL, ohne Klon.

| Commit | Inhalt |
|---|---|
| `f030c06` | `CHANGELOG.md` angelegt (diese Datei) + Verweise in `README.md`/`SKILL.md` |
| danach | Hinweis „liegt auf GitHub, nicht lokal" oben in `SKILL.md` und `README.md` |

**Für den Agenten:** die einzige Quelle ist das **öffentliche** Repo
`HofZeitV12/kunden-bestellshop`, Branch `main` → `SKILL.md`. `CHANGELOG.md` zuerst
lesen, damit kein veralteter Stand benutzt wird.

---

## 2026-10-03 (abends) — Audit gegen das laufende Muster „Leckerbissen"

**Geprüft:** der Skill (16 Dateien), das Vorlage-Repo `HofZeitV12/leckerbissen-speisekarte`
(lokal **und** remote), Webhook-Server (`/health`, `/api/status`, `/api/webhooks/stripe`),
Datenbank-Zugriff und die MCPs (GitHub, Resend, Vercel, Supabase, Hetzner).

**Belege:** die vollständige Analyse liegt als Canvas `skill-audit-kunden-bestellshop`
im Cursor-Projekt `c-Users-lecke-Downloads-WinOrder` (Dateiliste, Abweichungstabelle,
MCP-Stand, Live-Nachweis).

**Commits in diesem Repo (Branch `main`):**

| Commit | Inhalt |
|---|---|
| `4a32b1b` | Muster „Leckerbissen" konkret verankert (`references/referenz-leckerbissen.md`) |
| `6201433` | MCP-Stand 03.10. korrigiert; **neuer Abschnitt 8.1** „Vorlage nicht eingefroren"; **Falle 14** (Fork von altem Stand); erfundene Tabelle `project_memory_history` entfernt |
| `0aa4321` | Nutzung von jedem Rechner dokumentiert (public/`main`, Klon, ZIP-Fallback) |
| `499866b` | README-Abschnitt „Aktualisieren" entdoppelt |

**Was der andere Agent wissen muss:**

- Der Skill ist **öffentlich** unter `HofZeitV12/kunden-bestellshop`, Branch `main`.
  Laden: `https://raw.githubusercontent.com/HofZeitV12/kunden-bestellshop/main/SKILL.md`
- **Vor jedem Fork aktualisieren:** `git fetch` + `git rev-list --count HEAD..origin/main`
  muss **0** sein. Die Vorlage war am 03.10. **8 Commits hinter `main`** — ein alter
  lokaler Stand hat `check`/`types`/`suche` **nicht** (Skripte hängen am **Stand**, nicht
  am Projekt). Details: `references/referenz-leckerbissen.md`, Abschnitt 8.1.
- **MCP-Stand 03.10.2026:** GitHub, Resend und Vercel **OK** (Vercel antwortet wieder —
  war vormittags noch `403`). **Supabase-MCP** hat **keinen Zugriff** auf die geteilte
  „Üben"-Instanz (nur HTTP-API als Ersatzweg). **Hetzner-MCP** nicht erreichbar — Port
  `11436` von diesem Rechner zu, **SSH-Tunnel** über Port 22 als Weg.
- **Keine Zugangsdaten im Repo.** Server-IP, DB-Kennung und Keys bleiben in den
  Kundenumgebungen. Das ist Absicht — deshalb darf das Repo öffentlich sein.

---

## 2026-10-01 — Erste Fassung

| Commit | Inhalt |
|---|---|
| `0fda919` | neutraler Name `kunden-bestellshop`, Ablauf + Referenzen |
| `439b8a0` | Merge: eigene, verifizierte Fassung gewinnt |
