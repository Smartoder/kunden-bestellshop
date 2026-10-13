# Architektur der Vorlage — der Soll-Zustand, den ein Fork übernimmt

Diese Datei beschreibt **konkret**, wie das Referenzsystem aufgebaut ist und **welche
Werte der Fork übernehmen muss**. Sie ist die gemeinsame Wahrheit für alle anderen
Dateien dieses Skills. Alle Angaben wurden am **02.10.2026** am laufenden System
verifiziert (Live-HTTP + MCP-Abfragen), nicht aus dem Gedächtnis notiert.

> **Neutralität:** Tabellen-, Spalten- und Feldnamen sind Teil der Schnittstellen und
> werden deshalb genannt. Marken-, Orts- und Kundenangaben sind **Platzhalter**
> (`<KUNDE>`, `<DOMAIN>`, `<PLZ>`), weil sie pro Kunde neu gesetzt werden.

---

## 1. Der Weg einer Bestellung (Ende zu Ende)

```
Kunde (Browser)
   │  Bestellung auf  https://www.<kunde>.de/website/speisekarte
   ▼
Vercel (Next.js App Router)
   │  POST /api/checkout
   │   1. Menü serverseitig lesen (menultems) und Betrag NEU rechnen
   │   2. Lieferzone prüfen (Mindestbestellwert, Gebühr)
   │   3. Zeile in orders anlegen:  status = 'ausstehend'  (NOCH NICHT bezahlt)
   │   4. Stripe-Checkout-Session erstellen (metadata: order_id, order_type)
   │   5. stripe_session_id speichern; stripe_mode aus Session-Präfix ableiten
   ▼
Stripe (Checkout, redirect)
   │  Kunde zahlt mit Karte
   ├──────────────► Webhook #1 auf  www.<kunde>.de/api/webhooks/stripe
   │                → Status 'offen', bezahlt_am, Kundendaten aus der Session
   └──────────────► Webhook #2 auf  webhook.<kunde>.de/api/webhooks/stripe
                    → Status 'offen', bezahlt_am, stripe_mode
                    → Bestätigungsmail (Resend)
                    → Marker bestaetigung_mail_am  (NUR hier gesetzt)
   ▼
Supabase  (orders: status='offen', bezahlt_am gesetzt)
   │
   ├─ Erfolgsseite  /success?order_id=…  liest den Status bei Stripe gegen
   │  und setzt 'offen' als Fallback (schneller als der Webhook) — aber nur,
   │  wenn Stripe den Betrag als bezahlt meldet.
   ▼
Kassen-PC im Restaurant
   │  Hotfolder-Bridge  ODER  Kassen-REST-Webservice
   │   holt offene Bestellungen über  GET /api/export/winorder   (eigener Key)
   │   schreibt Kassen-JSON in den Hotfolder  ODER  liefert es per REST
   │   quittiert:  PATCH /api/export/winorder/<id>  → status='in_bearbeitung'
   ▼
Kassen-Software
   │  parst, ordnet dem Shop zu, druckt den Bon auf dem Thermodrucker
   │  meldet Fortschritt an  POST /api/winorder/SendTrackingStatus
   ▼
Bon auf dem Thermodrucker (mPOP o. ä.)
```

**Der Kernbeweis:** Kein Schritt verlangt manuelles Kopieren.

---

## 2. Statusmodell der Bestellung

Reihenfolge: `ausstehend → offen → in_bearbeitung → bereit → abgeholt`
(plus `storniert`).

| Status | Wer setzt ihn | Bedeutung |
|---|---|---|
| `ausstehend` | `/api/checkout` | Bestellung angelegt, **noch nicht bezahlt** |
| `offen` | Webhook **und** `/success` | bezahlt — **erst jetzt** darf die Küche dran |
| `in_bearbeitung` | Bridge/Export-Ack | von der Kasse übernommen |
| `bereit` | Kassen-Tracking | in Zubereitung / unterwegs |
| `abgeholt` | Kassen-Tracking | abgeschlossen |
| `storniert` | Kassen-Tracking | abgelehnt / rückerstattet / storniert |

