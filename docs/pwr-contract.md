# Der PWR-Vertrag — Projekt-Workspace-Repos als Schnittstelle zwischen den Produkten

Ein **Projekt-Workspace-Repo (PWR)** ist die Datenbasis, auf der Runway, Roger, cockpit,
design-system und help-center gemeinsam arbeiten. `be-plus` ist der erste; die Bausteine
müssen mit vielen funktionieren.

Dieses Dokument legt fest, **welcher Ordner welchem Produkt gehört** und wer ihn liest oder
schreibt. Es ist der Vertrag, gegen den jeder Baustein baut — nicht die Beschreibung eines
Ist-Zustands. Stand 2026-08-01, entstanden aus der Aufarbeitung des be-plus-Anforderungs\-
prozesses.

## Was „Owner" hier heißt — und was nicht

**Owner = normativer Besitzer des Formats.** Wer den Ordner definiert, sein Schema
verantwortet und Änderungen daran entscheidet. Ein Owner ist **keine Berechtigung**.

Zugriff wird nicht in diesem Vertrag geregelt, sondern folgt dem PD-Kernprinzip **„Access from
GitHub, not roles"**: Was jemand sehen und tun darf, ergibt sich aus den Repos, die er beim
Provider tatsächlich sieht. Roger hat dieses Modell übernommen (`roger/specs/006`) und erfindet
ausdrücklich keine eigene Rechteverwaltung. Ein Rollenmodell in diesem Vertrag wäre ein zweites,
konkurrierendes — und damit die erste Stelle, an der die Konstruktion auseinanderläuft.

Es gibt genau **eine Schreibregel**, und sie ist hart:

> **Kein Produkt schreibt Dateien in ein PWR. Dateien entstehen ausschließlich über einen
> Merge Request — von Menschen oder Entwicklungsagenten.**

Roger hält das bereits ein: `roger/specs/008` legt fest, dass Roger weder Code noch Branches
noch MRs erzeugt und genau **ein** schreibendes Werkzeug hat — `create_workspace_issue`. cockpit
ist über den gesamten Delivery-Flow read-only, ohne Writeback. design-system schreibt nur in
seinen eigenen Studio-Projekt-Raum. Damit bleibt das Repo die eine Wahrheit, und jede Änderung
daran durchläuft Review.

## Der Ordner-Vertrag

| Ordner | Inhalt | Owner | liest | schreibt |
|---|---|---|---|---|
| `specs/` | Produkt-Specs als Ordner, je Datei eine User Story oder Journey | Produkt (PWR) | Roger, design-system, Runway-assess | Mensch/Dev-Agent |
| `specs/backlog/` | Anforderungen ohne Spec-Zuordnung, `requested_by` als Nachfrage | Produkt (PWR) | Roger, Runway | Mensch/Dev-Agent |
| `specs/ux/`, `specs/personas.md` | Journeys, Feature-Journey-Matrix, Personas | design-system | design-system (`sync-specs`) | Mensch/Dev-Agent |
| `docs/` | Ist-Zustand: Architektur, Prozesse, Glossar, Onboarding | PWR | Roger (Kernsubstrat), help-center | Mensch/Dev-Agent |
| `ADRs/` | Entscheidungen mit Alternativen und Konsequenzen | PWR | Roger, Runway | Mensch/Dev-Agent |
| `plans/` | Umsetzungspläne laufender Arbeit, flüchtig | PWR | — | Mensch/Dev-Agent |
| `journal/` | ein Eintrag pro PR: was, Entscheidungen, Learnings | PWR | Roger, Runway (Org-Gedächtnis) | Mensch/Dev-Agent |
| `sales/prospects/`, `sales/customers/` | Kontakt-Historie, Eingangsdokumente, Antwortschreiben | Sales | CRM (Abgleich), Roger | Mensch |
| `product-briefs/` | non-tech Wertbeschreibung je Feature | Produkt | Marketing, Sales, Roger | Mensch/Dev-Agent |
| `CLAUDE.md` (auch nested) | Agent-Direktiven für diesen Ordner | PWR | alle Agenten | Mensch |
| `.claude/settings.json` | aktivierte Plugins je Projekt | PWR | Claude Code | Mensch |
| Issue-Tracker (extern) | Vorgänge: Anfragen, Triage, Bugs, laufende Arbeit | cockpit (State-Modell) | Roger, cockpit, Runway | Roger (nur Issues), Mensch/Agent |

