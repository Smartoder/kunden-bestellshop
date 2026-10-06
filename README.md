# Online-Bestell-Website für Kassensysteme — Skill

> 🌐 **Auf GitHub, nicht lokal.** Die einzige Quelle und für **jeden Agenten sichtbar**:
> **<https://github.com/HofZeitV12/kunden-bestellshop>**. Kein Klon nötig — der Skill
> kann direkt über die Roh-URL geladen werden (siehe unten). Änderungen stehen in
> [`CHANGELOG.md`](CHANGELOG.md).

**Ein erprobtes Restaurant-Bestellsystem als Vorlage nehmen und für einen neuen Kunden
aufsetzen** — eigenes Branding, eigene Infrastruktur, eigene Kasse.

Gleiches Konzept (Website → Zahlung → Bon), **neues Logo und neue Umgebung** pro Kunde.

Vorlage ist das **fertige, live betriebene Muster „Leckerbissen"** (Tarmstedt, Deutschland)
— eine Bestell-Website mit WinOrder-Kassen-Anbindung und STAR-mPOP-Bon:
**`https://www.leckerbissen.online/website/speisekarte`**.

> **Das Muster ist einmal komplett gebaut und mit echtem Geld bewiesen.** Ein neuer Kunde
> bekommt **dieselbe Website, dieselben Funktionen und denselben Aufbau** — nur mit
> eigener Marke, eigener Infrastruktur und eigener Kasse. Vollständig beschrieben in
> `references/referenz-leckerbissen.md`.

---

## In einen neuen Chat laden — ohne dass es lokal liegt

Cursor, Claude Code oder ein beliebiger Agent mit Netzzugriff:

```
Lade https://raw.githubusercontent.com/HofZeitV12/kunden-bestellshop/main/SKILL.md
und arbeite danach. Lies die references/*.md erst, wenn du an der Stelle bist.
```

Der Agent holt die Anleitung, befragt zuerst den Kunden und baut dann.

**Ohne Netzzugriff** (Agent kann keine URL öffnen): `SKILL.md` und `references/` in das
Projekt kopieren, dann:

```
Lies SKILL.md im Projektstamm und arbeite es ab.
```

Fertige Textbausteine für beide Wege: [COPY-PASTE.md](COPY-PASTE.md)

---

## Auf einem anderen Computer nutzen

Das Repo ist **öffentlich** — der Skill ist damit **von jedem Rechner erreichbar**, ohne
Einladung, ohne Token, ohne dass dieses Repo vorher dort liegt. Kontrolle:

```powershell
Invoke-WebRequest https://raw.githubusercontent.com/HofZeitV12/kunden-bestellshop/main/SKILL.md `
  -UseBasicParsing | Select-Object StatusCode     # erwartet: 200
```

Drei Wege, je nach Rechner:

| Situation | Vorgehen |
|---|---|
| **Agent mit Netzzugriff**, sonst nichts | Nur die Raw-URL laden lassen (siehe oben) — **kein** Klon, keine Installation nötig |
| **Cursor auf dem neuen Rechner**, Skill als `/kunden-bestellshop` | Klonen (siehe „Als Cursor-Skill installieren") — der Ordner **muss** `kunden-bestellshop` heißen |
| **Kein Netzzugriff** | `SKILL.md` + `references/` + `COPY-PASTE.md` herunterladen und ins Projekt legen |

> ⚠️ **Nur die Dateien bewegen, nie Zugangsdaten.** Der Skill enthält **keine** Server-IP,
> keine DB-Kennung und keine Keys — die Werte bleiben in den Umgebungen der einzelnen
> Kundenprojekte. Das ist Absicht und darf sich nicht ändern.
>
> ⚠️ **Auf dem neuen Rechner nicht doppelt ablegen** (siehe Kasten unten). Ein Skill aus
> zwei Orten lädt irgendwann die veraltete Fassung.

---

## Als Cursor-Skill installieren

PowerShell (Windows) — ein Befehl:

```powershell
git clone https://github.com/HofZeitV12/kunden-bestellshop.git `
  "$env:USERPROFILE\.cursor\skills\kunden-bestellshop"
```

macOS / Linux:

```bash
git clone https://github.com/HofZeitV12/kunden-bestellshop.git \
  ~/.cursor/skills/kunden-bestellshop
