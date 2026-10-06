# Kasse anbinden — Bestellung auf den Bon

Beide Betriebswege sind erprobt und liefern **dasselbe Bestellbild**:

| Weg | Wie | Wann sinnvoll |
|---|---|---|
| **A — Hotfolder-Bridge** | Ein Prozess auf dem Kassen-PC **pollt** den Export und legt je Bestellung eine Datei in einen überwachten Ordner | Standard der Vorlage; funktioniert auch, wenn die Kasse keinen Webservice hat |
| **B — REST-Webservice** | Die Kasse **ruft selbst** eine URL ab (`/GetNewOrders`) und meldet den Fortschritt an `/SendTrackingStatus` | wenn die Kasse es kann — kein Dauerprozess, kein Bridge-PC nötig |

Gemeinsame Quelle ist die Bestelltabelle: nur `offen` (bezahlt) geht in die Küche.
Der Hotfolder-Weg ist unten vollständig beschrieben; der REST-Weg ab Abschnitt 4.

```
Website → Zahlung → DB (status=offen)
   → Bridge (Poll alle N Sekunden)   holt offene Bestellungen
   → Hotfolder                        schreibt Kassen-JSON je Bestellung
   → Kassen-Software                  parst, ordnet dem Shop zu
   → Bon auf dem Thermodrucker
   → Bridge quittiert                 PATCH status=in_bearbeitung
```

**Reihenfolge im Statusmodell:** `ausstehend → offen → in_bearbeitung → bereit → abgeholt`
(auch `storniert`).

> ⚠️ **`ausstehend` ist NICHT bezahlt.** Der Status wird beim Anlegen gesetzt, **bevor**
> der Kunde zahlt. Würde die Bridge ihn im Live-Betrieb abholen, könnte ein
> abgebrochener Checkout einen Bon für eine unbezahlte Bestellung drucken. Nur der
> Zahlungs-Webhook setzt `offen`.
>
> ⚠️ **`in_bearbeitung` darf NICHT mitverarbeitet werden.** Das ist der Status, den die
> Bridge selbst setzt. Stünde er in der Liste, würde **jede** übernommene Bestellung bei
> **jedem** Lauf erneut gedruckt.

---

## 1. Die Artikel-Zuordnung (der teuerste Fehler)

Die Website schickt `Margherita`, die Kasse kennt `Pizza Margherita`. Ohne Übersetzung
findet die Kasse den Artikel nicht und legt ihn als neuen Artikel an **oder** macht
einen Kommentar daraus — **keine Umsätze, doppelte Stämme.**

**Zwei Dateien, identischer Inhalt — beide ändern:**

| Weg | Datei |
|---|---|
| App/Format (TypeScript) | `lib/…/articles.ts` |
| Bridge (PowerShell/JSON) | `tools/…-articles.json` |

**Grundlage sind echte Bonzeilen DIESER Kasse** — nicht geraten, nicht vom Vorbild
übernommen.

