# Regeln und Fallen

Alle Regeln stammen aus **tatsächlich passierten Schäden** in einem real betriebenen
System. Die Werte sind neutralisiert (`<KUNDE>`, `<KASSE>`, `<DOMAIN>`), die Ursachen
nicht.

---

## Harte Regeln

1. **Ein Kunde = ein eigenes Projekt.** Eigene DB, eigenes Hosting, eigene Domain,
   eigene Zahlung, eigener Kassen-Anschluss. Nichts teilen.
2. **Erst fragen, dann bauen.** `references/intake.md`. Nichts erfinden, was der Kunde
   beantworten kann.
3. **Jede Zustandsbehauptung braucht einen ausgeführten Befehl.** „Müsste laufen" ist
   keine Aussage.
4. **Erst lesen, dann ändern.** Nichts aus dem Gedächtnis annehmen.
5. **Kleinste sinnvolle Änderung.** Kein Umbau nebenbei.
6. **Kein echter Wert in einer getrackten Datei.** Nur Platzhalter + Fundort.
7. **Der Code gewinnt.** Widerspricht eine Doku dem Code, korrigiere die Doku.
8. **Ein Wert steht einmal.** Andere Stellen lesen daraus; die Doku **verweist**.
9. **Dokumentation im selben Vorgang.** Nicht sammeln und später nachtragen.
10. **Kundenkennung konsequent.** Jede Bestellung und jeder Filter benutzt die Spalte.
    Sonst liefert der Export ans Kassensystem **nichts**.

---

## Die teuersten Fallen (real passiert)

### Falle 1 — Mengenfilter löst Massendruck aus

Ein „Testmodus-Filter" war ein **Mengenfilter**. Ein Lauf schrieb **19 Testbons** und
brachte die Kasse **zweimal** zum Absturz.

**Regel:** Für Formatprüfungen immer `-DryRun`. Ein Testfilter ist kein Einzeltest.

### Falle 2 — `status=offen` heißt nicht „alles lief"

Ein Bestellstatus sagt **nichts** über den Mailversand. Nur ein eigener Mail-Zeitstempel
beweist, dass die Bestätigung raus ist. Prüfe **den**, nicht „bezahlt".

### Falle 3 — Ein gültiges Webhook-Secret beweist NICHT den Modus

Test- und Live-Ereignisse sind **beide** korrekt signiert — mit **verschiedenen**
Secrets. Wer sich nur auf die Signatur verlässt, verarbeitet Live-Ereignisse auf einem
Test-System. Der eigentliche Riegel ist der **Soll-Modus** + die Prüfung des
`livemode`-Felds.

### Falle 4 — Zwei Wahrheiten für dasselbe

Stammdaten lagen **im Code** (Fallback) **und** in der Datenbank (Config-Key). Beim
Branding wurde nur eine geändert → der Kunde sah **alte** Daten.

**Regel:** Ein Wert steht **einmal**. Der Fallback im Code liest aus der Config, statt
sie zu wiederholen.

### Falle 5 — Markenrest in Klassennamen

Ein Fork kopierte ein historisches Farbschema, dessen Klassenname die **alte** Marke
trägt — in **20+ Dateien**. Wer nur die neuen `-primary`-Tokens ändert,
lässt die halbe Seite in Altfarben.

**Regel:** Die Abhakliste `references/entbranden.md` vollständig abarbeiten, inkl.
**Alias-Klassen**.

### Falle 6 — Geteilte Datenbank mit Fremdprojekten

Die Vorlage-DB war **mit anderen Projekten geteilt**. Kerntabellen (Menü, Config) hatten
**keinen** Kunden-Schlüssel und **kein** RLS. Mit dem öffentlichen Schlüssel konnte
jeder diese Zeilen lesen **und ändern** — auch die Spalte, die Test- von Live-
Bestellungen trennt.

**Regel:** Eigene Instanz pro Kunde. Wenn geteilt, dann **erst**
`references/template-haerten.md`.

### Falle 7 — RLS einschalten ohne Policies blockiert alles

Der schnelle „Sicherheitsfix" (RLS an) macht **jeden** Zugriff kaputt, wenn die Policies
fehlen. Danach geht **nichts** mehr — Lesen wie Schreiben.

**Reihenfolge:** erst Policies definieren, dann aktivieren, dann testen.

### Falle 8 — UTF-8 vs. ANSI

Eine Artikel-JSON wurde als ANSI gelesen → „ö" wurde „Ã¶" → **kein** Schlüssel mit
Umlaut passte. Die Kasse fand „Pizzabrötchen" nie, der Bon blieb falsch.

**Regel:** UTF-8 **explizit** lesen. Kassen-Konfigdateien oft ISO-8859-1 — ebenfalls
explizit.

### Falle 9 — Falscher Hotfolder

Der Ordner hatte eine **Versionsnummer** im Pfad. Ein „richtig aussehender" Pfad ohne
die Nummer schrieb ins Leere.

**Regel:** Den Basispfad am **Kassensystem selbst** ablesen, nicht aus dem Gedächtnis.

### Falle 10 — Mehrere Sitzungen, ein Repo → Datenverlust

Mehrere Sitzungen im **selben** Repo mit `git reset`/Branchenwechsel löschten
**nicht committete** Änderungen — dreimal.

**Regel:** Konfigurationsarbeit sofort committen. Nicht mehrere Sessions parallel im
selben Repo. Vor Konfig-Arbeit `git status` prüfen.

### Falle 11 — Vercel-Hobby lehnt „fremden" Git-Autor ab

Zwei Erscheinungsformen, **dieselbe Ursache** — der Autor ist dem Hosting-Konto nicht
zuzuordnen. Der Deploy wird `BLOCKED`, **ohne Build (0 ms)**:

| `seatBlock.blockCode` | Wann | Maßnahme |
|---|---|---|
| `COMMIT_AUTHOR_REQUIRED` | Autor-E-Mail gehört zu **keinem** Konto (z. B. Fantasie-Adresse) | **E-Mail** des Autors korrigieren |
| „not a member" | Autor zuordenbar, aber nicht im Team | Autor ins Team holen oder E-Mail ändern |

Am **05.10.2026** belegt: fünf frühere Commits desselben Autors liefen durch, der
sechste wurde blockiert — die Prüfung greift **nicht** rückwirkend, sondern ab dem
Zeitpunkt ihres Eingreifens. Ein grüner Verlauf ist daher **kein** Beweis, dass der
Autor stimmt.

Vercel ordnet über die **E-Mail** zu (`githubCommitAuthorEmail`), **nicht** über den
Anzeigenamen. Der Markenname darf frei bleiben.

**Regel:** Autor **vor dem ersten Commit** setzen — Name beliebig, **E-Mail = Inhaber
des Hosting-Kontos**:

```bash
git config user.name  <markenname>
git config user.email <kontoinhaber@mail>
```

Schon gepusht? Dann neu autorisieren (Baum bleibt unverändert):

```bash
git commit --amend --no-edit --author "<markenname> <kontoinhaber@mail>"
git push --force-with-lease origin main
```

**Diagnose zuerst — nicht den Code durchsuchen.** Bei `BLOCKED` steht der Grund in der
API, nicht im Log:

```bash
vercel inspect --json <deploy-url>     # readyState, readyStateReason
# GET /v13/deployments/<id>?teamId=<team>  ->  readyStateReason, seatBlock.blockCode
```

Ein blockierter Deploy ist **kein** Code- und **kein** Git-Fehler.

### Falle 12 — Env geändert, aber nicht neu deployt

Geänderte Hosting-Variablen wirken **erst nach Redeploy**. Nur Server-`.env` geändert →
`docker compose up -d` genügt; **neuer Code** → Rebuild nötig.

**Regel:** Nach jeder Env-Änderung neu ausrollen und prüfen.

### Falle 13 — Ein MCP „grün" zu nennen ist kein Nachweis

Beim Prüfen des laufenden Musters (03.10.2026) waren die Werkzeuge **nicht** alle
einsatzbereit — obwohl ihre Namen verfügbar aussahen:

- **Supabase-MCP:** Antwort auf die Ziel-Instanz = *„keine Berechtigung"* → Schema und
  Zeilen ließen sich nur über die **HTTP-API** prüfen.
- **Vercel-MCP:** Nutzer-Ebene erreichbar, aber für die **Team-Ressourcen** `403`
  („re-authenticate to this scope").
- **Hetzner-MCP:** **Verbindung im Fehlerzustand** (Tool-Discovery fehlgeschlagen);
  der Server war nur per **SSH** prüfbar.
- **Nur** GitHub- und Resend-MCP antworteten.

**Regel:** Für jede Zustandsaussage den **ausgeführten** Befehl nennen — und wenn ein
MCP fehlt oder streikt, **ersatzweise** per HTTP, SSH oder Dashboard prüfen. Ein
Werkzeug zu besitzen heißt nicht, es nutzen zu können. Die beobachteten Grenzen stehen
in `references/referenz-leckerbissen.md`, Abschnitt 8.

### Falle 14 — Vom veralteten Arbeitsstand geklont

Die Vorlage wird weiterentwickelt. Ein Fork von einem **alten lokalen Stand** erbt
veraltete Skripte und ein altes Schema. Am **03.10.2026** belegt: der lokale
Arbeitsstand der Vorlage war **8 Commits hinter `origin/main`** — `package.json` hatte
dort noch **kein** `check`/`types`/`suche`, und eine neuere Migration fehlte.

**Regel:** Vor dem Fork **aktualisieren** (`git fetch` + `git rev-list --count
HEAD..origin/main` muss **0** sein) oder direkt von `origin/main` klonen. Skriptnamen
und Migrationen **am aktuellen Stand** ablesen, nie aus dem Gedächtnis. Details:
`references/referenz-leckerbissen.md`, Abschnitt 8.1.

---

## Verbote

- 🔴 **Mengenfilter (`-ModeFilter test` o. ä.) nie im echten Hotfolder**
- 🔴 **Kein Schreibvorgang in den Hotfolder ohne ausdrückliche Freigabe**
- 🔴 **Nie zwei Kunden auf derselben DB ohne Kunden-Schlüssel**
- 🔴 **RLS nie ohne Policies einschalten**
- **Secrets nie committen** — Keys, Tokens, `.env.*`-Dateien
- **Nie Kennungen, Tokens, Servernamen oder DB-Namen aus dem Vorlageprojekt in ein
  Kundenprojekt übernehmen.** Jedes Projekt hat eigene Infrastruktur.
- **Rechnungen in der Kasse nie per Skript löschen** — das ist Buchhaltung

---

## Nach jedem Schritt (im selben Vorgang)

| Änderung | Nachziehen |
|---|---|
| Neues Feature | `docs/START.md` → Stand; `CHANGELOG.md` |
| Schemaänderung | `CHANGELOG.md`; Datensatzverzeichnis bei personenbezogenen Daten |
| Neue externe Abhängigkeit | Zugangsregister, Datensatzverzeichnis |
| Architekturentscheidung | **Entscheidungseintrag** (vorher, nicht nachträglich) |
| Vorfall | `docs/RUNBOOK.md` → Störfälle |
| Neuer Befehl | `README.md`, `AGENTS.md` |

Details und Vorlagen: Skill `project-blueprint`.