> 🔴 **`ausstehend` ist NICHT bezahlt.** Würde die Küche diesen Status abholen, könnte
> ein abgebrochener Checkout einen Bon für eine **unbezahlte** Bestellung drucken. Nur
> der Zahlungsweg setzt `offen`.
>
> 🔴 **`in_bearbeitung` darf im Abholfilter NICHT mitgelesen werden.** Es ist der
> Status, den die Kasse selbst setzt; stünde er im Filter, würde **jede** übernommene
> Bestellung bei **jedem** Lauf erneut gedruckt.

---

## 3. Datenbank (Supabase)

**Instanz:** eigene Instanz pro Kunde (Standardweg). Region passend zum Kunden
(EU, wenn EU-Kunde). Die Vorlage läuft in einer EU-Region.

### Tabelle `menultems` (Speisekarte)

> ⚠️ Der Name ist **historisch** und enthält einen Tippfehler (`menultems` statt
> `menuitems`). **Nicht „korrigieren"** — der Name steht in Code, Migrationen und
> Abfragen. Wer ihn ändert, bricht alles.

Der Checkout liest `id, name, preis` und filtert `.eq('verfuegbar', true)`. Die
**Preise kommen niemals aus dem Browser** — sie werden serverseitig aus dieser Tabelle
gelesen und der Betrag neu gerechnet.

### Tabelle `orders` (Bestellungen)

Spalten, die der Bestellweg nutzt (verifiziert):

| Spalte | Zweck |
|---|---|
| `id` | Bestellnummer (auch die `OrderID` an die Kasse: `LB-<id>`) |
| `restaurant` | **Kundenkennung** — jede Abfrage filtert darauf |
| `order_type` | `tisch` \| `web_lieferung` \| `web_abholung` |
| `status` | siehe Statusmodell |
| `stripe_mode` | `test` \| `live` (bzw. NULL bei Altbeständen) |
| `stripe_session_id` | verlässlichste Zuordnung Test/Live (`cs_test_` / `cs_live_`) |
| `positionen` | JSONB-Liste der Positionen |
| `gesamtbetrag` | Gesamt (inkl. Liefergebühr) |
| `erstellt_am`, `bezahlt_am`, `created_at` | Zeitstempel |
| `tisch_nummer`, `kunden_name`, `kunden_email`, `kunden_telefon` | Bestelldaten |
| `liefer_adresse`, `liefer_plz`, `liefer_stadt`, `liefer_anmerkung` | Lieferdaten |
| `abhol_zeit`, `liefer_zeit` | Wunschzeiten |
| `bestaetigung_mail_am` | **Mail-Marker** — nur der Mailserver schreibt ihn |
| `bestaetigung_mail_status` | `pending` \| `sent` \| `failed` \| `skipped` |
| `bestaetigung_mail_fehler` | letzter Fehlertext |

### Tabelle `project_memory` (Wissensspeicher + Laufzeit-Config)

Key-Value-Speicher, Konvention `key = <bereich>.<thema>`. Seit dem **29.09.2026** hat
die Vorlage zusätzlich eine **generierte Volltext-Spalte** (`suchvektor`, Sprache
`german`) samt `GIN`-Index und einer SQL-Suchfunktion — **deterministisch, ohne
Vektoren**. Ein Fork muss das **nicht** übernehmen (der Bestellweg braucht es nicht),
der aktuelle Stand hat es aber.

> ⚠️ **Ausdrücklich NICHT bestätigt:** eine separate Tabelle `project_memory_history`.
> Eine Historientabelle steht **nirgends** im Vorlage-Code (Suche am 03.10.2026: kein
> Treffer). Wenn ein Fork eine Versionshistorie will, ist das eine **neue** Anforderung
> — nicht als vorhandene Vorlage-Eigenschaft annehmen.

> ⚠️ **Exaktes DDL.** Migrationen und Prüfcode müssen die echten Spalten treffen:
> `project_memory` hat **`key` + `content`** (nicht `value`); `menultems` hat
> **`verfuegbar`** (nicht `aktiv`). Wer diese Namen errät, bricht den Bestellweg.
> Namen **am lebenden Schema ablesen**, nicht aus dem Gedächtnis schreiben.

