---
name: kunden-bestellshop
description: Eine Online-Bestell-Website fuer Kassensysteme in Deutschland (Speisekarte, Warenkorb, Lieferung/Abholung, Online-Zahlung, Bon auf dem Kassensystem) als Vorlage nehmen und fuer einen NEUEN Restaurant-Kunden aufsetzen - eigenes Branding, eigene Infrastruktur (Supabase, Vercel, Stripe, Hetzner-Container), eigene Kasse und eigener Bon-Druck. Use when a new restaurant customer needs its own ordering website based on a proven German ordering-and-cash-register (Kassensystem) template, when cloning or duplicating the template for another restaurant, when onboarding a second restaurant onto the same concept with a new logo/branding, when rebranding an existing fork, or when asked "wie setze ich das gleiche System fuer einen neuen Restaurant-Kunden auf". Also use before any schema change to the template and when a customer project must stay cleanly separated from the template project.
---

# Online-Bestell-Website für Kassensysteme aufsetzen (Deutschland)

> 🌐 **Dieser Skill liegt auf GitHub — nicht auf einem Rechner.** Das ist die einzige
> Quelle und für **jeden Agenten sichtbar**:
> **<https://github.com/HofZeitV12/kunden-bestellshop>**
>
> In einen fremden Chat laden (nur diese URL, kein Klon nötig):
> ```
> Lade https://raw.githubusercontent.com/HofZeitV12/kunden-bestellshop/main/SKILL.md
> und arbeite danach. Lies die references/*.md erst, wenn du an der Stelle bist.
> ```
> **Was zuletzt geändert wurde**, steht in `CHANGELOG.md` (im selben Repo) — dort zuerst
> nachsehen, damit kein veralteter Stand benutzt wird.

Ein **erprobtes, live betriebenes Bestellsystem** wird zur **Vorlage**. Pro Kunde
entsteht daraus ein **eigenes Projekt** mit eigenem Logo, eigener Infrastruktur und
eigener Kasse — **gleiches Konzept, nicht gleiche Umgebung**.

```
Kunde (Browser) → Website (Vercel) → Zahlung (Stripe)
   → Webhook + Bestätigungsmail (Hetzner-Container) → Datenbank (Supabase)
   → Kassen-PC (Bridge/Webservice) → Kassen-Software → Bon auf dem Thermodrucker
```

> **Maßstab dieses Skills:** jede Aussage über den Zustand braucht einen **ausgeführten
> Befehl**. „Müsste laufen" zählt nicht. Projektkennungen, Tokens und Serverwerte
> stehen **nicht** hier — nur **wo** sie liegen und **wie** man sie ermittelt.

---

## Das Referenzsystem (die Vorlage) — konkret

**Vorlage ist das laufende Muster „Leckerbissen"** (Tarmstedt, Deutschland): eine
Bestell-Website mit Anbindung an das Kassensystem **WinOrder** und Bondrucker
**STAR mPOP**. Es ist **einmal komplett gebaut und mit echtem Geld bewiesen** — der
neue Kunde bekommt **dieselbe Website, dieselben Funktionen und denselben Aufbau**,
nur mit **neuer Marke und eigener Infrastruktur**.

**🔗 Muster-Bestellseite:** `https://www.leckerbissen.online/website/speisekarte`
Vollständig beschrieben in **`references/referenz-leckerbissen.md`**.

Prüfe das Muster **zuerst** gegen die Wirklichkeit. Es ist der Beweis, dass der
Bestellweg funktioniert. Die Werte unten wurden am **03.10.2026** von außen **live**
geprüft:

| Baustein | Was es ist | Geprüft mit | Ergebnis |
|---|---|---|---|
| **Website** | Bestellseite `/website/speisekarte` | `GET` der Seite | **HTTP 200** |
| **Speisekarte** | Menü-Endpunkt der Website | `GET /api/menu` | **HTTP 200, 51 Artikel** |
| **Stammdaten** | Laufzeit-Config der Website | `GET /api/store` | **HTTP 200** (genau 5 Felder) |
| **Datenbank** | Supabase-Instanz (EU-Region) | Tabellen + Zeilen lesen | `orders`, `menultems` (51), `project_memory` |
| **Server** | Hetzner-Container: Zahlungs-Webhook, Bestätigungsmail | `GET /health` | **200 `status: ok`** |
| **Kasse** | Partner-Export der Website | `GET` **ohne** Key | **HTTP 401** (Auth greift) |
| **Tresor** | Diagnose des Webhook-Servers | `GET /api/status` **ohne** Token | **HTTP 401** (Auth greift) |