```

Danach in Cursor `/kunden-bestellshop` aufrufen.

> **Tipp:** Im eigenen Projektordner liegt der Skill unter `.cursor/skills/`. Ein Skill
> in einem Workspace gilt **nur für diesen Workspace**. Für projektübergreifend direkt
> nach `~/.cursor/skills/` installieren (siehe oben).

> ⚠️ **Den Skill nicht doppelt ablegen.** Liegt er **gleichzeitig** im Workspace
> (`.cursor/skills/`), im Agent-Store **und** in `~/.cursor/skills/`, lädt der Agent
> irgendwann die **veraltete** Fassung — und die Fehlersuche beginnt an der falschen
> Stelle. Der Ordnername muss immer `kunden-bestellshop` heißen, sonst lädt der Skill
> nicht. Das Repo ist die **einzige Quelle**.

---

## Was der Skill tut

Der Agent arbeitet sieben Schritte ab. Jeder endet mit einer Meldung: was entstanden,
womit geprüft, was offen.

```
0  Vorlage prüfen        Muster „Leckerbissen" live kontrollieren
1  Kunde befragen        Stammdaten, Zonen, Zeiten, Kasse, Domain
2  Projekt anlegen       Repo, Supabase, Vercel, Stripe, Hetzner
3  Entbranden            Logo, Farben, Texte, Kennungen
4  Daten füllen          Menü, Store-Config, Lieferzonen, Öffnungszeiten
5  Kasse anbinden        Artikelmap, Bridge/Webservice, Hotfolder, Bon
6  Verifizieren          E2E: Bestellung → Zahlung → Mail → Bon
7  Dokumentieren         START.md, PROJEKT.md, Entscheidungen, Runbook
```

Vorher stellt er die **zwölf Intake-Fragen** (Marke, Stammdaten, Liefergebiet, Domain,
Kasse, Zahlung …). Ohne diese Antworten wäre jede Struktur geraten. Er wartet auf die
Antworten, bevor er baut.

---

## Die eine Regel

**Ein Kunde = ein eigenes Projekt.** Eigene Datenbank, eigenes Hosting, eigene Domain,
eigene Zahlung, eigene Kasse. Nichts teilen — außer vielleicht einen Server, dann
aber mit **getrennten Containern, Ports und `.env`**.

**Nie** zwei Restaurants auf **derselben** Datenbank, solange die Kerntabellen keinen
Kunden-Schlüssel tragen.

---

## Kosten und Mengengrenzen (Stand 03.10.2026)

Der Betrieb des Musters ist bewusst **kostengünstig**. Die Grenzen entscheiden, wie
viele Kunden möglich sind:

| Baustein | Tarif im Muster | Grenze |
|---|---|---|
| **GitHub** | kostenlos | privater Speicherplatz — unkritisch |
| **Vercel** | **Hobby** (kostenlos) | Git-Autor muss der **Kontoinhaber** sein; Team-Scope beachten |
| **Supabase** | **Free/Hobby** | eine Instanz pro Kunde → viele kleine Instanzen, nicht eine große geteilte |
| **Hetzner** | Cloud-Server-**Abo** (einstelliger bis ~19 €/Monat) | **1 Server trägt mehrere Kunden** — dann getrennte Container/Ports/`.env`, nie geteilter Mail-Absender |
| **Stripe** | pro Konto | Gebühren pro Transaktion; **eigenes Konto pro Kunde** empfohlen |
| **Resend** | **Free-Tier: 3 Domains**, 3.000 Mails/Monat | ⚠️ **die aktuelle Mengengrenze** — ab dem 4. Kunden höherer Tarif nötig |

> ⚠️ **Resend ist der Flaschenhals.** Jeder Kunde braucht eine **eigene verifizierte
> Absender-Domain**; der Free-Tier erlaubt **3**. Höherer Tarif einplanen, bevor mehr
> als drei Kunden dazukommen.

---

## Dateien

| Datei | Zweck |
|---|---|
| `SKILL.md` | **Der Einstieg.** Ablauf, Entscheidungsbaum, Kurzfassung |
| [`CHANGELOG.md`](CHANGELOG.md) | **Notiz für jeden Agenten:** was zuletzt geändert wurde und was zu beachten ist |
| `references/referenz-leckerbissen.md` | **Das konkrete Muster** (Leckerbissen ↔ WinOrder): Funktionen, Live-Nachweis, Klon-Fahrplan, Werkzeug-Grenzen |
| `references/intake.md` | Fragebogen + Ergebnisform |
| `references/infrastruktur.md` | Supabase, Vercel, Stripe, Hetzner je Kunde |
| `references/architektur.md` | Verifizierte Architektur der Vorlage (Datenfluss, Tabellen, Dateipfade, Env-Katalog) |
| `references/entbranden.md` | Vollständige Rebranding-Map als Abhakliste |
| `references/winorder-kasse.md` | Artikelmap, Bridge, REST-Webservice, Hotfolder, Bon-Druck, Tracking |
| `references/verifikation.md` | E2E-Abnahme + Prüftabelle |
| `references/template-haerten.md` | Vorlage mandantenfähig machen (Kunden-Schlüssel, RLS, Config) |
| `references/regeln-und-fallen.md` | Harte Regeln und teuer gelernte Fehler |

---

## Aktualisieren — und sofort auf allen Rechnern verfügbar

Der Skill gehört **nicht** auf den Kunden-PC und **nicht** in ein Kundenprojekt. Er ist
die **Vorlage**: er liegt hier im Repo und wird von überall geladen.

```powershell
cd <dieses-repo>
git pull
# Änderungen …
git add -A
git commit -m "Skill: …"
git push
```

Nach dem Push ist die neue Fassung **sofort** für jeden anderen Rechner da (der Agent
lädt `main`). Auf einem Rechner, der den Skill **geklont** hat, einmal `git pull` —
danach ist er wieder auf demselben Stand.

Das Repo ist die **einzige Quelle**. Wer den Skill zweimal ablegt (im Repo **und** im
Agent-Store), lädt sonst irgendwann die **veraltete** Fassung.

---

## Herkunft

Abgeleitet aus einem real betriebenen Online-Bestellshop mit Kassensystem-Anbindung und
Bon-Druck. Alle Regeln stammen aus tatsächlichen Schäden: Massendruck, offene
Datenbanken, vertauschte Secrets, verlorene Arbeit.

> **Hinweis zur Namensgebung:** Das **Muster** ist offen benannt („Leckerbissen" ↔
> WinOrder) — es ist die Vorlage und beweist, dass der Weg funktioniert. Zugangsdaten,
> Server-IPs, Datenbank-Kennungen und Keys stehen **nicht** im Repo; sie bleiben in den
> Umgebungen der einzelnen Kundenprojekte. Deshalb ist dieses Repo **öffentlich** und
> trotzdem unbedenklich.