- **`arch.store_config`** ist die **Laufzeit-Wahrheit** der Stammdaten. Die Store-Route
  der Website liest diesen Key und liefert seinen Inhalt **wörtlich** aus.
- Verifizierte Felder: `adresse`, `telefon`, `email`, `oeffnungszeiten`,
  `zubereitungszeit_min` — **genau diese fünf**.
- ⚠️ **Jedes zusätzliche Feld wird öffentlich sichtbar.** Feldnamen nie umbenennen,
  Werte nur bewusst ändern.

### Sicherheit

Supabase meldet für die Vorlage **RLS ist aus** auf sechs Tabellen als **kritisch**.
Auf der geteilten Instanz trifft das auch fremde Projekte. Für einen Kunden gilt:
eigene Instanz, RLS **mit Policies** (nicht einfach einschalten — siehe
`references/template-haerten.md` und `references/regeln-und-fallen.md`, Falle 7).

---

## 4. Hosting (Vercel)

- **Next.js App Router**, Team-Konto, **Git-Integration = einziger Deployweg**.
- Öffentliche Routen (durch die Middleware freigegeben):
  `/api/menu`, `/api/store`, `/api/checkout`, `/api/webhooks/*`, `/api/auth/*`,
  `/api/plz-lookup`, `/api/export/*`.
- Hinter Basic Auth: `/admin` und `/api/admin/*` (Realm-Text trägt den Kundennamen).
- Die Erfolgsseite `/success` liest den Zahlungsstatus **aktiv bei Stripe** gegen und
  schaltet nur bei `payment_status === 'paid'` auf `offen` — nie allein wegen der
  Browser-Rückkehr.

---

## 5. Zahlung (Stripe)

- **Redirect-Checkout** (`mode: 'payment'`), `metadata` trägt `order_id` und
  `order_type` — das ist die Zuordnung im Webhook.
- **Zwei Webhook-Endpunkte** (siehe oben). Nur der Server-Host verschickt Mails.
- **Modus-Wache** in beiden Empfängern: Soll (`STRIPE_EXPECTED_MODE`) muss zum Event
  (`livemode`) passen, sonst **409** ohne DB-Schreibung.
- **Race Condition:** `/success` setzt `bezahlt_am` sofort; der Mailversand hängt
  deshalb ausschließlich an `bestaetigung_mail_am`, das nur der Mailserver schreibt.

---

## 6. Server (Hetzner)

- **KVM-VPS**, Docker Compose hinter Caddy. Die Vorlage nutzt ein kleines Cloud-Abo
  (ein Server trägt Webhook + Mail + Wächter für **einen** Kunden); für weitere Kunden
  entweder ein eigener Server oder **getrennte Container/Ports/`.env`** auf demselben.
  Die tatsächliche Größe wird am Server abgelesen (`nproc`, `free -h`), nicht angenommen.
- Gehört zum Webhook/Container-Betrieb: **Zahlungs-Webhook**, **Bestätigungsmail**,
  **Küchenwächter** (meldet hängende bezahlte Bestellungen).
- **Caddy** ist die einzige öffentliche Fläche: vier Pfade, Rest 404.
- **Firewall:** nur die nötigen Ports (22/80/443); Diagnose-Ports möglichst auf feste
  IPs beschränken.
- **Küchenwächter** läuft als Timer und alarmiert per Mail an `ALERT_EMAIL`, wenn eine
  bezahlte Bestellung nicht an die Küche kommt.

---

## 7. Kasse (Kassen-PC im Restaurant)

Zwei Betriebswege, beide aus derselben Quelle gespeist (`orders`, Status `offen`):

| Weg | Wie | Wann |
|---|---|---|
| **Hotfolder-Bridge** | Ein PowerShell-Prozess pollt den Export und legt je Bestellung eine Datei in einen überwachten Ordner | Standard der Vorlage |
| **REST-Webservice** | Die Kasse ruft selbst `…/api/winorder/GetNewOrders` ab und meldet Status an `…/SendTrackingStatus` | wenn die Kasse es kann |

Beide Wege erzeugen **dasselbe Bestellbild**. Details:
`references/winorder-kasse.md`.

