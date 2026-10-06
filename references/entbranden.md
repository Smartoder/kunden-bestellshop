# Entbranden — die Vorlage auf den neuen Kunden umstellen

Marke und Kunde stecken **verteilt** im Code. Diese Datei ist die **vollständige
Abhakliste**. Erst wenn hier alles abgehakt ist, ist der Fork „der neue Kunde".

> **Vorher:** Markenreste finden. Ein schneller Suchlauf nach Name, Ort, Inhaber,
> alter Domain, alter Telefonnummer und **alten Farbklassen** zeigt, welche Dateien
> betroffen sind. Diese Suche ist der erste Schritt — nicht das Raten.

---

## 1. Design-Tokens und Farben

**Datei:** Theme-Konfiguration (z. B. `tailwind.config.js`) und `app/globals.css`.

- [ ] Primär-, Sekundär-, Akzentfarbe auf die Kundenfarben gesetzt
- [ ] **Alias-Klassen mitgezogen** — historisch gewachsene alte Farbnamen zeigen oft
      **noch** auf die alten Werte. Zeigt die Suche Treffer in 20+ Dateien: **nicht**
      einzeln umbenennen, sondern die **Aliase auf die neue Farbe zeigen lassen**
      (kleinste Änderung) — oder bewusst aufräumen, aber dann konsequent.
- [ ] Kontrast geprüft (Text auf Primärfarbe, Buttons, Fokusring) — siehe Skill
      `web-design-guidelines`
- [ ] Dark-Mode/Flächen (falls vorhanden) mitgezogen

> ⚠️ **Nicht nur die `-primary`-Tokens ändern.** Genau das übersieht die alten Klassen.

---

## 2. Logo und Bilder

- [ ] Header-Logo (`components/…Header…`) — `src`, `alt` (mit Kundenname)
- [ ] Startseiten-Hero (`components/…Hero…`)
- [ ] Hero-Slideshow-Bilder (eigene Liste, z. B. `lib/hero-images.ts`)
- [ ] Favicon + App-Icons (`public/`, Icon-Generator-Skript)
- [ ] `public/images/` — alle Fotos des alten Kunden ersetzen/entfernen
- [ ] OG-/Social-Vorschaubild (falls vorhanden)

**Bildnachweis:** Nur Fotos verwenden, deren Rechte der Kunde hat. Stock-Fotos
gehören dokumentiert (Quelle + Lizenz).

---

## 3. Stammdaten

**Zwei Orte, die auseinanderlaufen können — beide prüfen:**

- [ ] `app/api/store/route.ts` — **Fallback-Werte** (Adresse, Telefon, E-Mail,
      Öffnungszeiten, Zubereitungszeit)
- [ ] Wissensspeicher `project_memory`, Key `arch.store_config` (JSON) — die
      **Laufzeit-Wahrheit**, die die Route bevorzugt liest

> ⚠️ Ändert man nur den Fallback, aber nicht `store_config`, sieht der Kunde **alte**
> Daten. **Immer beide** auf denselben Stand bringen.
>
> ⚠️ **`store_config` ist ein Laufzeit-Interface.** In der Vorlage enthält der Key
> genau fünf Nutzfelder (`adresse`, `telefon`, `email`, `oeffnungszeiten`,
> `zubereitungszeit_min`) — weil die Route den Inhalt wörtlich an die Website
> ausliefert. **Jedes zusätzliche Feld wird öffentlich sichtbar.** Feldnamen nie
> umbenennen, Werte nur bewusst ändern.

---

## 4. Liefergebiet

**Datei:** Lieferzonen-Modul (eine Wahrheit pro Wert).

- [ ] Zonen: Name, PLZ-Liste, Mindestbestellwert, Liefergebühr
- [ ] „Zu weit"-Fehlertext mit den **neuen** Ortsnamen
- [ ] Falls PLZ online geprüft werden: Anbieter-Endpunkt prüfen (externer Call)
- [ ] **Betrag im Checkout** wird serverseitig neu gerechnet — Zonen-Werte sind die
      Quelle. Nach der Änderung einen Checkout mit Grenzfall (unter Mindestwert)
      testen.