Die letzte Zeile ist bewusst dabei: der Issue-Tracker ist Teil des PWR-Substrats, auch wenn er
nicht im Repo liegt. Die Trennlinie ist die Lebensdauer — **Issue = solange offen, Datei = für
die Ewigkeit** (siehe `be-plus/docs/prozess-anforderungen.md`).

## Was jedes Produkt vom PWR erwartet und zurückgibt

### Roger — die Vordertür

Liest alles, schreibt nichts ins Repo. Braucht ein **Workspace-Manifest** (Betreiber-Konfiguration
mit Roots, enthaltenen Repos, Provider-Referenzen, zulässigen Lesequellen). Rät die Struktur
nicht aus dem Dateisystem — Repo-Layouts sind Betreiberkonfiguration, sibling oder nested.

Nimmt Feedback, Fragen, Bugs und Ideen entgegen und legt daraus **Issues** an. Das ist bewusst
die Grenze: aus einem Issue wird ein Backlog-Eintrag durch einen MR, den ein Mensch oder ein
Entwicklungsagent stellt.

### cockpit — Zustand und Koordination

Kennt das PWR nicht als Dateisystem, sondern über den Issue-Tracker: Lebenszyklus-State als
Label (`New → Ready → Planned → Doing → Review → Done`), `epic:` zur Bündelung, `topic:` zur
Bindung an einen Arbeitsfaden. Vollständig read-only, kein Writeback. Liefert zusätzlich
Identität, Mandantengrenze und den Ergebnis-Bus (News als Fortschritt).

### design-system — die Sicht

Liest über `sync-specs` einen festen Satz kanonischer Pfade aus dem Quell-Repo und legt sie als
versionierte Fixtures in einen Studio-Projekt-Raum, **mit Herkunft**. Verbundene Repos stehen in
`repo-quellen.config.json` (`id`, `repo`, `repoRoot`, `status`). Hartes Fehlschlagen bei
struktureller Anomalie statt stillem Überschreiben.

### help-center — das Widget im Produkt

Einbettbare Vue Web Component plus Go-Backend; verfeinert rohes Nutzer-Feedback agent-assistiert
zu Issues. Multi-Tenant von Anfang an. Rendert Hilfeseiten aus Markdown — der natürliche
Herkunftsort dafür ist `docs/` des jeweiligen PWR.

### Runway — Hülle und Regeln

Besitzt den **system-neutralen Issue-Vertrag** (`runway/specs/013`): `id`, `intent`,
`acceptance_criteria`, `topic`, `epic`, `state`, `source`, `readiness`, `risk_tier`,
`provenance` — mit `source: {system: repo-spec, ref: <pfad>}` als ausdrücklich vorgesehenem
Fall. `specs/backlog/*.md` **ist** dieser Fall.

Dazu der Modul-Blueprint mit eingebautem Tracing (`runway/specs/006`): Governance und Monitoring
gehören in jedes Modul, nicht in eine Schicht darüber.

## Vier Konflikte, die dieser Vertrag sichtbar macht

Sie sind nicht theoretisch — sie bestehen heute zwischen bereits gebauten Teilen.

**1 · Zwei Journey-Konventionen.** `design-system/scripts/sync-specs.mjs` liest fest
`specs/ux/journeys.md`, `specs/ux/feature-journey-audit.md` und `specs/personas.md`. be-plus hat
seit ADR-0027 stattdessen `specs/vX-NNNN_name/UJ-NNNN-MM_<slug>.md` — 93 Journeys als
Einzeldokumente mit Front-Matter. Beides sind Journeys, in zwei Formaten.

Empfehlung: **die UJ-Dateien sind die Quelle**, `sync-specs` bekommt einen Resolver, der sie
bevorzugt und auf `specs/ux/journeys.md` zurückfällt. Einzeldateien mit Front-Matter sind für
einen Ingest die bessere Quelle als ein geparster Sammel-Markdown — genau die
Parse-Heuristiken, deren Bruch dort als harter Fehler behandelt wird, entfallen.

**2 · Drei Zustands-Vokabulare.** cockpit-`delivery-flow` (Labels am Issue), Runway 013
(`state` im Vertrag) und der `status` im PWR-Backlog. Normativer Besitzer ist **Runway 013**;
die anderen beiden müssen darauf abbilden. Detail: cockpit führt `roadmap` seit #161 als
**orthogonales Flag**, nicht als State — der be-plus-Backlog kennt es gar nicht. Das muss eine
Festlegung werden, keine Auslegungssache.

**3 · Provider-Annahmen.** help-center schreibt GitHub-Issues, be-plus liegt auf **GitLab**,
Roger ist provider-agnostisch gebaut. Der Vertrag muss den Provider als Eigenschaft des PWR
führen, nicht als Annahme des Bausteins.