---

## 8. Datei- und Modulverzeichnis der Vorlage (Entbranding-Anker)

Diese Pfade sind die Stellen, die ein Fork anfassen muss. Namen können leicht
abweichen — über die **Suche** nach dem alten Kundennamen prüfen, nicht blind ersetzen.

| Zweck | Pfad |
|---|---|
| Checkout (Bestellung, Betrag, Stripe-Session) | `app/api/checkout/route.ts` |
| Stripe-Webhook (Hosting) | `app/api/webhooks/stripe/route.ts` |
| Stammdaten-Route | `app/api/store/route.ts` |
| Speisekarten-Route | `app/api/menu/route.ts` |
| Partner-Export (Kasse holt) | `app/api/export/winorder/route.ts` |
| Partner-Ack (`PATCH` Status) | `app/api/export/winorder/[id]/route.ts` |
| Kassen-REST (abholen) | `app/api/winorder/GetNewOrders/route.ts` |
| Kassen-REST (Status zurück) | `app/api/winorder/SendTrackingStatus/route.ts` |
| Erfolgsseite | `app/success/page.tsx` |
| Modus-Wache | `lib/stripe-mode.ts` |
| Lieferzonen | `lib/delivery-zones.ts` |
| Öffnungszeiten / Wunschzeit | `lib/opening-hours.ts` (`prepMinutes`, `computeSlots`) — Mo–Fr vs. Sa–So siehe `references/referenz-leckerbissen.md` §3 |
| Kassen-Format | `lib/winorder/format.ts` |
| Kassen-Artikelmap (TS) | `lib/winorder/articles.ts` |
| Kassen-Artikelmap (JSON) | `tools/…-articles.json` |
| Status-Zuordnung Kasse→App | `lib/winorder/tracking.ts` |
| Kunden-Statusmails | `lib/winorder/notify.ts` |
| Partner-/Kassen-Auth | `lib/partner-auth.ts`, `lib/winorder/auth.ts` |
| Admin-Auth / Realm | `middleware.ts`, `lib/auth.ts` |
| Bridge (Hotfolder) | `tools/…bridge….ps1` |
| Webhook-Server (Hetzner) | `deploy/webhook-server/server.mjs` |
| Mail-Vorlage | `deploy/webhook-server/email-templates.mjs` |
| Reverse-Proxy | `deploy/webhook-server/Caddyfile` |

---

## 9. Umgebungsvariablen (Katalog)

**Namen** sind stabil, **Werte** gehören nie ins Repo. Vollständige Zuordnung
(Hosting vs. Server) in `references/infrastruktur.md`, Abschnitt 6.

| Variable | Baustein | Zweck |
|---|---|---|
| `SITE_URL` / `NEXT_PUBLIC_SITE_URL` | Vercel | Kunden-Domain (auch Checkout-Rückkehr) |
| `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Vercel (Client) | öffentlicher Zugang |
| `SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` (auch `SUPABASE_SERVICE_KEY`) | Vercel + Server | Service-Role, **nur serverseitig** |
| `STRIPE_SECRET_KEY` | Vercel + Server | Zahlung |
| `STRIPE_WEBHOOK_SECRET` (`_TEST` optional) | Server + Webhook | Signaturprüfung |
| `STRIPE_EXPECTED_MODE` | Vercel + Server | Soll-Modus `test` \| `live` |
| `EXPORT_API_KEY` | Vercel + Kassen-PC | Kassen-Auth (`x-api-key`) |
| `ADMIN_USER`, `ADMIN_PASSWORD` | Vercel + Kassen-PC | Admin/Partner Basic Auth |
| `ADMIN_TOKEN` | Server | Diagnose + Mail-Retry |
| `RESEND_API_KEY`, `RESEND_FROM`, `RESEND_REPLY_TO` | Server | Mailversand + Absender |
| `ALERT_EMAIL` | Server | technischer Alarm (nicht Kunde) |
| `WINORDER_USER` | Kassen-PC | Kassenbenutzer |
| `GITHUB_WEBHOOK_SECRET`, `GITHUB_TOKEN` | Vercel | optionaler GitHub-Webhook |