- [ ] Beispielbons des Kunden exportieren und Artikelnamen auslesen
- [ ] Namensregeln ableiten (z. B. „Pizza trägt Präfix `Pizza `, Dips nicht")
- [ ] Map für **alle** Menü-Artikel des Kunden füllen
- [ ] Unbekannte Namen **nicht raten** — unverändert durchreichen; die Kasse meldet sie
- [ ] Artikel-Nummern (schlagen den Namen) mitführen, falls vorhanden
- [ ] Log auf `FEHLT`-Einträge prüfen **nach jeder** Bestellung

> ⚠️ **UTF-8-Encoding — zwei Stellen, beide schon einmal schiefgegangen.**
> Die JSON-Map **explizit als UTF-8** lesen. Wird sie als ANSI gelesen, wird aus
> „ö" ein „Ã¶" und **kein** Schlüssel mit Umlaut passt mehr. Die Kasse hat dann
> „Pizzabrötchen" nie gefunden.
>
> **Und die Bestelldaten selbst:** Wer die Bestellungen per `Invoke-RestMethod`
> abholt, bekommt **doppelt kodierte Umlaute** auf den Bon (`BÃ¶lstedter`), weil
> PowerShell 5.1 die Antwort mit dem System-ANSI-Zeichensatz dekodiert, obwohl der
> Server UTF-8 sendet und keinen `charset`-Parameter mitschickt. Richtig ist:
>
> ```powershell
> $res  = Invoke-WebRequest -Uri $uri -Headers $Auth -UseBasicParsing
> $json = [System.Text.Encoding]::UTF8.GetString($res.RawContentStream.ToArray())
> return ($json | ConvertFrom-Json)
> ```
>
> Die Kassen-Software war **nicht** die Ursache: sie schreibt in ihren eigenen
> Ausgabedateien korrektes UTF-8. Der Fehler entsteht **beim Lesen**.
> Nachweis: `C3 B6` (sauber) gegen `C3 83 C2 B6` (doppelt kodiert).
> Die Spec verlangt die richtige Kodierung ausdrücklich: *„Beachten Sie die richtige
> Kodierung vor allem von Sonderzeichen. Zur Fehlersuche können Sie die Datei mit dem
> Internet-Explorer öffnen. Hier muss sie richtig angezeigt werden."*

---

## 2. Der Hotfolder

```
<WINORDER-BASIS>\EShop\Incoming      ← hier ablegen
<WINORDER-BASIS>\EShop\Processed     ← Kasse verschiebt hierhin (geparst)
```

- [ ] **Exakten Basispfad ermitteln** — die Ordner- und Versionsbezeichnung kann
      abweichen (z. B. eine Versionsnummer im Pfad). Ein falscher Pfad schreibt ins Leere.
- [ ] Erlaubte Datei-Endungen kennen (typisch `.json`, `.xml`, `.zlib`)
- [ ] Lebenszyklus beobachten:

| Beobachtung | Bedeutung |
|---|---|
| Datei bleibt in `Incoming` | Hotfolder wird nicht beobachtet |
| `.json` → `Processed\*.xml` | **geparst** (Kassen-Bestätigung) |
| umgewandelt, aber kein Bon | Bestellung keinem Shop zugeordnet |

---

## 3. Die Bridge bedienen

```powershell
.\<bridge>.ps1 -DryRun                 # abholen + formatieren, KEIN Bon   ← sicher zum Üben
.\<bridge>.ps1 -Once                   # ein Durchlauf in den Hotfolder
.\<bridge>.ps1 -Once -OrderId N        # NUR Bestellung N (sonst: alle offenen!)
.\<bridge>.ps1 -IntervalSeconds 15     # Dauerbetrieb
```

- [ ] Einzelinstanz (Mutex), damit zwei Läufe **keinen** doppelten Bon erzeugen
- [ ] Ack (`PATCH in_bearbeitung`) nach erfolgreicher Übergabe
- [ ] Bei abgewiesenem Ack einer Testbestellung: **nicht abbrechen**, aber auch **nicht**
      endlos wiederholen (sonst druckt jeder Lauf erneut)

---

## 4. Der REST-Weg (Kasse holt selbst)

Wenn die Kasse einen Webservice anbietet, entfällt der Bridge-PC. Aus derselben
Bestelltabelle speist ein **`GetNewOrders`**-Endpunkt die Kasse, und ein
**`SendTrackingStatus`**-Endpunkt nimmt den Fortschritt zurück. Beide sind erprobt.

**Einrichtung in der Kasse** (Menü der Kasse → Online-Shops):

| Feld | Wert |
|---|---|
| Übertragungsart | Webservice (REST) |
| Webservice-URL | `https://www.<kunde>.de/api/winorder` |
| Benutzername / Kennwort | `ADMIN_USER` / `ADMIN_PASSWORD` |

- [ ] Die Kasse **ruft in ihrem Intervall selbst ab** — kein Dauerprozess nötig
- [ ] Auth wie beim Export: Basic Auth; zusätzlich Kopfzeilen `username`/`password`
      werden akzeptiert, weil Kassen die Zugangsdaten dorthin schreiben
- [ ] Der Abhol-Endpunkt liefert nur `offen`/`ausstehend` — **nie** `in_bearbeitung`
- [ ] Der Endpunkt gibt **WinOrder-JSON** zurück (Content-Type `application/json`)

**Status-Rückmeldung.** Nach Übernahme und bei jedem Fortschritt meldet die Kasse an
`…/SendTrackingStatus`. Die Codes werden auf die App-Status abgebildet:

| Kassen-Code | Bedeutung | App-Status |
|---|---|---|
| `0`, `OK`, `1` | empfangen / wird angenommen | `in_bearbeitung` |
| `2` | in Zubereitung | `bereit` |
| `5` | unterwegs | `bereit` |
| `6` | abgeschlossen | `abgeholt` |
| `7`, `8`, `10` | abgelehnt / rückerstattet / storniert | `storniert` |
| `9` | Übernahme fehlgeschlagen | `ausstehend` |

- [ ] Statuscode-Tabelle **je Kasse** dokumentieren (die Codes variieren)
- [ ] Der Kundenmail-Versand darf die Kassenmeldung **nicht** scheitern lassen
      (`try/catch` — eine fehlgeschlagene Mail ist kein Kassenfehler)
- [ ] Testbestellungen (`stripe_mode = test`) **nie** benachrichtigen

> ⚠️ **Beide Wege brauchen dieselbe Artikelmap.** Ob Hotfolder oder REST: die
> Artikelnamen werden nach derselben Tabelle übersetzt. Sonst ordnet die Kasse falsch
> zu — oder legt den Artikel neu an.

---

## 5. 🔴 Zwei Regeln, die nie gebrochen werden dürfen

### Regel 1 — Kein Mengenfilter im echten Hotfolder

Ein Testmodus-Filter ist ein **Mengenfilter**, kein Einzeltest. Er schreibt **jede**
Testbestellung der Datenbank auf einmal in den Hotfolder.

| Aufruf | Wirkung | Bon? |
|---|---|---|
| `-DryRun` | in ein TEMP-Verzeichnis, kein Ack | **nein** |
| `-Once` (Standard) | nur Live-Bestellungen | nur bei echten |
| `-Once -ModeFilter test` | **ALLE Testbestellungen** | **JA — für jede!** |

**Real passiert:** ein Lauf erzeugte **19 Bons** und **zwei Kassen-Abstürze**. Für
Formatprüfungen **immer `-DryRun`**.

### Regel 2 — Kein Schreibvorgang in `Incoming` ohne ausdrückliche Freigabe

Auch ein Test-Skript, das direkt in den Hotfolder schreibt, erzeugt einen Bon. Nur
bewusst und mit Einverständnis.

---

## 6. Die Kasse einrichten (Vorbereitung, der Kunde klickt)

- [ ] Thermodrucker verbunden (Bluetooth!) — Status **„Verbunden"**, nicht nur „gekoppelt"
- [ ] Kasse: Drucker auswählen, Bon-Typ zuweisen, **Testdruck aus der Kasse** (nicht nur
      aus dem Drucker-Tool)
- [ ] Kasse: Online-Shop / EShop-Schnittstelle aktiv
- [ ] **Unbekannte Artikel: „Immer abfragen"** — **nicht** automatisch anlegen
      (Auto-Create schreibt jeden falschen Namen als neuen Artikel → Müll im Stamm)
- [ ] Hotfolder existiert und wird beobachtet
- [ ] Auto-Druck für Shop-Bestellungen aktiv
- [ ] **Zahlungsart anlegen:** Der `PaymentType`-Text aus der Bestellung (z. B.
      „Online bezahlt") muss in der Kasse unter Stammdaten/Zahlungsarten existieren,
      sonst ordnet die Kasse die Zahlung nicht zu.

### Die Zahlungsarten der Kasse **auslesen**, nicht raten

Der `PaymentType` muss **wörtlich** einem Stammdaten-Wert entsprechen. In einer
Kasse ohne den passenden Eintrag erscheint die Bestellung mit **gelbem Warnzeichen**
und muss von Hand zugeordnet werden.

**Rate nicht, welche Zahlungsarten es gibt** — lies sie in der Kasse nach
(`Stammdaten → Zahlungsarten`) oder suche sie in der Kassen-Datenbank. Eine
Firebird-Kasse lässt sich dafür im laufenden Betrieb nicht direkt kopieren
(Datei-Sperre). Zwei Wege: eine **Sicherung** (`*.wob` entpacken) verwenden, oder
die Datei mit `FileShare.ReadWrite` in eine Kopie lesen.

**Beispiel Leckerbissen** (ausgelesen 06.10.2026 — so sieht ein typisches Ergebnis aus):

| `orders.zahlungsart` | `PaymentType` | Weg |
|---|---|---|
| `online` (oder NULL/Altbestand) | `Online bezahlt` | Online-Zahlung |
| `bar` | `Barzahlung` | Übergabe (Fahrer / Theke) |
| `karte_vor_ort` | `EC-Karte` | Theke (nur Abholung) |

> 🔴 **`Bar` und `Kartenzahlung` gab es in dieser Kasse NICHT** — genau diese zwei
> Texte standen aber im Code. Der Bon hätte ein unbekanntes Zahlungsmittel gezeigt.
>
> ⚠️ **Häufige Falle:** Der Text `Bar` ist naheliegend, heißt in der Kasse aber
> `Barzahlung`. Ebenso gibt es kein `Kartenzahlung`, sondern `EC-Karte`,
> `Kreditkarte` und anbietergebundene Varianten wie `Sumup Kartenzahlung`.
> **Immer nachsehen.** Der PHP-Referenzserver der Kassen-Software verwendet
> `'Barzahlung'`.
>
> **Schreibweise exakt übernehmen** — inklusive Bindestrich (`EC-Karte`) und
> Groß-/Kleinschreibung.

### 🔁 Nach jeder Format-Änderung: den Abhol-Prozess **neu starten**

Ein dauerhaft laufender Abhol-Prozess (Bridge/Dienst) liest sein Skript **beim
Start**. Änderungen am Formatter wirken **erst nach einem Neustart**.

**Real passiert (06.10.2026):** Der Fix für die Zahlungsart lag seit dem Vortag im
Repo, der laufende Prozess hatte aber noch den Stand von **zwei Wochen** vorher — der
Bon zeigte weiterhin das falsche Zahlungsmittel, obwohl der Code längst korrekt war.

- [ ] Nach jeder Änderung an Format/Vorlage den Prozess **neu starten**
- [ ] **Startzeit** des neuen Prozesses prüfen (`CreationDate`) — nicht nur, dass
      überhaupt einer läuft
- [ ] Prüfen, dass der **Wächter** läuft und den Prozess wirklich neu startet. Läuft
      er nicht, bleibt der Kassen-Anschluss nach einem Absturz **stumm** liegen —
      ohne Fehlermeldung, ohne Bon
- [ ] **Der Wächter braucht einen eigenen Einzelinstanz-Riegel.** Hat er keinen,
      erzeugt jeder weitere Autostart einen zweiten Wächter: beide starten Prozesse
      und schreiben in dieselbe Logdatei → der Anschluss **flackert**.
      Real passiert (06.10.2026): zwei Wächter starteten den Abhol-Prozess alle
      ~15 Sekunden neu, bis ein Mutex (`…BridgeWatcher`) das beendete
- [ ] **Zwei Autostart-Wege sind erst mit dem Riegel sicher:** Eintrag in
      `…\CurrentVersion\Run` **und** Verknüpfung im Startordner. Ohne Riegel
      bekämpfen sie sich

> ⚠️ **Ein einzelner Autostart-Eintrag ist kein Nachweis.** Am 06.10.2026 war der
> Run-Eintrag vorhanden, wurde aber nicht ausgeführt — der Prozess hing stattdessen
> an einer Editor-Sitzung. Deshalb zusätzlich eine Verknüpfung im Startordner
> anlegen. Beides zusammen ist gefahrlos — **sobald der Riegel existiert**.

> ⚠️ **Konfigurationsdateien der Kasse nie mit falschem Encoding** lesen/schreiben
> (oft ISO-8859-1). Und: die Kasse liest beim **Start**, schreibt beim **Beenden** —
> Änderungen bei laufender Kasse werden überschrieben.

---

## 7. Tracking zurück (Kunde und Küche)

Die Kasse meldet Statuscodes zurück; die App bildet sie auf Bestellstatus und
**Kundenmails** ab (bestellt → in Zubereitung → unterwegs → fertig).

- [ ] Statuscode-Tabelle je Kasse dokumentieren (Codes variieren)
- [ ] Mails **nie** an Testbestellungen
- [ ] Eine fehlgeschlagene Mail darf die Kassenmeldung **nicht** scheitern lassen
      (`try/catch` in der Route)
- [ ] **Mail-Marker ≠ Bezahlt-Marker.** Die Bestätigungsmail hängt an einem **eigenen**
      Zeitstempel, den **nur** der Mailserver schreibt — nicht an „bezahlt". Sonst gibt
      es Doppel- oder gar keine Mails.

---

## 8. Diagnose „bezahlt, aber kein Bon"

Der **häufigste** Fall zuerst prüfen:

1. **Testmodus?** Ist die Bestellung als Test markiert, wird sie **absichtlich**
   zurückgehalten. Erst das prüfen, dann alles andere.
2. **Läuft der Abhol-Prozess noch — und mit dem *aktuellen* Skriptstand?** Er liest
   sein Skript beim Start. Nach einer Format-Änderung muss er neu gestartet worden
   sein (Startzeit prüfen). Läuft der Wächter nicht, kommt nach einem Absturz
   stillschweigend nichts mehr an — ohne Fehlermeldung.
3. **Status `offen`?** (bezahlt) — nicht `ausstehend`.
4. **Hotfolder:** liegt die Datei in `Incoming` oder schon in `Processed`?
5. **Kassen-Log** auf den Shopnamen prüfen.
6. **Fehlerdatei** der Kasse.
7. **Druckauftrag** in der Warteschlange.
8. **Artikelmap:** `FEHLT` im Bridge-Log?
9. **Umlaute verstümmelt** (`BÃ¶lstedter`)? Dann liest der Prozess den Export mit
   `Invoke-RestMethod` (ANSI) statt explizit als UTF-8.
10. **Zahlungsmittel falsch oder leer?** Der `PaymentType` muss **wörtlich** einer
    Zahlungsart in den Stammdaten entsprechen — die Werte dort nachsehen, nicht raten.
11. **Hausnummer fehlt im Bon-Feld?** Prüfen, ob die Adresszerlegung das Trennzeichen
    des Gastes kennt (Punkt, Bindestrich, Slash — nicht nur Leerzeichen).

```powershell
Get-ChildItem "<WINORDER-BASIS>\EShop\Incoming"
Get-ChildItem "<WINORDER-BASIS>\EShop\Processed" |
  Sort-Object LastWriteTime -Descending | Select-Object -First 3
Get-PrintJob -PrinterName "<drucker>"
```

### Wenn zu viele Bons gedruckt wurden

```powershell
# 1. Druckauftraege SOFORT loeschen
Get-PrintJob -PrinterName "<drucker>" |
  ForEach-Object { Remove-PrintJob -PrinterName "<drucker>" -ID $_.Id }
# 2. Eigene EShop-Dateien entfernen
Get-ChildItem "<WINORDER-BASIS>\EShop\Incoming" -Filter "<praefix>*" | Remove-Item -Force
Get-ChildItem "<WINORDER-BASIS>\EShop\Processed" -Filter "<praefix>*" | Remove-Item -Force
# 3. Laufende Bridge beenden
Get-CimInstance Win32_Process -Filter "Name='powershell.exe'" |
  Where-Object { $_.CommandLine -match '<bridge>\.ps1' } |
  ForEach-Object { Stop-Process -Id $_.ProcessId -Force }
```

> **Die Rechnungen in der Kasse werden NICHT gelöscht** — sie bleiben und müssen dort
> **storniert** werden. Das ist ein Buchhaltungsvorgang, kein Skript-Schritt.

---

## 9. Erfolgsdefinition

Eine Online-Bestellung erscheint **ohne manuelles Kopieren** in der Kassen-Software
und löst einen **Bon** aus. Erst dann ist die Kassen-Anbindung fertig.

- [ ] End-to-End mit **echter** (oder bewusst freigegebener) Bestellung bewiesen
- [ ] Beweis festgehalten: Bestellnummer, Zeit, Rechnungsnummer, Druckauftrag
- [ ] Artikelmap-Log ohne `FEHLT`