**Der Kernbeweis:** Eine Online-Bestellung läuft **ohne manuelles Kopieren** durch
bis zum Bon — Website → Zahlung → Datenbank → Bridge/Webservice → Kasse → Bon
(im Muster mit echter Karte belegt: Order #32 → Rechnung #6457).

> ⚠️ **Das Muster ist die Vorlage, nicht der Bauplan für den Kunden.** Die Muster-DB
> ist mit fremden Projekten **geteilt** und hat **kein RLS** (Supabase meldet das als
> *kritisch*). Das ist **kein** Muster zum Nachbauen: für jeden Kunden gilt **eigene
> Instanz** (Standardweg) und **eigene Trennung**.
>
> 🔒 **Keine Zugangsdaten im Skill.** Dieses Repo ist **öffentlich**. Server-IP,
> SSH-Schlüssel, Datenbank-Kennungen, Keys und Projekt-IDs stehen **nicht** hier,
> sondern im Zugangsregister des jeweiligen Projekts — der Skill nennt nur **wo** sie
> liegen und **wie** man sie ermittelt.

---

## Die eine Regel, die alles andere entscheidet

**Ein Kunde = ein eigenes Projekt.** Eigene Datenbank, eigenes Hosting, eigene Domain,
eigene Zahlungsumgebung, eigener Kassen-Anschluss.

**Niemals** zwei Restaurants auf **derselben** Datenbank, solange die Kerntabellen
keinen Kunden-Schlüssel tragen. Die geteilte Datenbank ist die teuerste Falle dieses
Systems — siehe `references/template-haerten.md`.

| Ebene | Trennung |
|---|---|
| Repository | eigenes Repo, eigener Name, frischer Git-Start |
| Datenbank | eigene Supabase-Instanz (Standardweg) |
| Hosting | eigenes Vercel-Projekt + eigene Domain |
| Zahlung | eigene Keys + eigene Webhook-Endpunkte |
| Server | eigener Container/Port auf dem Server, eigene Server-`.env`, eigener Mail-Absender |
| Kasse | eigener Hotfolder-Pfad **oder** eigener Webservice-Zugang, eigene Artikelmap |

**Faustregel:** Teilen zwei Kunden irgendwo denselben Schlüssel, dieselbe Tabelle oder
denselben Endpunkt, ist die Trennung nicht vollständig.

---

## Der Ablauf

```
0.  Vorlage prüfen        Muster „Leckerbissen" live kontrollieren (Tabelle oben)
1.  Kunde befragen        zwölf Fragen: Stammdaten, Zonen, Zeiten, Kasse, Domain
                          → references/intake.md
2.  Projekt anlegen       Repo, Supabase, Vercel, Stripe, Hetzner
                          → references/infrastruktur.md
3.  Entbranden            Logo, Farben, Texte, Kennungen
                          → references/entbranden.md
4.  Daten füllen          Menü-Seed, Stammdaten, Lieferzonen, Öffnungszeiten
5.  Kasse anbinden        Artikelmap, Bridge/Webservice, Hotfolder, Bon
                          → references/winorder-kasse.md
6.  Verifizieren          E2E: Bestellung → Zahlung → Mail → Bon
                          → references/verifikation.md
7.  Dokumentieren         START.md, PROJEKT.md, Entscheidungen, Runbook
                          → über Skill `project-blueprint`
```

**Jede Phase endet mit einer Meldung:** was entstanden ist, womit es geprüft wurde,
was offen blieb. Vorlage: Ende von `references/verifikation.md`.

---

## Entscheidungsbaum

```
Soll der Kunde auf der geteilten Vorlage-DB laufen?
├─ JA  → ⚠️ NICHT ohne Kunden-Schlüssel auf den Kerntabellen.
│        Erst references/template-haerten.md abarbeiten. (Nicht empfohlen.)
└─ NEIN → Eigene Supabase-Instanz pro Kunde.  ← Standardweg
          → references/infrastruktur.md, Abschnitt „Datenbank"

Wie holt die Kasse die Bestellungen?
├─ Hotfolder-Bridge  → Datei je Bestellung in einen überwachten Ordner
│                      (Bridge läuft auf dem Kassen-PC; Standardweg der Vorlage)
└─ REST-Webservice   → die Kasse ruft selbst eine URL ab (kein Dauerprozess nötig)
                       Beides: → references/winorder-kasse.md
```

**Basis ist immer das Muster „Leckerbissen"** — dieselbe Bestell-Website, nur mit
neuer Marke und neuer Umgebung. Den vollständigen Klon-Fahrplan und die Werkzeug-Grenzen
enthält `references/referenz-leckerbissen.md`.

---

## Die fünf Kritikalitäten des Bestellwegs

### 1. Zwei Stripe-Endpunkte pro Kunde

Ein Endpunkt auf der **Hosting-Domain** (Status + Kundendaten), einer auf
`webhook.<kunde>.de` (Status + **Bestätigungsmail**). Nur der Server verschickt Mails.
Der Mail-Marker (`bestaetigung_mail_am`) wird **ausschließlich** vom Mailserver
geschrieben — **nicht** an `bezahlt_am` hängen (Race Condition, in der Vorlage real
passiert und behoben).

### 2. Modus-Wache statt Signatur-Vertrauen

Test- und Live-Events sind **beide** gültig signiert — mit **verschiedenen** Secrets.
Der Riegel ist die Kombination aus Soll-Modus (`STRIPE_EXPECTED_MODE`) und
Ist-Modus (`event.livemode`). Bei Konflikt → **409**, keine Datenbankschreibung. Der
Schlüssel-Präfix allein (`sk_`/`rk_` test/live) genügt nicht.

### 3. Caddy: vier Pfade, Rest 404

Der öffentliche Webhook-Host lässt nur diese Pfade durch:
`/api/webhooks/stripe*`, `/health`, `/api/status`, `/api/mail-retry`. Fehlt einer,
ist die Diagnose von außen tot. Der **Kassen-Export gehört nicht hierher**.

### 4. Der Export ans Kassensystem läuft auf dem Hosting

Der Partner-Export (`/api/export/winorder`) läuft auf dem **Vercel-Projekt**, nicht auf
dem Webhook-Server. Die Kasse ruft ihn mit **eigenem** Key auf. Ein Aufruf **ohne**
Key muss **HTTP 401** liefern — das ist der Live-Test, dass die Auth greift.

### 5. Git-Autor = Inhaber des Hosting-Kontos

Vercel-Hobby bricht den Deploy ab, wenn der Git-Autor **keinem** Konto zugeordnet
werden kann. Der Commit wird dann `BLOCKED` — **ohne Build (0 ms)**, aber mit
`readyStateReason` und `seatBlock.blockCode`. Zwei Codes sind zu unterscheiden:

| Code | Bedeutung | Maßnahme |
|---|---|---|
| `COMMIT_AUTHOR_REQUIRED` | Autor keinem Git-/Hosting-Konto zuordenbar | **E-Mail** des Autors korrigieren |
| „not a member" | Autor zuordenbar, aber nicht (mehr) im Team | Autor ins Team holen oder E-Mail ändern |

**Der Name ist frei, die E-Mail nicht.** Vercel ordnet über die **E-Mail** zu
(`githubCommitAuthorEmail`), nicht über den Anzeigenamen. Eine Fantasie-Adresse wie
`…@users.noreply.github.com` genügt **nicht**, wenn sie zu keinem Konto gehört.
Autor **vor dem ersten Commit** setzen:

```bash
git config user.name  <markenname>            # egal, z. B. HofZeitV12
git config user.email <kontoinhaber@mail>     # MUSS dem Hosting-Konto zugeordnet sein
```

Bei Hobby + privatem Repo gilt zusätzlich: **nur der Kontoinhaber** darf der Autor
sein (Kollaboration gibt es dort nicht). Der Name, unter dem die Kasse und die Marke
firmieren, hat darauf keinen Einfluss — er darf der Kundenname bleiben.

**Falls der Deploy schon blockiert ist** (der Inhalt ist bereits gepusht):

```bash
git config user.email <kontoinhaber@mail>
git commit --amend --no-edit --author "<markenname> <kontoinhaber@mail>"
git push --force-with-lease origin main      # Baum bleibt unveraendert, nur der Autor
```

**Zuerst den Grund lesen, nicht den Code suchen.** Blockierte Deploys sind kein
Code-Fehler:

```bash
vercel inspect --json <deploy-url>           # readyState, readyStateReason
# oder direkt: GET /v13/deployments/<id>?teamId=<team>  ->  readyStateReason, seatBlock
```

---

## Was ein Kunde **immer** braucht

Ohne diese sechs Angaben ist jede Struktur geraten. Volle Liste:
`references/intake.md`.

1. **Marke** — Name, Logo-Datei, Primärfarbe(n), Slogan, Sprache
2. **Stammdaten** — Adresse, Telefon, E-Mail, Öffnungszeiten, Zubereitungszeit
3. **Liefergebiet** — PLZ-Liste, Mindestbestellwert, Liefergebühr je Zone
   (und ob es eine **Aktion/Rabatt** geben soll, z. B. „Lieferung gratis")
4. **Domain** — welche Domain, wer besitzt sie, DNS-Zugang
5. **Kasse** — welches System, wie heißt der Artikelstamm, wie kommen Bestellungen an
6. **Zahlung** — eigenes Konto oder geteiltes? Test- oder Live-Start?

**Erst fragen, dann bauen.** Nichts erfinden, was der Kunde beantworten kann.

---

## Die Rebranding-Berührungspunkte (Kurzfassung)

Marke und Kunde stecken **verteilt** im Code, nicht an einer Stelle. Vollständige
Abhakliste: `references/entbranden.md`.

| Bereich | Typische Datei(en) |
|---|---|
| Farben/Theme | Theme-Konfiguration (`tailwind.config.js`) + `globals.css` |
| Logo/Bilder | Header-Komponente, Hero, Hero-Bilderliste, `public/` |
| Stammdaten | Store-Route + Wissensspeicher-Key `arch.store_config` (**beide**) |
| Liefergebiet | Lieferzonen-Modul (PLZ, Mindestwert, Gebühr, Fehlertext, **Aktion/Rabatt**) |
| Öffnungszeiten/Wunschzeit | Öffnungszeiten-Modul |
| Rechtstexte | Impressum, Datenschutz, AGB |
| Kassen-Artikelmap | `lib/…/articles.ts` **und** `tools/…-articles.json` |
| Kassen-Formattexte | Format-Modul (`StoreName`, `Agent`, `Referer`, `OrderID`-Präfix, `PaymentType`) |
| Kundenmail | Webhook-Server: E-Mail-Vorlagen, Absender |
| App/PWA | Manifest, Service Worker, Icon-Generator, Capacitor |
| Kennungen im Code | `RESTAURANT`-Konstante, hartcodierte Namen, Middleware-Realm, `SITE_URL`-Defaults |

> ⚠️ **Markenreste in Klassennamen.** Ein Fork kopiert historisch gewachsene Farbnamen
> mit (in der Vorlage 20+ Dateien). Beim Entbranden **alle** Stellen prüfen — nicht nur
> die `-primary`-Tokens. Die Abschluss-Suche in `references/entbranden.md` findet sie.

---

## Diagnose: den Zustand jedes Bausteins prüfen

Nimm dir **vor Schritt 1** diese Diagnose vor und dokumentiere die echten Antworten.
Fehlt ein MCP-Tool in deiner Umgebung, prüfe ersatzweise über HTTP(S) oder die Konsole
des Anbieters.

| Baustein | Prüfung | Erwartet |
|---|---|---|
| Supabase | Instanzen auflisten, Zielinstanz wählen | Status **healthy** |
| Supabase | Tabellen + Zeilen lesen (`orders`, `menultems`, `project_memory`) | vorhanden, plausibel |
| Supabase | Sicherheits-/Leistungshinweise abrufen | keine kritischen offen |
| Supabase | MCP-Zugriff auf die Zielinstanz (nicht jede ist freigegeben!) | sonst **HTTP-API** als Ersatzweg |
| Vercel | Projekte des Teams auflisten | Kundenprojekt gelistet |
| Vercel | Umgebungsvariablen prüfen (Namen, Modus) | vollständig, richtiger Modus |
| Vercel | **Git-Autor** = Kontoinhaber (bei Hobby-Pflicht!) | Deploy **nicht** `BLOCKED` / „not a member" |
| GitHub | Repo + Hauptzweig lesen, `.env.example` | nur Platzhalter |
| GitHub | **Vorlage auf `origin/main` aktualisieren** (`git rev-list --count HEAD..origin/main` = 0) | kein Fork von altem Stand |
| Hetzner | Server auflisten, Zielserver lesen | **running**, Firewall aktiv |
| Hetzner | Firewall-Regeln lesen | nur 22/80/443 (+ Diagnose-Port) |
| Hetzner | Container-Health (`/health`) | **200 `status: ok`** |
| Hetzner | Diagnose-Port (`11436` o. ä.) von hier erreichbar? | sonst **SSH-Tunnel** über Port 22 |
| Stripe | Webhook-Endpunkte im Dashboard | `livemode`, `url`, `status` je Endpunkt |
| Live | Website, `/api/menu`, `/api/store` | **HTTP 200** |
| Live | Export **ohne** Key; Diagnose **ohne** Token | **HTTP 401** |

> ⚠️ **Ein MCP „grün" ist kein Nachweis.** Fehlt ein MCP-Tool oder ist es im
> Fehlerzustand, wird ersatzweise per **HTTP, SSH oder Dashboard** geprüft. Die
> tatsächlich beobachteten Grenzen der MCPs stehen in
> `references/referenz-leckerbissen.md`, Abschnitt 8.

---

## Dokumentation gehört zum Ergebnis

Ein Kundenprojekt ohne Doku gilt als **nicht fertig**. Nach dem Aufsetzen:
`docs/START.md` (Einstieg), `docs/PROJEKT.md` (Zweck + „was nicht"),
`docs/entscheidungen/` (warum), `docs/RUNBOOK.md` (Störfall). Vorlage und Ablauf:
Skill **`project-blueprint`**.

> 🤝 **Notiz für andere Agenten:** Änderungen an **diesem** Skill stehen in
> [`CHANGELOG.md`](CHANGELOG.md) (neueste zuerst). Wer den Skill auf einem anderen
> Rechner laden soll, liest zuerst `README.md` → „Auf einem anderen Computer nutzen"
> und dort das `CHANGELOG.md` — so ist klar, welcher Stand gilt und was zu beachten ist.

---

## Referenzen

| Datei | Inhalt |
|---|---|
| `references/referenz-leckerbissen.md` | **Das konkrete Muster** (Leckerbissen ↔ WinOrder): Funktionen, Live-Nachweis, Klon-Fahrplan, Werkzeug-Grenzen |
| `references/intake.md` | Fragebogen + Ergebnisform |
| `references/infrastruktur.md` | Zielarchitektur + Supabase, Vercel, Stripe, Hetzner je Kunde, Umgebungsvariablen-Katalog |
| `references/architektur.md` | Verifizierte Daten- und Dateipfade der Vorlage (Soll-Zustand, den der Fork übernimmt) |
| `references/entbranden.md` | Vollständige Rebranding-Map als Abhakliste |
| `references/winorder-kasse.md` | Artikelmap, Bridge, REST-Webservice, Hotfolder, Bon-Druck, Tracking |
| `references/verifikation.md` | E2E-Abnahme + Prüftabelle |
| `references/template-haerten.md` | Vorlage mandantenfähig machen (Kunden-Schlüssel, RLS, Config) |
| `references/regeln-und-fallen.md` | Harte Regeln und teuer gelernte Fehler |

**Verwandte Skills:** `project-blueprint` (Aufbau + Doku), `frontend-design`
(Branding-Oberfläche), `web-design-guidelines` (Barrierefreiheit),
`supabase-postgres-best-practices` (Schema, RLS, Migrationen), `devops` (CI/CD, Docker),
`web-app-launch` (Go-Live: Domain, Live-Zahlung, erste Bestellung).