### 4a. Aktion / Rabatt (z. B. „Lieferung gratis")

Soll der Kunde zeitweise die Liefergebühr erlassen (Werbung, Testkunden, Neustart),
**nicht** die Zone verstellen: die reguläre Gebühr bleibt als **Streichpreis** stehen,
und eine **eine** Schaltstelle im Lieferzonen-Modul berechnet den tatsächlichen Betrag
(Vorlage: `DELIVERY_FEE_FREE` + `currentDeliveryFee(zone)`). Die Anzeige zeigt
`1,99 €` durchgestrichen → `0,00 €` + „Aktion: Lieferung gratis".

- [ ] Reguläre Gebühr **erhalten** (nur die Berechnung auf 0 schalten) — so ist die
      Aktion in einem Schritt wieder aus
- [ ] **Freie Lieferung ≠ kein Mindestbestellwert.** Der Mindestwert bleibt bestehen;
      die Aktion senkt nur den Lieferbetrag
- [ ] **Jede** Ableitung prüfen — die Gebühr ist **nicht** zentral, sie wird an
      mehreren Stellen aus den Zonen gezogen (Website-Kopfzeile, Setup-Dialog,
      Warenkorb, `/api/checkout`, **Admin-Bestellaufnahme**, Mail/Bon). Eine
      vergessene Stelle rechnet weiter den vollen Betrag — siehe Falle 15.

---

## 5. Öffnungszeiten und Wunschzeit

**Datei:** Öffnungszeiten-Modul (Slots aus Freitext + Zubereitungszeit).

- [ ] Freitext-Öffnungszeiten passend zur Kunden-Angabe
- [ ] Vorlaufzeiten (Lieferung/Abholung) plausibel
- [ ] Geschlossen-Logik (Slots für morgen) getestet

---

## 6. Rechtstexte (Pflicht, nicht optional)

- [ ] Impressum (Anbieter, Inhaber, Adresse, Kontakt, Rechtsform, DSA-Kontaktstelle)
- [ ] Datenschutzerklärung (welche Daten, Zwecke, Empfänger, Aufbewahrung)
- [ ] AGB (Lieferung/Abholung, Preise, Zahlung)
- [ ] Widerruf/Kündigung — nur falls Fernabsatz für Verbraucher zutrifft
- [ ] **Barrierefreiheits-Information** — pflichtnah; prüfen

Nutze die Skills `legal-de` und `legal-eu`. **Keine** Texte eines anderen Kunden
übernehmen — Adressen, Zuständigkeiten und Aufbewahrungszeiten unterscheiden sich.

> Der Justiziariat steht beim Kunden, nicht bei dir. Die Texte sind **Vorbereitung**,
> keine Rechtsberatung — im Impressum/der Doku kenntlich machen.

---

## 7. Kassen-Anbindung (der teuerste Bereich)

**Zwei Dateien, identischer Inhalt — beide ändern:**

- [ ] Artikel-Map im **TypeScript** (`lib/…/articles.ts`)
- [ ] Artikel-Map im **PowerShell/JSON** (`tools/…-articles.json`)

- [ ] Format-Modul: `StoreName`, `Referer` (Domain), `Agent`, **OrderID-Präfix**
      (Vorlage: `LB-<id>` — auf den Kunden anpassen) und `PaymentType`
