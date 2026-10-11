# Verifikation — den neuen Kunden abnehmen

**Kein „fertig" ohne ausgeführten Befehl.** Jede Zeile dieser Tabelle wird **wirklich**
ausgeführt. War eine Prüfung nicht möglich, steht das dort — mit Grund.

---

## Vor der Fehlersuche

Prüfe zuerst, **wie viele Entwicklungsprozesse** laufen. Mehrere auf demselben
Bauverzeichnis überschreiben sich gegenseitig und erzeugen Fehler ohne Bezug zur
Ursache.

```powershell
Get-Process node -ErrorAction SilentlyContinue |
  ForEach-Object { (Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine }
```

Nur **einen** Dev-Server laufen lassen. `build` **nie** neben laufendem `dev` —
sonst alle stoppen, Bauverzeichnis löschen, neu bauen.

---

## Prüftabelle

| Bereich | Prüfung | Erwartet |
|---|---|---|
| Code | Projekt-Prüfbefehl (**erst `package.json` lesen** — Name ist pro Projekt verschieden) | **grün** |
| Geheimnisse | Repo nach Schlüssel-Präfixen durchsuchen | **kein Treffer** |
| Geheimnisse | Ignorierliste greift (Testdatei anlegen) | wird **nicht** angezeigt |
| Schema | doppelte Migrationskennungen | **leere** Ausgabe |
| Schema | Migration ein zweites Mal anwenden | **kein** Fehler |
| Zugriff | mit öffentlicher Rolle **schreiben** | **schlägt fehl** |
| Zugriff | sensible Tabelle ohne Serverschlüssel lesen | **schlägt fehl** |
| Trennung | Kunde A sieht **kein** Menü/keine Config von Kunde B | getrennt |
| Branding | Abschluss-Suche nach Altmarke | kein Treffer in Code/Config/`public` |
| Bestellweg | Bestellstatus-Modell | `ausstehend` wird **nicht** in die Küche gelassen |
| Website | Startseite + Speisekarte | HTTP 200 |
| Menü | Menü-Endpunkt | Artikelzahl des Kunden |
| Stammdaten | Stammdaten-Endpunkt | **Kunden**daten, keine Altwerte |
| Kasse | Export **ohne** Key | HTTP 401 |
| Kasse | Abhol-Endpunkt (REST) | liefert nur `offen`/`ausstehend` |
| Kasse | **Extras** erscheinen als `SubArticleList.SubArticle` (nicht als eigene Artikel) | je Extra ein `SubArticle` |
| Kasse | **Bemerkung** als per-Artikel-`Comment` (nicht Kopf-`Comment`) | „ohne Mais" am Gericht |
| Kasse | **Extra-Namenskollision:** „Salami" bleibt „Salami" | **nicht** „Pizza Salami" |
| Kasse | Extra nur bei Pizza/Calzone; unbekannte Extra-ID | **400** |
| Kasse | Export **mit** Key, Standard | leer im Testbetrieb (gewollt) |
| Zahlung | Checkout mit leerem Warenkorb | HTTP 400 |
| Zahlung | Modus-Wache passt zu den Keys | kein 503/409 |
| Mail | Mailserver-Health | `ok` |
| Mail | Testbestellung: `bestaetigung_mail_am` gesetzt | ja |
| E2E | Eine Bestellung bis zum **Bon** | Bon gedruckt |
| Doku | jeder Link in `docs/START.md` | alle vorhanden |
| Doku | fremde Person startet anhand der Dateien | gelingt ohne Rückfragen |

---

## Die E2E-Kette (jede Stufe kann unabhängig ausfallen)

```
1. Session im richtigen Modus?     Session-ID-Präfix test/live
2. Bestellung bezahlt?             status=offen UND bezahlt_am gesetzt
3. Kundendaten da?                 kunden_name/-email gefüllt (kommt NUR vom Webhook)
4. Mail versandt?                  bestaetigung_mail_am gesetzt, _status='sent'
5. Küche draußen?                  Export: Test leer / Live enthält die Bestellung
6. Bon gedruckt?                   Kassen-Log / Processed-Datei / Druckauftrag
```

```sql
select id, status, bezahlt_am, kunden_email,
       bestaetigung_mail_am, bestaetigung_mail_status
from public.orders order by id desc limit 3;
```

> **Häufigster Ausfall auf dieser Strecke:** Punkt 4 (Mail) oder Punkt 5 (Modus).
> Erst den Modus prüfen, dann die Mail-Secrets.
>
> ⚠️ **Race Condition Mail vs. Erfolgsseite:** Die Erfolgsseite nach der Zahlung setzt
> `bezahlt_am` sofort — schneller als der Webhook. Wer die Mail an `bezahlt_am`
> hängt, überspringt den Versand. Der Mail-Marker (`bestaetigung_mail_am`) darf
> **nur** vom Mailserver geschrieben werden.

---

## Zugriff prüfen (der wichtigste Sicherheitstest)

Der Test, der zählt: **mit der öffentlichen Rolle schreiben muss fehlschlagen.**
Ein **gelungener** Aufruf ist ein **gefundener Fehler**.

- [ ] Anon-Key + Schreibversuch auf `orders` → **403/Fehler**
- [ ] Anon-Key + Ändern der **Modus-Spalte** (`stripe_mode` o. ä.) → **Fehler**
- [ ] Anon-Key + Lesen von `menultems` → erlaubt (öffentlicher Shop) **oder** bewusst
      geschützt
- [ ] Service-Role-Key **nur** serverseitig — nie im Client-Bundle

---

## Branding prüfen

```powershell
rg -i "ALTNAME|ALTER-ORT|ALTE-DOMAIN|alte-farbklasse|ALTER_ORDER_PREFIX" `
  --glob "!node_modules" --glob "!.next" --glob "!.git"
```

- [ ] Kein Treffer in Code, Konfiguration oder `public/`
- [ ] Treffer **nur** in `docs/` als bewusste Historie
- [ ] Logo, Hero, Icons zeigen **Kunden**marke
- [ ] Impressum/Datenschutz/AGB nennen den **richtigen** Anbieter

---

## Trennung prüfen

- [ ] Jeder Kunde: eigene DB, eigenes Hosting, eigene Domain
- [ ] Kein Schlüssel, Endpunkt, Hotfolder zwischen zwei Kunden geteilt
- [ ] Falls gemeinsamer Server: getrennte Container, Ports, `.env`, Mail-Absender

---

## Abnahmekriterium (vom Kunden bestätigt)

Aus `references/intake.md`, Frage 12: **„Fertig, wenn …"**.

- [ ] Diesen Satz wörtlich erfüllt
- [ ] Mit **echter** Bestellung bewiesen (nicht simuliert)
- [ ] Beweis festgehalten: Nummer, Zeit, Rechnung, Bon

---

## Abschlussmeldung je Phase

```markdown
## Phase N – <Name>

**Angelegt:** … (mit echten Kennungen)
**Geprüft:** `<Befehl>` → <Auszug der echten Ausgabe>
**Offen:** … und warum.
**Dokumentation:** welche Dateien nachgezogen wurden.
```