**4 · Journeys sind spec-übergreifend.** Ein Onboarding-Ablauf berührt Einladung, Stammakte,
Dokumente und Benachrichtigungen. Die aktuelle Zuordnung „Journey gehört zu der Spec, in der sie
beschrieben war" ist eine Näherung. Da design-system Journeys als **eigene Ebene** über den
Features führt (Feature-Journey-Matrix), ist die Wahrscheinlichkeit hoch, dass die
spec-übergreifende Variante die richtige ist. Vor der Entscheidung auszählen.

## CRM — die Abgrenzung

Das CRM (holunderwunder → Performance CRM, multi-tenant) verantwortet **Lead-Gewinnung und
Marketing-Prozesse**. Produkt-Anforderungen und Feedback sind davon getrennt und leben im PWR.
Diese Trennung ist richtig und sollte nicht aufgeweicht werden: ein CRM optimiert auf
Kontakt-Historie und Funnel, ein PWR auf Nachvollziehbarkeit von Verhalten.

Die einzige nötige Kante ist eine **stabile Kunden-/Prospect-Kennung**. `requested_by` im
Backlog-Eintrag führt sie mit; damit ist beides möglich, ohne eine zweite Datenhaltung:

- **Vom PWR ins CRM:** „diese fünf Anforderungen kommen von DEMA, drei davon sind offen" —
  als Gesprächsgrundlage im Vertrieb.
- **Vom CRM ins PWR:** Fragebögen und Discovery-Antworten aus dem Lead-Prozess landen als
  Eingangsdokument im Prospect-Ordner und, wenn eine Anforderung darin steckt, als
  Backlog-Eintrag.

Was **nicht** passieren sollte: das CRM als Quelle von Produktanforderungen. Ein Lead-Datensatz
beantwortet „wer und wie warm", nicht „welches Verhalten fehlt".

## Die Produkt-Web-UI

Sie ist kein neues Repo. Der Eingang und die Auskunft gehören zu **Roger** — Roger ist per
Definition die non-tech-facing Vordertür und hat mit `009_web-transport` bereits eine
Web-Schiene. help-center liefert das einbettbare Widget und gehört fachlich dorthin
integriert; design-system liefert Tokens und Komponenten; cockpit liefert Zustand, Identität und
Mandantengrenze; das PWR liefert die Wahrheit.

**Korrektur einer früheren Annahme:** Roger soll **keine** Merge Requests schreiben. Das war ein
naheliegender, aber falscher Vorschlag — `roger/specs/008` schließt es ausdrücklich aus und
ersetzt bei Widerspruch die Schreibannahmen der früheren Specs. Der Übergang vom Issue zur Datei
gehört einem Entwicklungsagenten, nicht der Vordertür. Das ist die sauberere Grenze: die
Vordertür nimmt auf, das Substrat wird über Review verändert.

## Nächste Schritte

1. **Backlog-Front-Matter auf Runway 013 abbilden** — `intent`, `acceptance_criteria`, `state`,
   `provenance`, `readiness`. Macht be-plus zum ersten `source: repo-spec`-Konsumenten und
   verhindert ein Parallelvokabular.
2. **Journey-Konflikt entscheiden** — vorher auszählen, wie viele der 93 be-plus-Journeys
   spec-übergreifend sind.
3. **Provider als PWR-Eigenschaft** in Workspace-Manifest und `repo-quellen.config.json`.
4. **Diesen Vertrag gegen ein zweites PWR prüfen.** Er ist aus genau einem entstanden; das ist
   für einen Vertrag zu wenig.

## Quellen

- `runway/specs/013_runway-issue-interface.md` — der Issue-Vertrag
- `runway/specs/010_cockpit-roger-integration.md` — Rang und Verdrahtung der Module
- `runway/specs/006_traceability-blueprint.md` — Tracing im Modul-Blueprint
- `roger/specs/002_workspace-substrat.md` — was Roger vom Workspace erwartet
- `roger/specs/006_identitaet-zugriff-multiworkspace.md` — Zugriff über Provider-Sicht
- `roger/specs/008_workspace-operationen-und-aufruferzugriff.md` — die Schreibgrenze
- `cockpit/specs/delivery-flow/product-spec.md` — State-Modell, read-only
- `design-system/repo-quellen.config.json`, `design-system/scripts/sync-specs.mjs` — Ingest-Vertrag
- `help-center/specs/0001_product_help-center.md` — Widget und Feedback-Pfad
- `be-plus/docs/prozess-anforderungen.md` — der Anforderungsprozess im PWR
