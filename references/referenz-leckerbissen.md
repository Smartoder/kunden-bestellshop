# Referenzsystem „Leckerbissen" — das fertige Muster

Der Skill war bewusst neutral. **Ab jetzt hat er ein konkretes Muster:** die laufende
Bestell-Website **Leckerbissen** (Tarmstedt, Deutschland) mit Anbindung an das
Kassensystem **WinOrder**. Dieses Muster ist **einmal komplett gebaut und mit echtem
Geld bewiesen**. Für einen neuen Kunden wird es **nachgebaut, nicht neu erfunden**:

> **Gleiche Website, gleiche Funktionen, gleicher Aufbau — neue Marke, eigene
> Infrastruktur, eigene Kasse.**

Diese Datei ist die **konkrete Wahrheit** hinter `references/architektur.md`
(dort neutral beschrieben) und der Fahrplan für den nächsten Kunden.

---

## 1. Was das Muster ist

| | |
|---|---|
| Marke / Betrieb | **Leckerbissen**, Tarmstedt (Niedersachsen) |
| Website | `https://www.leckerbissen.online` · Bestellseite `/website/speisekarte` |
| Produkt | Eigener Online-Bestellshop (Lieferando-artig): Speisekarte, Warenkorb, Lieferung/Abholung, Online-Zahlung, Bon in der Kasse |
| Kasse | **WinOrder 2025** (Version 10.0.0.14) auf einem PC im Restaurant |
| Bondrucker | **STAR mPOP** (Bluetooth) |
| Repo | `Smartoder/leckerbissen-speisekarte` **(privat)** — vorher `HofZeitV12/…`, GitHub meldet beim Push „This repository moved", der alte Pfad leitet um |
| Status | 🟢 **LIVE** — echte Zahlungen, End-to-End bis zum Bon bewiesen (Order #32 → Rechnung #6457) |

**Bewiesen am 27.09.2026 mit echter Karte:** Website → Stripe LIVE → Datenbank
(`status=offen`, `stripe_mode=live`) → Bridge → Hotfolder → WinOrder → **Rechnung/Bon**.

> ⚠️ **Dieses Muster ist die Vorlage, nicht der Bauplan.** Konkrete Kennungen
> (Server-IP, SSH-Schlüssel, Supabase-Ref, Vercel-Projekt-ID, Keys) stehen **absichtlich
> nicht** hier — sie liegen in der Projekt-Doku und im Zugangsregister. Der Skill ist
> ein **öffentliches** Repo: **keine Zugangsdaten, keine Server-IP, keine DB-Refs.**

---

## 2. Verifizierter Live-Zustand des Musters

Alle Zeilen wurden am **03.10.2026** von einem Entwicklungsrechner aus **ausgeführt**
(nicht aus dem Gedächtnis notiert):

| Baustein | Befehl | Ergebnis |
|---|---|---|
| Website / Bestellseite | `GET /website/speisekarte` | **200** (19 KB) |
| Speisekarte | `GET /api/menu` | **200**, **51 Artikel** |
| Stammdaten | `GET /api/store` | **200** — genau 5 Felder: `adresse`, `telefon`, `email`, `oeffnungszeiten`, `zubereitungszeit_min` |
| Kassen-Export **ohne** Key | `GET /api/export/winorder?status=offen` | **401** (Auth greift) |
| Mailserver | `GET https://webhook.<domain>/health` | **200 `status: ok`** |
| Diagnose **ohne** Token | `GET /api/status` | **401** (Auth greift) |

> Die zwei 401er sind **kein** Fehler — sie sind der Beweis, dass Export und Diagnose
> geschützt sind. Ein Aufruf **ohne** Key, der **200** liefert, wäre der Fehler.

---

## 3. Der Funktionsumfang, den ein neuer Kunde **1:1** bekommt

Das ist der „gleiche Aufbau". Ein Fork startet mit **genau dieser Oberfläche**.

**Öffentliche Seiten**

| Route | Zweck |
|---|---|
| `/website/speisekarte` | **Hauptkanal** — Menü, Warenkorb, Lieferung/Abholung, Checkout |
| `/website` | Weiterleitung auf `/website/speisekarte` |
| `/website/impressum` · `/datenschutz` · `/agb` | Rechtstexte |
| `/website/galerie` · `/kontakt` · `/reservierung` | Info-Seiten |
| `/success` | Nach der Zahlung; nimmt `ausstehend → offen` nur bei **bezahlter** Stripe-Session |
| `/status/[orderId]` | Bestellstatus für den Kunden |
| `/tisch/[tischnummer]` | QR-Tischbestellung (**Rest aus dem Ursprungs-Template**, nicht der Betriebsweg) |
| `/offline` | PWA-Offline-Seite |

**API**

| Route | Auth | Zweck |
|---|---|---|
| `POST /api/checkout` | öffentlich | Order `ausstehend` + Stripe-Session; **Modus-Wache** (503 bei Konflikt) |
| `POST /api/webhooks/stripe` | Signatur + `livemode` | Zahlung → `offen`; **409** bei Modus-Konflikt |
| `GET /api/menu` · `GET /api/store` | öffentlich | Speisekarte · Stammdaten |
| `GET /api/plz-lookup` | öffentlich | PLZ → Ort (externer Call) |
| `GET /api/export/winorder` | `x-api-key` | Partner-Export (JSON/CSV); **`mode=live` ist Standard** |
| `PATCH /api/export/winorder/[id]` | `x-api-key` | Ack `in_bearbeitung`; **409** bei Testbestellung |
| `GET/POST /api/winorder/GetNewOrders` | Kassen-Auth | REST-Abholung (Kasse holt selbst) |
| `POST /api/winorder/SendTrackingStatus` | Kassen-Auth | Status 0–11 zurück |
| `GET /api/GetNewOrders` · `/api/SendTrackingStatus` | Kassen-Auth | **Aliase** für andere Pfad-Schreibweisen der Kasse |
| `/api/admin/*` | Basic Auth | Admin-UI (**Template-Rest**, nicht Betriebsweg) |
| `PATCH /api/cleanup-stuck-orders` | Partner-Auth | `ausstehend → offen` (Testbestellungen ausgenommen) |

> **Die Aliase nicht löschen.** WinOrder hängt den Pfad unterschiedlich an, je nachdem
> ob die Webservice-URL auf `…/api` oder `…/api/winorder` zeigt. Alle Varianten werden
> bedient, damit eine Fehlkonfiguration nicht zu einem stillen Ausfall führt.

**Funktionen, die neu dazugekommen sind** (Stand 03.10.2026 — die Liste oben ist die
**Form**, kein eingefrorener Katalog):

- **Deterministische Volltextsuche** über den Wissensspeicher (`npm run suche`,
  SQL-Funktion über `project_memory`) — **bewusst ohne** Vektoren/RAG (Eigennamen,
  Nummern, Kennungen). Ein Fork kann sie mitnehmen oder weglassen; sie berührt den
  Bestellweg **nicht**.
- Zusätzliche Admin-/Diagnose-Route `app/api/debug/route.ts` (hinter Basic Auth). Eine
  neue Route im Fork heißt: die **öffentliche** Middleware-Liste gegenprüfen — sonst
  wird etwas versehentlich öffentlich oder blockiert.

> **Vor dem Fork den aktuellen Stand ablesen** (siehe Abschnitt 8.1), nicht diese
> Tabelle als eingefroren behandeln.

**Funktionen**

- Menü aus der Datenbank, **Preise serverseitig neu gerechnet** (nie aus dem Browser)
- Lieferzonen als **Allow-List** (PLZ, Mindestbestellwert, Gebühr), Fehlertext im Modul
- Öffnungszeiten + **Wunschzeit** aus Freitext (15-Min-Raster, Vorlauf)
- Warenkorb mit Kundendaten, Liefer- oder Abholmodus
- Stripe **Redirect-Checkout** (`mode: payment`), `metadata` = `order_id` + `order_type`
- **Zwei Webhooks**, **Modus-Wache**, Mail-Marker `bestaetigung_mail_am`
- Kundenmails über den **Hetzner-Server** (Resend) — Vercel verschickt **keine** Mails
- **Küchenwächter** (Timer): meldet per Mail, wenn eine bezahlte Bestellung keinen Bon auslöst
- PWA/Manifest + Icons (Generator-Skript)
- Bridge (PowerShell) + REST-Weg ans Kassensystem

---

## 4. Der Werkzeugkasten in Zahlen (Muster)

| Baustein | Stand im Muster | Für einen neuen Kunden |
|---|---|---|
| **GitHub** | 1 privates Repo/Kunde | **1 eigenes privates Repo** (frischer Start, ohne Muster-History) |
| **Vercel** | Team **Hobby** (kostenlos), Git-Integration | eigenes Projekt im selben oder eigenen Team |
| **Supabase** | Projekt „Üben", **geteilte** Instanz, **kein** RLS | **eigene Instanz** (Standardweg) — die geteilte Muster-DB **nie** nachbauen |
| **Hetzner** | 1 KVM-Cloud-Server (Abo ~19 €/Monat), Docker + Caddy | eigener Server **oder** getrennte Container/Ports/`.env` auf demselben Server |
| **Stripe** | Live, 2 Endpunkte, **eingeschränkter** Schlüssel `rk_live_…` | eigenes Konto; Test zuerst, dann live (Reihenfolge siehe `references/infrastruktur.md`) |
| **Resend** | Free-Tier, **1 verifizierte Domain**, Limit **3 Domains** | **bald höherer Tarif** — erst dann > 3 Kunden auf einem Resend-Konto |
| **WinOrder** | WinOrder 2025, 1 Kasse, Hotfolder-Bridge | eigene Kasse, **eigener** Hotfolder-/Artikelstamm |
| **Kasse: Weg** | **Hotfolder-Bridge** funktioniert; REST-Weg **tot** (Kasse fragt nicht ab) | Hotfolder als Standard; REST nur, wenn die Kasse es nachweislich kann |

> ⚠️ **Resend ist die aktuelle Mengengrenze.** Der Free-Tier erlaubt **3 Domains**
> (verifiziert am 03.10.2026: `Domains: 1 / 3`, `Emails: 16 / 3000 monatlich`).
> Ab dem vierten Kunden braucht jeder **eigene** Absender-Domain einen höheren Tarif.

---

## 5. Die Kassen-Anbindung des Musters (konkret)

Der Weg, der **funktioniert** — und der Weg, der **nicht** funktioniert:

| Weg | Stand im Muster |
|---|---|
| **Hotfolder-Bridge** (PowerShell pollt den Export, legt JSON in den überwachten Ordner) | ✅ **läuft**, Autostart ohne Adminrechte (HKCU-Run + Aufpasser) |
| **REST-Webservice** (`ProviderType = esptREST`) | ❌ **tot** — WinOrder fragt SmartOrder nicht ab. Zwei unabhängige Tests negativ; Ursache offen (Support-Ticket) |

**Der Hotfolder** (Pfad am Kassensystem ablesen, **nicht** raten):

```
<WINORDER-BASIS>\EShop\Incoming     ← hier ablegen
<WINORDER-BASIS>\EShop\Processed    ← Kasse verschiebt hierhin (geparst)
```

> ⚠️ **Im Muster trägt der Ordner eine Versionsnummer** (`…\WinOrder10\…`), nicht nur
> `WinOrder`. Ein „richtig aussehender" Pfad ohne die Nummer schreibt ins Leere. Der
> Basispfad wird am **Kassensystem selbst** abgelesen.

**Feste Formattexte des Musters** (beim Klonen anpassen):

| Feld | Muster-Wert |
|---|---|
| `OrderID`-Präfix | `LB-<id>` (z. B. `LB-32`) |
| `PaymentType` | `Online bezahlt` (muss als **Zahlungsart** in der Kasse existieren) |
| Artikel-Präfix | Pizza trägt `Pizza `, Dips/Beilagen/Getränke **ohne** Präfix |
| Artikel-Zuordnung | **zwei** Dateien, identischer Inhalt: `lib/winorder/articles.ts` + `tools/winorder-articles.json` |

Details: `references/winorder-kasse.md`.

---

## 6. Was **kundenspezifisch** ist — niemals mitkopieren

| Bleibt **einmalig** beim Muster | Beim neuen Kunden **neu** |
|---|---|
| Server-IP, SSH-Schlüssel, Hostname | eigener Zugang |
| Supabase-Ref und dessen Tokens | eigene Instanz |
| Vercel-Projekt-ID, Domains, DNS | eigene Domain |
| Stripe-Konto, Keys, Webhook-Secrets, Endpunkt-IDs | eigenes Konto/Endpunkte |
| Resend-Key, Absender, `Reply-To` | eigener Absender |
| `EXPORT_API_KEY`, `ADMIN_PASSWORD`, `ADMIN_TOKEN` | eigene Betriebsschlüssel |
| Hotfolder-Pfad, `WINORDER_USER`, Artikelstamm | eigene Kasse |
| Marke, Farben, Logo, Fotos, Stammdaten, Lieferzonen, Rechtstexte | eigene Marke |

**Faustregel:** Teilen zwei Kunden einen Schlüssel, eine Tabelle, ein Konto oder einen
Endpunkt, ist die Trennung nicht vollständig.

---

## 7. Klon-Fahrplan — vom Muster zum neuen Kunden

Der neue Kunde bekommt **dieselbe Website mit denselben Funktionen**, nur mit eigener
Marke und eigener Umgebung. Reihenfolge (Details in der jeweiligen Referenz):

| # | Schritt | Quelle |
|---|---|---|
| 0 | **Muster live prüfen** (Tabelle aus Abschnitt 2) — beweist, dass der Weg läuft | diese Datei |
| 1 | **Intake** — zwölf Fragen beantworten, nichts erfinden | `references/intake.md` |
| 2 | **Repo** — privater Fork/Kopie **ohne** Muster-History; Git-Autor = Hosting-Mitglied | `references/infrastruktur.md` |
| 3 | **Datenbank** — eigene Supabase-Instanz, Migrationen in Reihenfolge, Menü-Seed, `arch.store_config` | `references/infrastruktur.md` |
| 4 | **Entbranden** — Farben (inkl. Alias-Klassen), Logo, Stammdaten (Route **und** `store_config`), Lieferzonen, Texte, Mail, PWA | `references/entbranden.md` |
| 5 | **Hosting** — neues Vercel-Projekt, Env vollständig, Domain, Redeploy | `references/infrastruktur.md` |
| 6 | **Server** — Container/`.env`/Domain je Kunde, Caddy-Pfade, Küchenwächter | `references/infrastruktur.md` |
| 7 | **Zahlung** — eigenes Stripe-Konto, zwei Endpunkte, Modus-Wache, Test zuerst | `references/infrastruktur.md` |
| 8 | **E-Mail** — Absender + **verifizierte** Domain beim Mailanbieter | `references/entbranden.md` |
| 9 | **Kasse** — Hotfolder, Artikelmap aus **echten** Bonzeilen, Zahlungsart, Auto-Druck | `references/winorder-kasse.md` |
| 10 | **Verifizieren** — E2E bis zum Bon, Zugriffstest (Anon muss schreiben **scheitern**) | `references/verifikation.md` |
| 11 | **Dokumentieren** — START, PROJEKT, Entscheidungen, Runbook | Skill `project-blueprint` |

**Fertig ist der neue Kunde erst**, wenn Abschnitt 9 „Erfolgsdefinition" aus
`references/winorder-kasse.md` mit einer **echten** Bestellung bewiesen ist.

---

## 8. Grenzen der Automatisierung (Ist-Zustand der Werkzeuge)

Ehrlich festgehalten, damit kein Agent „alles automatisch" verspricht:

Nachgeprüft am **05.10.2026** (die Lage ändert sich — vor jeder Aussage neu prüfen).
Die Tabelle vom **03.10.2026** ist darunter als Vergleich erhalten:

| Werkzeug | Stand am 03.10.2026 | Grenze / Ersatzweg |
|---|---|---|
| **GitHub MCP** | ✅ autorisiert als Projekt-Inhaber (7 private Repos) | Repo-Zugriff hängt am Konto |
| **Resend MCP** | ✅ autorisiert | zeigt Domains + Usage, **nicht** das Tagesgeschäft der Mails |
| **Vercel MCP** | ✅ **antwortet wieder** (5 Projekte, Deploys `READY`) | war am Vormittag noch `403` („re-authenticate"). Ein Token/Scope kann **jederzeit** wieder ablaufen — erneut prüfen, nicht annehmen |
| **Supabase MCP** | ❌ Zielprojekt **nicht freigegeben** (`no permission` bei Tabellen/Advisors, Projektliste leer) | Muster nur über die **HTTP-API** auslesbar; für neue Kunden die eigene Instanz als MCP verbinden |
| **Hetzner MCP** | ❌ **nicht erreichbar** — Diagnose-Port `11436` von diesem Rechner **geschlossen** (443/22 offen); `mcp_auth` läuft in einen Timeout | Der Port hängt an einer **IP-Freigabeliste** der Hetzner-Firewall. Weg: **SSH-Tunnel** über Port 22 (bleibt ohne Firewall-Änderung nutzbar), sonst Firewall-Regel für die eigene IP |

**Nachtrag 05.10.2026:**

| Werkzeug | Stand | Bemerkung |
|---|---|---|
| **Vercel-MCP (Plugin, OAuth)** | ❌ `403` — „re-authenticate to this scope" bei **Team**-Ressourcen | Nutzer-Ebene erreichbar, Team-Scope nicht |
| **Vercel-MCP (Token-Variante)** | ✅ `HTTP 200` auf `POST https://mcp.vercel.com` mit Bearer-Token | **Ersatzweg:** eigenen Eintrag in `~/.cursor/mcp.json` (`type: http`, `url`, `headers.Authorization = Bearer vcp_…`). Kein OAuth nötig |
| **Hetzner-MCP** | ✅ erreichbar; Server `running` | zusätzlich per **SSH** geprüft (Port 22 offen, Port 11436 an IP-Freigabeliste) |
| **Supabase-MCP** | ⚠️ Liste zeigt **fremde** Projekte mit | Nur die **eigene** Instanz anfassen — nie ein fremdes Projekt im selben Konto |

> **Merksatz:** Ein MCP grün zu *nennen* ist kein Nachweis. Jede Aussage braucht den
> **ausgeführten** Befehl — und wenn ein MCP fehlt, wird ersatzweise per **HTTP/SSH/
> Dashboard** geprüft (siehe `SKILL.md` → Diagnose).

### 8.2 Ein `BLOCKED`-Deploy ist kein Code-Fehler

Am **05.10.2026** blieb ein Produktions-Deploy `BLOCKED` — **ohne Build (0 ms)**. Die
Seite zeigte weiter den alten Stand. Der Grund stand **nicht** im Build-Log:

```
readyStateReason: "The deployment was blocked because Vercel couldn't find a
                   Git account for the commit author."
seatBlock.blockCode: COMMIT_AUTHOR_REQUIRED
```

Ursache: Der Commit-Autor trug eine E-Mail, die zu **keinem** GitHub-/Vercel-Konto
gehört. Fünf vorherige Commits **desselben Autors** waren noch durchgelaufen — die
Prüfung greift also **ab einem Zeitpunkt**, nicht rückwirkend. Ein grüner Verlauf ist
**kein** Beweis.

**Regel:** Bei `BLOCKED` **zuerst** den Grund aus der API lesen, **nicht** den Code
durchsuchen:

```bash
vercel inspect --json <deploy-url>          # readyState, readyStateReason
# GET /v13/deployments/<id>?teamId=<team>   # readyStateReason, seatBlock.blockCode
```

Details und der Fix (Commit neu autorisieren, Baum bleibt unverändert):
`references/regeln-und-fallen.md`, Falle 11.

### 8.1 Die Vorlage ist **nicht eingefroren** — immer vom aktuellen Stand klonen

Die Vorlage (das private Repo des Musters) wird **weiterentwickelt**. Ein Fork, der von
einem **alten lokalen Arbeitsstand** abzweigt, erbt veraltete Skripte und ein altes
Schema. Zwei belegte Beispiele vom **03.10.2026**:

- `package.json` der Vorlage hat heute **`check`, `types`, `suche`** — ein älterer
  lokaler Stand hatte nur `dev/build/start/lint/gen-icons`. Der Prüfbefehl hängt also
  vom **Stand**, nicht nur vom Projekt ab.
- `supabase/migrations/` hat heute einen **älteren + einen neueren Satz** (u. a. eine
  Migration für Volltextsuche über `project_memory`). Wer von einem alten Stand klont,
  verliert sie.

**Regel:**

```bash
git fetch origin
git rev-list --count HEAD..origin/main     # 0 = aktuell; >0 = veraltet
```

Vor dem Fork **auf `origin/main` aktualisieren** (oder direkt von `origin/main` klonen)
und anschließend `package.json` **und** `supabase/migrations/` am aktuellen Stand
ablesen — nie aus dem Gedächtnis.

---

## 9. Was am Muster **nicht** vorbildlich ist

Das Muster beweist, dass der Weg funktioniert — es ist **kein** Sicherheitsvorbild.
Beim neuen Kunden **anders** machen:

- **Geteilte Datenbank ohne RLS.** Die Muster-DB teilt sich mit fremden Projekten;
  `orders`, `menultems`, `project_memory` sind per Anon-Key les- **und** änderbar.
  → Neuer Kunde: **eigene Instanz**, RLS **mit** Policies (`references/template-haerten.md`).
- **Zugangsdaten in getrackten Dateien.** Im Muster stand ein Live-Key in einer
  git-getrackten Datei; nur Rotation in der Quelle hilft. → Neuer Kunde: **kein** Wert
  in getrackten Dateien, `references/regeln-und-fallen.md` Falle 6/7.
- **Zwei Zeitstempel** (`erstellt_am` + `created_at`) und historische Namen
  (`menultems` mit Tippfehler). → Übernehmen, **nicht** „aufräumen".

---

## 10. Referenz-Dokumente des Musters (im privaten Projekt)

Für die tiefe Diagnose am Muster (nur mit Zugriff auf das Privat-Repo):

| Thema | Datei im Muster-Repo |
|---|---|
| Gesamtdoku Kasse ↔ Website | `docs/KASSE-WINORDER-GESAMTDOKU.md` |
| Server + Zugänge (MCP, IP, Firewall) | `docs/SERVER-WISSENSSPEICHER.md` |
| Betriebsmodus Test/Live | Skill `stripe-leckerbissen` |
| Website-Gesamtzustand | Skill `leckerbissen-website` |
| Kassen-Anbindung | Skill `winorder-leckerbissen` |

> Diese Pfade gehören **nicht** in ein Kundenprojekt. Sie sind hier nur als Fundort
> für die Arbeit **am Muster** genannt.