- [ ] **Zahlungsart in der Kasse anlegen.** Der `PaymentType`-Text aus der Bestellung
      (Vorlage: „Online bezahlt") muss im Artikel-/Zahlungsstamm **wörtlich**
      existieren — sonst ordnet die Kasse die Zahlung nicht zu
- [ ] Falls hartcodiert: Kassenbenutzer (`WINORDER_USER` / `RESTAURANT`-Konstante)

> ⚠️ **Jede Kasse hat einen eigenen Artikelstamm.** Die Map ist **kundenspezifisch**,
> kein Copy-Paste-Feld. Grundlage sind echte Bonzeilen **dieser** Kasse.
> Die Kasse ordnet Artikel **per Name** zu — bei Doppelnamen (z. B. ein Belag als
> Calzone **und** als Pizza) bricht die Zuordnung. Solche Doppel im Menü eindeutig
> benennen, **bevor** die Map gebaut wird. Details: `references/winorder-kasse.md`.

---

## 8. Kennungen im Code (verstreut)

Suche nach dem alten Namen und ändere **jede** Stelle:

- [ ] `RESTAURANT`-Konstante (Projektkennung, oft mehrfach definiert)
- [ ] Hartcodierte Kundennamen in API-Routen (Export-Filter, Statusabfragen)
- [ ] Middleware-Realm (Anmelde-Dialog-Text)
- [ ] `SITE_URL` / `NEXT_PUBLIC_SITE_URL` Defaults
- [ ] `.env.example` (Kommentare, Platzhalter-Domain)
- [ ] `package.json` `name`
- [ ] `README.md`, `AGENTS.md`, `docs/*` Kopfzeilen

> ⚠️ **Die Kundenspalte in der Bestelltabelle.** Filter auf den Kundennamen sind im
> Export und im Statusabgleich an mehreren Stellen hartcodiert. Eine vergessene Stelle
> liefert **leere** Exportergebnisse — der Bon bleibt aus.

---

## 9. E-Mail

- [ ] E-Mail-Vorlagen des Webhook-Servers (`email-templates.mjs`): Logo, Farben,
      Anbieter, Rechtsteil
- [ ] Absender (`RESEND_FROM`), Reply-To (`RESEND_REPLY_TO`)
- [ ] Alarm-Empfänger (`ALERT_EMAIL`)
- [ ] Domain im Mail-Anbieter **verifiziert** (neue Domain → neue DNS-Einträge)

---

## 10. App / PWA / Mobile

- [ ] `manifest.webmanifest` (Name, Kurzname, Theme-/Hintergrundfarbe, Icons)
- [ ] Service Worker (`sw.js`) — App-Name/URLs
- [ ] Icon-Generator ausgeführt (`npm run gen-icons` o. ä.)
- [ ] Capacitor/Android: `capacitor.config.ts`, `strings.xml` (falls Mobile-Build)
- [ ] iOS/Android Bundle-ID, falls Store-Veröffentlichung geplant

---

## 11. Abschluss-Suche (die Gegenprobe)

Zum Schluss **erneut** nach allen alten Begriffen suchen. Erwartung: **kein Treffer**
außer in der Historie-Doku.

```powershell
# alte Marke, Ort, Inhaber, Domain, Telefon, alte Farbklassen, alte Konstanten
rg -i "ALTER-NAME|ALTER-ORT|ALTE-DOMAIN|alte-farbklasse|ALTER_ORDER_PREFIX" `
  --glob "!node_modules" --glob "!.next" --glob "!.git"
```

**Treffer in `docs/` sind erlaubt**, wenn sie bewusst als Historie stehen.
Treffer in **Code, Konfiguration oder `public/`** sind ein Fehler.

---

## Abhakliste „fertig entbrandet"

- [ ] Farben inkl. Aliase
- [ ] Logo, Hero, Icons, Fotos
- [ ] Stammdaten (Route **und** `store_config`)
- [ ] Lieferzonen inkl. Fehlertext
- [ ] Öffnungszeiten
- [ ] Impressum / Datenschutz / AGB
- [ ] Artikelmap (TS **und** JSON) + Formattexte
- [ ] `RESTAURANT` + hartcodierte Namen + `SITE_URL`
- [ ] Mail-Vorlagen + Absender + verifizierte Domain
- [ ] Manifest / Service Worker / Icons
- [ ] Abschluss-Suche: kein Treffer
