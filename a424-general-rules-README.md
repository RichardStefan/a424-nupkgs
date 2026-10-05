# A424 — Regelwerk der Gesamtlösung

Kurzfassung **aller** verbindlichen Regeln der Solution `A424DataManagement` (Import, Merge, Watermark,
Export mit 23 Datensets, Setup, `A424.Shared`, `A424Rules`).

| | |
|---|---|
| **Stand** | 05.10.2026 |
| **Versionen** | Export **2.81.0** · Merge **1.27.0** · Import **1.16.0** · Setup **1.19.0** · Watermark **1.8.0** · Shared/A424Rules **1.27.0** |
| **Quelle** | die Projekt-Dokumente `claude/regel-*.md`, die `SPEC.md` der Datensets, `README.md` der Adapter und die `docs/*.docx` |
| **Zweck** | eine Seite, die sagt, was gilt — nicht warum. Die Begründung, die Messwerte und der Nachweis stehen im jeweiligen Projekt-Dokument. |

**Pflege:** dieses Dokument ist die *abgeleitete* Fassung. Verbindlich bleibt der Code und die genannte
Langdoku. Jede neue oder geänderte Regel wird **hier nachgetragen** (Abschnitt
[23 Pflege des Regelwerks](#23-pflege-des-regelwerks)).

---

## Inhalt

1. [Grundsätze](#1-grundsätze) · 2. [Plattform und Repository](#2-plattform-und-repository) ·
3. [Kommandozeile und Konfiguration](#3-kommandozeile-und-konfiguration) · 4. [Data-Root](#4-data-root) ·
5. [Exit-Codes und Lauf-Ergebnis](#5-exit-codes-und-lauf-ergebnis) · 6. [Konsole, Log, Statistik](#6-konsole-log-statistik) ·
7. [Zeichensatz der Dateien](#7-zeichensatz-der-dateien) · 8. [Externe Files](#8-externe-files) ·
9. [Merge](#9-merge) · 10. [Watermark](#10-watermark) · 11. [Import](#11-import) ·
12. [Export-Framework](#12-export-framework) · 13. [Export-Filter](#13-export-filter) ·
14. [ARINC-Regelprüfung](#14-arinc-regelprüfung-a424rules) · 15. [Zentrale Werte](#15-zentrale-werte-und-berechnungen) ·
16. [Luftstraßen](#16-luftstraßen) · 17. [Prozeduren](#17-prozeduren) · 18. [Heliporte](#18-heliporte) ·
19. [Simulator-Dateien](#19-simulator-dateien-msfs--x-plane) · 20. [Setup](#20-setup) ·
21. [Neuer Export-Adapter](#21-neuer-export-adapter--checkliste) · 22. [Doku, Tests, Lieferung](#22-doku-tests-lieferung) ·
23. [Pflege](#23-pflege-des-regelwerks)

---

## 1 Grundsätze

| ID | Regel |
|---|---|
| **G-01** | **Geliefert vor gerechnet.** Ein im Record vorhandener Wert gewinnt immer gegen jede eigene Berechnung; gerechnet wird nur, wo das Feld leer ist. (E2/E3, 21.09.2026) |
| **G-02** | **Kein Wert wird erfunden.** Fehlt die Grundlage, bleibt das Feld leer bzw. `NULL` — nie eine ersatzweise `0`. |
| **G-03** | **Ein offensichtlicher Altfehler wandert nicht mit in die Migration** (2.15.0). Die Byteübereinstimmung mit dem Altstand darf dafür sinken; die Abweichung wird dokumentiert und dem Anwender genannt. |
| **G-04** | **Das Sample ist Nachweis, nicht Vorschrift.** Eine Regel des Anwenders schlägt jedes Referenz-Byte. |
| **G-05** | **Eine Regel steht an genau einer Stelle.** Dieselbe Rechnung zweimal im Code ist ein Befund, nicht eine Variante. |
| **G-06** | **Jede Regel gilt für alle Adapter — die heutigen und alle künftigen**, sofern nicht ausdrücklich ein Datenset ausgenommen ist (heute nur `fslabs`, siehe RW-06/W-07). |
| **G-07** | **Jede Zahlen- und Datumsformatierung invariant** (`CultureInfo.InvariantCulture`); kein `ToString()` ohne Kultur, kein `double.Parse` ohne sie. |
| **G-08** | **Erst messen, dann bauen.** Vor jeder Regel, die aus Zeichen oder Mengen ableitet, am echten Zyklus nachzählen (`select distinct`, Spaltenlage, Kollisionen) — nicht der Spezifikationsliste glauben. |
| **G-09** | **Eine Herkunfts- oder Schemaaussage ist erst belegt, wenn die erzeugende Stelle gezeigt ist.** Zusammenpassende Indizien sind keine Quelle; Vermutungen werden als Vermutung gekennzeichnet. |
| **G-10** | **Keine Option für etwas, das Konvention sein kann.** Parameter auf das Minimum; Ablageorte, Logging, Überschreiben, Verifikation und Vergleich sind fix. |

## 2 Plattform und Repository

| ID | Regel |
|---|---|
| **R-01** | Monorepo `A424DataManagement`; gemeinsamer Code **einmal** in `shared/A424.Shared` (+ `.Postgres`, `.Sqlite`, `A424Rules`), Adapter referenzieren per `ProjectReference` — kein NuGet, keine Submodule, nichts kopiert. |
| **R-02** | Zielframework über `A424TargetFramework` (`Directory.Build.props`), Central Package Management (`Directory.Packages.props`), Nullable + XML-Doku an, Gesamtbuild mit `-warnaserror`. Projekt ohne Punkt im Namen = Exe. |
| **R-03** | Build-Ausgabe aller Adapter nach `runtime/<Configuration>/<Adapter>/` (gitignored). |
| **R-04** | **Alle** Solution-Filter (`.slnf`) liegen in `datasets/` — auch die von Import, Merge, Setup, Watermark, Shared. Ein Filter nennt die Gesamtsolution; die Projektpfade im Filter sind **relativ zur Solution**, nicht zum Filter. (01.10.2026) |
| **R-05** | **Jeder Filter hat einen Matrixeintrag in `.github/workflows/build.yml`, jeder Matrixeintrag eine Datei.** Beim Anlegen eines Adapters mitprüfen. |
| **R-06** | Jeder Export-Adapter liegt **direkt** unter `adapters/A424Export/<Ordner>`; der Ordnername nennt das **Datenset**, nicht die Version (`AirbusX_Extended` = v1.15+, `AirbusX_Extended_Legacy` = v1.10−). |
| **R-07** | Schrittreihenfolge eines Zyklus: **Merge → Watermark → Import → Export → Setup**. `scripts/Run-A424Cycle.ps1` und `.github/workflows/cycle.yml` sind die Einstiege. |

## 3 Kommandozeile und Konfiguration

| ID | Regel |
|---|---|
| **C-01** | Gemeinsame Optionen (`CommonOptions`): `--data-dir/-d` (`A424_DATA_DIR`, **Pflicht**), `--cycle/-c` (`A424_CYCLE`, optional — Merge liest ihn aus HDR01, Import nimmt den neuesten Cycle-Ordner), `--profile` (`A424_PROFILE`, Pflicht für Import), `--quiet`, `--verbose/-v`, `--help/-h`, `--version`, `--show-config`. |
| **C-02** | **Vorrangkette jeder Einstellung, immer gleich:** CLI je Datenset/Paket → CLI für alle → Env je Datenset/Paket → Env für alle → Datei-Eintrag → Datei-Sektion → eingebauter Default. |
| **C-03** | **Unbekannter Name:** in der Konfigurationsdatei → Warnung und weiter; auf der Kommandozeile → `UsageException` (Exit 1). |
| **C-04** | Eine **ausgelaufene** Einstellung wird begründet gemeldet, nicht als Tippfehler (`IDatasetExporter.RetiredSettings`, `MovedToSetup`). |
| **C-05** | Boolesche Werte überall gleich: `true/false/1/0/yes/no/on/off`; Schalter über `CommonOptions.Switch` (CLI vor Env). |
| **C-06** | `--show-config` nennt **jede** Einstellung mit ihrer **Herkunft**; die Log-Zeile des Laufs nennt die Werte ohne Herkunft. |
| **C-07** | Passwörter stehen **nie** in `profiles.json` — nur `A424_PG_PASSWORD` / `PGPASSWORD`. |
| **C-08** | **Bewusst entfernte Optionen nicht wieder einführen:** alle `--pg-*`, `--sqlite*`, `--compare*`, `--verify`, `--reconstruct`, `--stage`, `--log*`, `--stats*`, `--allow-*`, `--init`, `--overwrite`, `--output`, `--source-dir`, `--adapters`, alle Merge-Regelschalter sowie die Kommandos `validate`/`verify`/`compare`. |

## 4 Data-Root

| ID | Regel |
|---|---|
| **D-01** | **Kein Adapter setzt einen Pfad selbst zusammen.** Jeder Pfad kommt aus `A424.Shared.Data.A424DataLayout`. |
| **D-02** | Struktur: `a424.json` · `schema/a424-18.json` (genau **eine** Schemadatei) · `profiles/profiles.json` · `external/` (cycle-unabhängig) · `cycles/<cycle>/` mit `source/`, `log-stat/{logs,stats,comp}`, `export/<datenset>/`, `packages/`, `run.<step>.result.json`. |
| **D-03** | `log-stat/` ist dreigeteilt: `logs/` (`<schritt>.log`), `stats/` (`<schritt>.stats.json`), `comp/` (Vergleichsberichte). Direkt in `log-stat/` nur die drei Merge-Arbeitsergebnisse (`merge.origin`, `merge.references.txt`, `A424_Vfr.txt`). Alles wird je Lauf **ersetzt**. |
| **D-04** | **`log-stat/` wird als Workflow-Artefakt hochgeladen** — dort darf nichts liegen, was den Datenbestand kompromittiert, **insbesondere nicht das Watermark-Register** (`watermark.journal` zeigt außerhalb des Data-Roots). |
| **D-05** | Ordner-Anlage: der **Merge** legt den Cycle-Baum an, der **Export** bei Bedarf aus der Datenbank heraus; **Import und Setup legen nichts an**. |
| **D-06** | Eingabefile-Auflösung: nur Dateiname → `source/`; Pfad → so verwendet; `archiv.zip::eintrag` → ZIP-Eintrag; nur Adaptername → Namensmuster des Adapters, Suchreihenfolge siehe **E-03**. |
| **D-07** | Datenset-Name überall klein mit Bindestrich (`vatsim-aeronav`); `export/<datenset>/` gehört dem Datenset, wird je Lauf **komplett ersetzt** und enthält **nie** das Archiv. |
| **D-08** | Merge-Ergebnis heißt `<stem>.merged.<ext>` und liegt in `source/`. **Ein Archiv ist nie ein Merge-Ergebnis** (`A424DataLayout.IsArchive`: `.zip .7z .rar .gz .tgz .bz2 .xz .tar`) — ein `*.merged.zip` daneben ist eine abgelegte Kopie und wird übergangen. Zwei echte Ergebnisse bleiben ein Fehler. |
| **D-09** | Ein **Providerfile darf** gepackt sein (`HeaderRecord1.PeekCycle` schaut hinein); ein Merge-Ergebnis entsteht immer ungepackt. |
| **D-10** | Manifest eines Export-Datensets für das Setup ist `log-stat/stats/export.<datenset>.stats.json`. |

## 5 Exit-Codes und Lauf-Ergebnis

| ID | Regel |
|---|---|
| **X-01** | Einheitlicher Vertrag für **alle** Adapter (`A424.Shared.Diagnostics.ExitCodes`): **0** success · **1** usage · **2** validation · **3** target · **4** verification · **130** cancelled. |
| **X-02** | **Warnungen ändern den Exit-Code nie.** Ein Lauf kann `errors > 0` und Exit 0 haben (z. B. fehlendes Pflicht-File im `external`-Inventar). |
| **X-03** | Sammellauf: **die höhere Zahl gewinnt** (`ExitCodes.Worst`); ein Code außerhalb des Vertrags wird auf **3** normalisiert. |
| **X-04** | Je Lauf ein Urteil in `cycles/<cycle>/run.<step>.result.json` (Schema `a424.run-result/1`, `Parts[]` je Datenset/Paket) — **direkt im Cycle-Ordner**, nicht in `log-stat/`. Steuerbar über `--result-file` / `A424_RESULT_FILE` / `resultFile`; Wert `off` schreibt nichts. Ohne Cycle gibt es keine Datei — **der Exit-Code ist der Vertrag, nicht die Datei.** |
| **X-05** | Jeder Adapter schreibt `exit_code`, `status`, `success`, `warnings`, `errors`, `message`, `result_file` (auch mit Schritt-Präfix) nach `GITHUB_OUTPUT` und eine Teile-Tabelle nach `GITHUB_STEP_SUMMARY`. |
| **X-06** | Im Workflow macht **genau ein** Schritt (*Evaluate result*) aus dem Code ein Job-Ergebnis; dort gilt **4 bewusst als Fehler**. |
| **X-07** | Das Schlussurteil schreibt **eine** Stelle farbig: `RunResults.WriteVerdict` — 0 grün, 4 gelb, alles andere rot. Keine Farbe bei `--quiet`, `NO_COLOR`, umgeleiteter Ausgabe oder Konsolen-`IOException`; Kanal bleibt stdout. |

## 6 Konsole, Log, Statistik

| ID | Regel |
|---|---|
| **L-01** | **Konsole kurz, Log-File vollständig.** `LogEntry.ConsoleMessage`: `null` = gleicher Text auf beiden Wegen, Text = nur Konsole, `""` = nicht auf der Konsole. Präfixe bleiben gleich, damit Konsole und File spaltengleich stehen. |
| **L-02** | **Warnungen und Fehler stehen immer auf der Konsole** — verlagert werden nur ihre Anhänge (Listen, Einzelbefunde), nie das Urteil. |
| **L-03** | `--verbose` / `-v` / `A424_VERBOSE=1` / `"verbose": true` holt die vollständige Konsole zurück; `--quiet` gewinnt. |
| **L-04** | Log-Ebenen unverändert: Logger ab `Info`, `Detail` (Einzelrecords) aus. Log-File, Statistik, Berichte und DB-Logtabelle ändern sich durch die Kurzkonsole **nie**. |
| **L-05** | Fortschritt: **je Quelle/Datenset eine Konsolenzeile**, nur Zahlen ≠ 0. |
| **L-06** | Logging per Schalter zu- und abschaltbar; Log als Tabelle (Datenbank) bzw. Textfile, dazu je Lauf eine Statistikdatei. Logs und Statistik sind **UTF-8**. |

## 7 Zeichensatz der Dateien

| ID | Regel |
|---|---|
| **A-01** | **Alle Nutzdaten-Files von Import, Merge und Watermark sind einbyteig: striktes ASCII (7 Bit), längentreu.** Immer an, kein Schalter. (29.09.2026) |
| **A-02** | `A424.Shared.Files.SingleByteText` ist die Regel: 523 ausgeschriebene Einträge (`ß`→`S`, `Ø`→`O`, `°`→Leerzeichen, typografische Striche/Anführungszeichen …), **ein Zeichen wird ein Zeichen**, damit die Spaltenlage eines 132-Zeichen-Records hält. Unbekanntes wird `?` und zählt getrennt. Keine Laufzeitnormalisierung (`string.Normalize` hängt an ICU). |
| **A-03** | Einbyteig sind: Merge-Ergebnis, `fixedwing-<cycle>.txt`, `helicopter-<cycle>.txt`, `A424_Vfr.txt`, Origin-Map, markiertes Merge-Ergebnis, `watermarks.jsonl`. **Unberührt**: `raw-<cycle>.txt` (byteweise Kopie bleibt byteweise Kopie). |
| **A-04** | **Der Import weist ab, statt umzuschreiben**: jedes Byte außerhalb `0x20–0x7E` beendet den Lauf mit **Exit 2** und nennt Zeile, Spalte, Byte — sonst bräche der byteweise Round-Trip. |
| **A-05** | Ein Umschreiben warnt (erste **20** Funde mit Zeile/Spalte/Zeichen/Ersatz), der Lauf bleibt erfolgreich; Zähler `SingleByteCharactersReplaced`, `SingleByteLinesChanged`, `SingleByteCharactersWithoutEquivalent`. |
| **A-06** | Ausnahme `VfrTables.ToAsciiName` darf Ligaturen ausschreiben (`Æ`→`AE`), weil es ein **Feld** baut, dessen Breite danach gesetzt wird — `SingleByteText` steht am Dateirand und ist längentreu. |

## 8 Externe Files

| ID | Regel |
|---|---|
| **E-01** | Alle cycle-unabhängigen Files liegen **direkt** in `<root>/external/` unter ihrem Liefernamen. Nur zwei Unterordner, für Files mit wechselnden Namen: `external/faa/` (`CIFP_<cycle>.zip`) und `external/tailored/`. (30.09.2026) |
| **E-02** | **`A424.Shared.Data.ExternalFiles` ist die eine Stelle**, die die Liste hält (Name, Art, Pflicht/Optional, Zweck, Leser). Es ist **keine Konfiguration**: `import.codeTables`, `merge.sources` und `simulatorDataFolder` gewinnen weiter — das Register legt nur fest, wo **ohne** Einstellung gesucht wird. |
| **E-03** | Suchreihenfolge je Merge-Adapter (`ISourceAdapter.ExternalSubFolder`): `faa` → `external/faa/`, `source/`, `external/`; `tailored` → `external/tailored/`, `external/`, `source/`; `sim` → `external/`, `source/`; `gates`/`vfr` → `source/`, `external/`. |
| **E-04** | Nur zwei Zustände: **`Present`** und **`Missing`**; `Misplaced` entfällt. In `external_files` der `import.stats.json` steht `present`, `absent (optional)` oder `missing` — **nie ein Pfad**. |
| **E-05** | Ein fehlendes **Pflicht**-File ist ein **ERROR** (rote Zeile), erhöht `errors`, **ändert den Exit-Code aber nicht**. Geprüft in Import-Schritt 2 jedes Laufs und identisch in `A424Import show-config` ohne Datenbank. Fehlt `external/` ganz, ist das ebenfalls ein Fehler. |
| **E-06** | Optional mit Ersatz: fehlt eine Simulatorliste, greift die des anderen Simulators mit Warnung und Zähler `simulator.filesOfTheOtherSimulator`. Fehlt eine **optionale** Liste ganz, läuft der Lauf mit Warnung weiter — jeder Wert hat einen Default. |

## 9 Merge

| ID | Regel |
|---|---|
| **M-01** | Der Merge verschmilzt externe Files in **ein** ARINC-424-Source-File; Regeln sind **fix, nicht schaltbar**: Sortierung, vollständige Neunummerierung, Korrektur der Recordzahl im Header, Entity-Key-Abgleich, Duplikatprüfung gegen den Provider, Cycle-Prüfung. |
| **M-02** | **Jede Zeile wird geprüft** (`record length … instead of 132` bricht hart ab), das Ergebnis wird **noch einmal** geprüft (`FilteredFileWriter`). Der Merge bleibt streng. |
| **M-03** | Nicht ARINC-konforme externe Files werden **in das Schema des aktuellen Zyklus** gebracht, bevor sie verschmolzen werden; der Merge ist ein eigener Adapter im Workflow. |
| **M-04** | **Der Merge fasst keine gelieferten Records an und erzeugt keine, die der Provider nicht geliefert hat.** Jede Auflösung (Circle-to-Land, `RWnnB`) geschieht **im Export**, nicht hier. (Merge 1.27.0) |
| **M-05** | Herkunft je Record: der Merge schreibt die **Origin-Map** (`log-stat/merge.origin`, Bereiche `von-bis Quelle`, lückenlos, 1-basierte Datenrecord-Ordinale); der Import füllt daraus die Spalte `source`. Fehlt die Map, bekommt alles das Provider-Label (Log-Hinweis). |
| **M-06** | Quell-Label über `Naming.ToSourceLabel` (`Jeppesen`, `Gates`, `Faa`, `Tailored`, `Vfr`, …). |
| **M-07** | **Blindfleck des Altparsers, nicht nachgebildet** (29.09.2026): der alte Parser verlor 2.699 Flugplätze, sobald eine Subsection außer der Reihe kam (Gate-Record `G`, Bahn-Record `M`, Bahn-Record `F` an einem Provider-Flugplatz). **Alle Flugplätze werden exportiert, kein Schalter.** Für die betroffenen Ausgaben ist das Sample damit kein Byte-Abnahmekriterium mehr. |
| **M-08** | Leer-/Filler-Records des alten Source-Files werden **überlesen, nicht korrigiert** (`Arinc424Layout.IsBlankFillerRecord`, `tolerateBlankFillerRecords`): Warnung (erste 20), Zähler `blank_filler_records`, Ablage **byteexakt** in `sys_unmatched` (Status `BlankFiller`). Jede andere Längenabweichung bricht ab. |
| **M-09** | Referenzprüfung über **alle** relevanten Recordtypen; der Bericht offener Referenzen ist `log-stat/merge.references.txt`. |

## 10 Watermark

| ID | Regel |
|---|---|
| **W-01** | Der Watermark-Adapter ist ein eigener Schritt und arbeitet **in place** am Merge-Ergebnis (temporär `.inserting`, dann ersetzen) und zieht die Origin-Map nach — danach sieht alles **genau eine** Quelldatei je Cycle. |
| **W-02** | Der Ident folgt **ARINC 424-18, 7.3.6**: Position 1–2 `VP`, `VC`, `VF` oder `VS`, Position 3–5 numerisch; Waypoint Type `V` in Spalte 27; **Name = Ident**. |
| **W-03** | **Eindeutig im ganzen Area Code**, nicht nur im 50-NM-Umkreis: gesammelt werden alle Kennungen dieser Form aus `EA`, `PC` und `HC` je Area Code. |
| **W-04** | **`VP`/`VC` bevorzugt** (1.998 Kombinationen); `VF`/`VS` erst, wenn im Area Code keine bevorzugte Kennung mehr frei ist. |
| **W-05** | Register: **eine Zeile je Wasserzeichen**, Identität = (Cycle, Ordinal, RecordSha256) über Primary **und** Continuation. Bekannt → `Runs`+1, `LastSeenUtc`, ggf. `LastAdapterVersion`; unbekannt → neue Zeile. |
| **W-06** | Nur die Buchführungsfelder (`Runs`, `LastSeenUtc`, `LastAdapterVersion`) sind **nicht** signiert; alle beschreibenden Felder werden nie wieder angefasst. Format 2: `Previous` = **Signatur** der Vorgängerzeile, Signatur über **14** Felder. |
| **W-07** | Alte Zeilen werden gelesen, nach der Regel **ihres eigenen Formats** geprüft und **nie umgeschrieben**. |
| **W-08** | `--compact-register` prüft erst die **gesamte** Kette und verweigert die Arbeit bei jedem Befund; die ersetzte Datei bleibt als `<name>.before-compact` liegen. |

## 11 Import

| ID | Regel |
|---|---|
| **I-01** | Der Import legt **Tabellen und Spalten selbständig aus dem JSON-Schema** an und füllt sie aus dem Source-File. Genau **eine** Schemadatei in `schema/`. |
| **I-02** | **Die Datenbank ist immer das Original.** Der Import verändert, klont und filtert **nichts** — jede Auflösung geschieht im Export. |
| **I-03** | Quelle: `*.merged.*` in `source/`, sonst das Providerfile (die einzige Datei dort, die mit `HDR01` beginnt). |
| **I-04** | **Verify und Compare laufen immer**: byteweiser Round-Trip gegen das Source-File und Vergleich gegen den Vorzyklus; Berichte in `log-stat/comp/`. **Ein Cycle wird immer ersetzt.** |
| **I-05** | Jede Daten-, Header- und Unmatched-Tabelle trägt `source` **VARCHAR(32) NOT NULL, indiziert, direkt nach `raw_record`**. Header-Zeilen tragen immer das Provider-Label. |
| **I-06** | `sys_unmatched.raw_record` ist **`VARCHAR`** (nicht `CHAR`) — der einzige Ort, der Records abweichender Länge halten darf; `CHAR` würde beim Lesen auffüllen und den byteexakten Nachweis brechen. |
| **I-07** | HDR-Zeilen nach `a424_hdr_header` mit `run_id`, `importer_version`, `imported_utc`. Fehlt der Cycle-Ordner: Exit 1 mit „run A424Merge first". |

## 12 Export-Framework

| ID | Regel |
|---|---|
| **F-01** | **Jedes Datenset ist ein eigener Adapter**, einzeln lauffähig und testbar; dazu ein Runner, der alle oder ausgewählte Datensets in einem Lauf fährt (`--dataset`). Dazwischen liegt die Abstraktionsschicht `A424Export.Core`. |
| **F-02** | **`ExportContext.Source` ist immer der Wrapper `FilteringExportSource`** — der init-Setter wickelt die rohe Quelle ein; `Inner` ist die ungefilterte Quelle. `Scalar` bleibt ungefiltert. |
| **F-03** | Jeder Plan (`ExportableAirports`, `ArincRulePlan`, `CircleToLandPlan`, `BothRunwaysPlan`, `HelicopterData`, `EnrouteAirways`) wird **einmal je Lauf aus der ungefilterten Quelle** gebaut — unabhängig von der Form der Abfragen eines Datensets. |
| **F-04** | Der Wrapper erkennt die Tabelle per `FROM a424_…`; Section = 6. Zeichen, Subsection = 7. Zeichen des Tabellennamens. |
| **F-05** | **`raw_record` ist Pflicht, wo ARINC-Regeln gelten**: jede Abfrage auf eine Tabelle mit Bahnkennung (PG, PI, PL, PM, PT, PP, PD, PE, PF, HD, HE, HF) wählt `raw_record` mit — sonst greift `arincRules` dort nicht. (Grundsatz 04.10.2026, 2.81.0) |
| **F-06** | Fehlt `raw_record` bzw. die Kennungsspalte, **warnt** das Rahmenwerk **einmal je Tabelle** und lässt die Zeilen **ungefiltert** durch. Das ist der Vertrag — kein Abbruch. Zwei Selbsttests sichern ihn für künftige Adapter. |
| **F-07** | **Kein Datenset enthält zwei Flugplätze mit derselben exportierten Kennung.** Der Schlüssel ist die **Vier-Zeichen-Kennung**; gesucht wird unter **beiden** Kennungsregeln (`Identifier4ByArea`, `Identifier4`). Es überlebt einer nach `ExportableAirports.RecordClassPrecedence = "STFMV"` (bei gleicher Klasse ordinal nach Kennung, nie nach `id`), jeder andere entfällt **samt allen seinen Records**. Gilt **unabhängig von `airportsWithoutRunway`**. Zähler `airports.skippedDuplicateIdentifier` + Warnung je Kollision. |
| **F-08** | Ein Record, der auf einen nicht exportierten Flugplatz verweist, entfällt mit ihm (Bahnen, Prozeduren, Wegpunkte, NDB, Gates, Com, Localizer, Marker, MSA, Pfadpunkte, GLS, EP-Holdings an seinen Fixen) — **nichts im Datenset verweist auf etwas, das das Datenset nicht enthält**. |
| **F-09** | Continuation-Records folgen immer ihrem Primary: wird ein Primary übersprungen, ersetzt oder geklont, **fallen bzw. folgen alle seine Continuations**. |
| **F-10** | **Kein Kommentarkopf in einer Ausgabedatei**, die das Format nicht vorsieht — kein „generated by", kein Zeitstempel. |
| **F-11** | Nichtdeterministische Felder (`timestamp=`, `build=`, `TIMESTAMP`, SQLite-Headertabellen) sind als **volatil** gekennzeichnet und werden beim Bytevergleich normalisiert, nicht verglichen. |
| **F-12** | `--resume` ist **Opt-in**, arbeitet **je Datenset bzw. je Paket** und stützt sich allein auf das Urteil `"result": "success"` der Statistikdatei; kein zusätzlicher Zustand. Ohne den Schalter läuft alles von vorn — **nie wird still übersprungen**. |
| **F-13** | Übersprungen wird nur bei lückenloser Kette: Statistikdatei dieses Cycles vorhanden und `success`, Adapter-Version unverändert, Einstellungs-Fingerabdruck unverändert, (Setup) Export nicht neuer als das Paket, Ausgabe vorhanden und nicht leer. Jeder Grund steht in Konsole und `run.<step>.result.json` (`skipped: true`). Der **Exit-Code bleibt unberührt**. |
| **F-14** | Die Entscheidung fällt **vor** dem Anlegen des Loggers (das Anlegen kürzt die Logdatei); ein übersprungener Teil behält alle Dateien des erzeugenden Laufs. |
| **F-15** | `settingsFingerprint` = 16 Hexziffern (SHA-256, erste 8 Bytes) über **die Einstellungen, die den Teil erreichen** — nicht über die ganze Konfigurationsdatei. Eine neue Einstellung **muss** dort hinein. |

## 13 Export-Filter

**Alle Filter gelten für jedes Datenset, das heutige und jedes künftige.** Global `export.filters`, je Datenset
`filters` im Eintrag; `--filter name=wert`, `--filter datenset=name=wert`, `A424_EXPORT_FILTERS`. Vorrang nach **C-02**.

| Filter | Default | Was der Default bedeutet |
|---|---|---|
| `circletoland` | **false** | CTL-Anflüge werden **aufgelöst** — je Bahn ein Klon, die Klone ersetzen das Original. `true` = wie geliefert. |
| `writeBRunways` | **false** | `RWnnB`-Transitionen werden **aufgelöst** — je physischer Bahn eine Kopie, die Kopien ersetzen das Original. `true` = wie geliefert. |
| `airportsWithoutRunway` | **false** | Flugplätze ohne Bahn-Record mit Schwellenkoordinaten werden **weggelassen**, samt allen ihren Records. |
| `includeHelicopterData` | **false** | **Kein** Datenset liefert Helikopterdaten, solange sein Eintrag den Filter nicht einschaltet (heute nur `tds-gtnxi`). |
| `arincRules` | **true** | Nur Records, die die Regeln von `A424Rules` erfüllen, werden exportiert. |
| `heliportsAsWaypoints` | **false** | Heliporte werden nicht ausgeliefert (siehe Abschnitt 18). |

| ID | Regel |
|---|---|
| **FL-01** | **Entweder–oder, nie beides:** eine Auflösung **ersetzt** das Original an seinem Platz; Original und Klon dürfen nicht zusammen in einer Ausgabe stehen. |
| **FL-02** | Ein Klon ist **1:1 — nur die Kennung ändert sich.** Kein geänderter Recordtyp, kein geänderter Area Code; Primary und alle Continuations. |
| **FL-03** | Was **nicht** auflösbar ist (keine passende Bahn, alle Kennungen belegt), bleibt **wie geliefert** und wird gewarnt. Ein Duplikat wird übersprungen und gewarnt; eine Kollision mit einem bereits vorhandenen Record wird übersprungen (Detail). |
| **FL-04** | `includeHelicopterData=false` entfernt dreierlei: die **ganze** Heliport-Sektion (jede Tabelle `a424_h?_*`, **nicht** `a424_hdr_header`), die **Helikopter-Legs** der Flugplatzprozeduren (PD/PE/PF, Spalte 120 ∈ `H`/`I`/`L` am Primary, mit Continuations) und die **EP-Holdings an Heliport-Fixen** (Spalte 37 = `H`). |
| **FL-05** | Gefiltert wird nach **Recordtyp, nicht nach Verweis**: ein Record, der einen Heliport nur nennt, bleibt. |
| **FL-06** | Alle Filterwerte stehen in der Statistik (`filters`) und in `--show-config` mit Herkunft. |

## 14 ARINC-Regelprüfung (`A424Rules`)

| ID | Regel |
|---|---|
| **AR-01** | Die Regelbibliothek `shared/A424Rules` ist die **eine** Stelle, an der eine ARINC-Regel als Code steht; der Export wendet sie über `ArincRulePlan` an, gesteuert von `arincRules` (Default **true**). |
| **AR-02** | **Bahnkennung nach 5.46:** `RW` + `01`–`36` + optional `L R C T W` (`^RW(0[1-9]\|[12][0-9]\|3[0-6])[LRCTW]?$`); als **Transition** zusätzlich `B` (5.11). |
| **AR-03** | **Als Bahnkennung gilt nur `RW` + zwei Ziffern.** Eine Transition, die nach einem **Fix** heißt (`RWF`, `RWO`, `RWA1`, `RWA2`), ist keine Bahnkennung und wird **nicht** gefiltert. (Anwenderbefund 04.10.2026, 2.81.0 — bis 2.80.2 wurden dadurch 49 Records aus 15 Anflügen in jedem Datenset still verloren.) |
| **AR-04** | Wo die Bahnkennung steht, nach Recordtyp: **PG** 14–18 · **PI, PL, PM, PT** 28–32 · **PP** 20–24 · **PD, PE, PF, HD, HE, HF** 21–25. |
| **AR-05** | Ein regelwidriger Record wird **im Export** weggelassen — **Merge-Ergebnis und Datenbank behalten ihn wie geliefert**. `false` exportiert den Zyklus wie geliefert; ein Datenset mit Fremdschlüsseln (DFD) kann daran scheitern. |

## 15 Zentrale Werte und Berechnungen

| ID | Regel |
|---|---|
| **V-01** | **Jeder abgeleitete Wert hat genau eine Routine** in `A424Export.Core` (`ArincValues`, `RunwayBearing`, `RunwayGradient`, …). Kein Datenset rechnet eigenständig nach. |
| **V-02** | **Erde und Koordinaten:** WGS 84; Distanzen und Peilungen geodätisch (Vincenty); Geoid-Undulation aus `egm96-5.pgm`; Missweisung aus dem Weltmagnetmodell nur dort, wo kein gelieferter Wert existiert. |
| **V-03** | **Gegenbahnkennung** nur über `ArincValues.OppositeRunway`: Nummer `n+18` bis 18, `n−18` darüber, **immer zweistellig**; von den Designatoren tauschen **nur `L` und `R`** — jedes andere Zeichen (`C`, `W`, `T`, `U`, `S`, `G`, Ziffern) steht an beiden Enden gleich. Keine Bahnform → leere Kennung. |
| **V-04** | **Wahrer Bahnkurs** über die zentrale Kette `RunwayBearing.True`: gelieferter Kurs des Simulation-Records (Sp. 52–56) vor der Geometrie der beiden Schwellen. |
| **V-05** | **Bahngradient** (`RunwayGradient`, DFD-Familie): geliefertes Feld Sp. 52–56 ÷ 1000 in Prozent → sonst gerechnet `(Höhe Gegenschwelle − eigene Schwellenhöhe) / geodätische Schwellendistanz × 100`, drei Nachkommastellen, gedeckelt bei ±9,000 % → sonst `NULL`. Nenner ist die geodätische Schwellendistanz; ohne Koordinaten oder unter **30 m** Schwellenabstand gilt die Bahnlänge Sp. 23–27. |
| **V-06** | **Das ILS einer Bahn ist ausschließlich der Localizer, den ihr PG-Record in Sp. 82–85 nennt** (`RecordIndex.IlsByRunwayAndIdentifier`). Kein Treffer → **kein ILS**, keine Ersatzwerte. Ausnahme: WPNAV-Familie (`leveld`) und `a320pic` nehmen über `IlsByRunway` weiter den Localizer der niedrigsten Kategorie. |
| **V-07** | **`Ils@magvar` wird gerechnet**: veröffentlichte magnetische Peilung (PI 52–55) − wahrer Bahnkurs (V-04). Gedreht wird nur eine **ausgerichtete** Anlage — Grenze **3,0°** (`IlsElements.MaximumAlignmentDegrees`) und Kategorie `0`–`3`. Alles andere behält die Kette Stationsdeklination (PI 91–95) → Final Approach Course Fix → Weltmagnetmodell. Der Positionsbezug (Sp. 79) ist **kein** Ausschlussgrund. |
| **V-08** | **Wahrweisende Station** (`trueReferenced="TRUE"`): `magvar = 0`, `heading` = veröffentlichter Kurs − Missweisung der Kette (der BGL-Compiler lässt daneben keine Missweisung zu, `#C2676`). Bei wahrweisender VOR (`T0000`) wird das Vorzeichen der Null unterdrückt (`0.00000`); **minus Null bleibt überall sonst erhalten**. |
| **V-09** | **`Ils@alt`** = Schwellenhöhe der **Gegenbahn** (PG 67–71), Fallback Flugplatzhöhe (PA 57–61) — der Wert steht für den Antennenstandort hinter dem fernen Bahnende. |
| **V-10** | **`GlideSlope@alt`** — fünfstufige Kette: (1) Tabelle `msfs2024_gs_altitudes.txt` über Flugplatz + Bahn + Localizer, **Frequenz muss stimmen** · (2) dieselbe Tabelle über Localizer + ICAO-Code + Frequenz, **nur für die Zeilen ohne Platz und Bahn** · (3) PI 98–102 · (4) PG 67–71 · (5) PA 57–61. Nur `msfs2024` liest die Tabelle; `msfs2020` beginnt bei (3). Höhe mit einer Nachkommastelle (2024) bzw. ganzen Fuß (2020). |
| **V-11** | **`GlideSlope@thresholdCrossingHeight`** = PI 96–97, sonst PG 76–77, sonst **50 ft** (PI vor PG, weil PI zu diesem Gleitweg gehört, PG zum Verfahren der Bahn, 5.270). Die Schwellenüberflughöhe wird **nicht** erzwungen; alle Gleitweg-Rechnungen sind zurückgenommen (2.49.0). |
| **V-12** | **Bogenpeilung einer Luftraumgrenze:** ein Satz mit Boundary Via `L`/`R` liefert **18 Punkte**; die **Startpeilung ist die gelieferte Bogenpeilung** (5.147, Zehntelgrad), die Endpeilung bleibt die gerechnete Peilung zum nächsten Punkt. **Ausnahme letzter Satz** (2. Zeichen = `E`): dort gilt die gerechnete Peilung, damit der Bogen auf dem Anfangspunkt schließt. Gilt für **jede** Auflösung von UC/UR/UF in Punkte. |
| **V-13** | **Dublette an einer FIR-Grenze:** ein Punkt, den zwei Grenzsätze gleich nennen, wird einmal ausgegeben. |
| **V-14** | `fslabs` ist **ausgenommen**: dort gilt die mitgelieferte **Schemadatei** als Format (`RUNWAY_GRADIENT` in Hundertstel, `MAG_VAR`, `THETA`, `RHO`, `GRID_MORA` …). Die Schemadatei bleibt unangetastet — sie geht verbatim nach `CYCLE_INFO` und wird byteweise geprüft. (18.09.2026) |

## 16 Luftstraßen

| ID | Regel |
|---|---|
| **AW-01** | **Die Routenkennung trägt ihren Suffix** (4.1.6.1 Note 1, Spalte 19): `A604` + `F` = `A604F`. Jedes Datenset, das die Kennung als **ein** Feld ausgibt, schreibt `route_identifier + suffix`. |
| **AW-02** | **Ausnahme:** hat das Zielformat eine **eigene Spalte** für den Suffix (DFD v2 `route_identifier_postfix`), bleibt `route_identifier` ohne ihn. (Anwenderkorrektur 04.10.2026) |
| **AW-03** | **Luftstraßen werden vor der Ausgabe immer nach Routenkennung, Suffix und Sequenznummer sortiert** — stabil, in **einer** Stelle (`EnrouteAirways.Sort`, angewendet von `FilteringExportSource`). Gilt für alle Export-Adapter, die heutigen und alle künftigen. (Anwendervorgabe 04.10.2026, 2.80.0) |
| **AW-04** | **Der Area Code ist kein Sortier- und kein Trennkriterium.** Eine Luftstraße über zwei Area Codes ist **eine** Route; ein Selbsttest verbietet jede Sortierung nach Area Code. |
| **AW-05** | Der Sortierschlüssel wird **gepolstert** verglichen (Kennung auf 5, Suffix auf 1 Zeichen), damit `G10 D` und `G103` nicht vertauschen. |

## 17 Prozeduren

| ID | Regel |
|---|---|
| **P-01** | **Circle-to-Land** (Anflug, dessen Kennung keine Bahn nennt — `VORA`, `RNVB`, `NDBA`): Kandidat ist `^[A-Z]+$` mit ≥ 3 Zeichen und einem MAP (Sp. 43 = `M`, Sp. 39 ∈ 0/1) an einem exportierten Flugplatz. Aufgelöst wird **je Bahn des Flugplatzes**; die Kennung bekommt die Form der Altstrecke (`V05-A`, `V05LA`, Mapping inkl. `CVOR`→`R`). Nennt der MAP-Fix eine Bahn (`RWxx`), gilt **nur diese**. |
| **P-02** | **`RWnnB`-Transition** (5.11, beide Parallelbahnen): aufgelöst **je physischer Bahn** des Flugplatzes; `RW04B` → `RW04C`, `RW04L`, `RW04R`, je nachdem, welche Bahnen der Platz hat. Die Kopien **ersetzen** das Original an seinem Platz. |
| **P-03** | **Eine Bahn, die die Prozedur schon als eigene Transition trägt, wird nicht zweimal erzeugt.** |
| **P-04** | **PD, PE und PF sind drei Recordtypen.** Die Kollisionsprüfung von P-03 arbeitet **je Subsection** — eine Transition, die im PD existiert, belegt im PE nichts. (Anwenderbefund 04.10.2026, 2.80.2; Beispiel HAAB IMKI1A: PD `RW07L`, PE `RW07B` → PE muss `07L` **und** `07R` liefern.) |
| **P-05** | Die Auflösung geschieht **ausschließlich im Export** über `BothRunwaysPlan`/`CircleToLandPlan`; der Merge expandiert nichts mehr (Merge 1.27.0, `BothRunwayTransitionExpander` entfernt). |
| **P-06** | Zähler sichtbar machen: `circleToLand.recordsReplaced/.recordsAdded/.unresolved`, `bothRunways.transitions/.expanded/.copies/.unresolved/.copiesSkippedExisting/.recordsReplaced/.recordsAdded`. |
| **P-07** | **Heliport-Anflug ohne Bahn:** die Heliport-Anflugkennung ist Typbuchstabe + **drei** Ziffern Endkurs + optionaler Zusatz (`R352`, `R002M`) — **kein `runway`, kein `designator`**; Zusatz = 5. Zeichen, Typ aus dem 1. Buchstaben (R→RNAV, P→GPS, N→NDB, D→VORDME, T/V/S→VOR). `VORA`/`VORB` fallen auf die Flugplatzregel zurück. |
| **P-08** | **Heli-SID:** bei einem Heliport bleiben die Common-Legs Common und der Pflicht-Container `RunwayTransitions` bleibt leer — der Transition-Bezeichner nennt ein **Pad** (`H`, `H1`), und `stRunwayNumber` kennt nur 0–36. |

## 18 Heliporte

| ID | Regel |
|---|---|
| **H-01** | `heliportsAsWaypoints` (Default **false**) schaltet die **gesamte** Heliport-Auslieferung. Es ist die **einzige** Ausnahme von `includeHelicopterData`: die Heliport-Tabellen werden an einer Stelle aus `FilteringExportSource.Inner` gelesen (`HeliportRepository.Tables`). |
| **H-02** | **Draußen bleiben immer:** HK (TAA), HS (MSA), die Helikopter-Legs der Flugplatzprozeduren und die EP-Holdings an Heliport-Fixen. **Ein Heliport ist kein Flugplatz des Datensets** — kein Eintrag in den Flugplatzlisten. |
| **H-03** | Ein Heliport wird als **`Airport` mit `Helipad`-Kindern** geschrieben (beide MSFS-Datensets ab 2.76.0), nicht als Waypoint. |
| **H-04** | **Identität = Kennung + ICAO-Code**; die Pads sind alle HA-Primary-Records dieses Paars in Cycle-Reihenfolge, Pad-Kennung Sp. 17–21 (`H`, `H1`, `H2`). Kopfwerte aus dem **ersten** Record. |
| **H-05** | **Layout-Regel:** die Heliport-Sektion ist mit der Flugplatz-Sektion **byteidentisch** (HA/PA-Supplemental, HC/PC, HV/PV, HD/PD, HE/PE, HF/PF) — die Element-Bauer des Flugplatzes lesen die H-Records unverändert, **kein neuer Parser**. Ausnahme ist nur der Primary (HA hat `PAD Identifier` 17–21). |
| **H-06** | Werte eines Pads: `lat`/`lon` = Bezugspunkt (33–51) · `surface` `A`→ASPHALT, `C`→CONCRETE, Default **CONCRETE** · `heading` Default **0.0** · `type` nur, wenn `stHelipad` ihn kennt, Default **NONE** · `length`/`width` aus Pad Dimensions 86–91 (`LLLWWW`), Breite `000` = runder Pad → Durchmesser = Breite, **ohne Länge kein Pad**. |
| **H-07** | `alt` = `pad_elevation` der Simulatorliste; **weicht sie um mehr als 1.000 ft** von der veröffentlichten Höhe (57–61) ab, gewinnt die **veröffentlichte**. `altType` = **`GEOID`**. Die Höhenkette wird **genau einmal je Pad** entschieden. |
| **H-08** | Paarung mit der Simulatorliste über die **Kennung allein** (der `icao_code` der Liste ist abgeleitet); Zeilen in Dateireihenfolge, der n-te Pad-Record nimmt die n-te Zeile. Die Helipad-Liste ist **optional** — fehlt sie, läuft der Lauf mit Warnung, jeder Wert hat einen Default. |
| **H-09** | **Verfahren aus:** `HeliportProcedures` ist in **beiden** Datensets aus — HD/HE/HF liefern keine `Departure`/`Arrival`/`Approach`, und ohne den Schalter werden die drei Tabellen nicht gelesen. `HeliportStarts` nur in `msfs2024` (`type="HELIPAD"`, `altType="GEOID"`, `number`/`designator` **leer**, **Position nicht versetzt**). |
| **H-10** | Kein `Ils`, kein `Ndb`, kein `Gls`/`PathPoint` an einem Heliport (es gibt keine HI-, HN-, HT-/HP-Sektion). **GLS-Sperre:** 186 Kennungen sind Heliport **und** Flugplatz — ohne Sperre bekäme der Heliport den GLS-Pfad des gleichnamigen Flugplatzes. Eigener Index für die SBAS-Stufen (`a424_hf_continuation_procedure_data`). |
| **H-11** | Die Heliport-Dateien liegen in einem **eigenen Szeneriebaum** `fs-base-heli/scenery/<ordner>/HTX<zelle>.xml` (gleiches Raster wie ATX), damit das Heliport-Paket einzeln ausgeliefert oder weggelassen werden kann und kein Fix zweimal im Paket steht. Dazu schreibt das Setup eine dritte Parameterdatei `navigraph-navdata-heli.xml`. |

## 19 Simulator-Dateien (MSFS / X-Plane)

| ID | Regel |
|---|---|
| **S-01** | Simulatorlisten werden **nach Position gelesen, nie nach Spaltennamen** (`BglData.Rows` überspringt Zeile 1). Eine Prüfung der Kopfzeile ist **absichtlich nicht** eingebaut; die gültigen Kopfzeilen stehen als Konstanten an einer Stelle (`BglData.AirportListHeader`, `ExcludeListHeader`, `GlideslopeTableHeader`) und die Selbsttests schreiben ihre Beispieldateien damit. |
| **S-02** | Beide MSFS-Datensets lesen **denselben** Ordner `external/` (`BglRun.DefaultSimulatorDataFolder = A424DataLayout.ExternalFolder`); `BoundariesCENTER.xml` liegt einmal. `simulatorDataFolder` bleibt als Ausnahme. |
| **S-03** | Querregeln des BGL-Compilers stehen in **keinem Schema** — eine Schemaprüfung gegen `bglcomp.xsd` findet sie nicht; sie kommen nur über Anwendermeldungen herein. |
| **S-04** | **X-Plane-Versionszeile fest verdrahtet, keine Einstellung** (gilt für jedes X-Plane-Datenset): Build-Datum = Tag des Laufs (`yyyyMMdd`, UTC aus `ExportContext.StartedUtc`), Copyright = `Copyright (c) <Jahr> Navigraph, Datasource Jeppesen` (`XPlaneHeader`). |
| **S-05** | Folge: **jede Datei mit Kopfzeile ist volatil** — bei `xplane11` die sechs Wurzeldateien **und alle 4.104 Kacheln**. CIFP-Dateien haben keine Versionszeile und werden byteweise verglichen. |

## 20 Setup

| ID | Regel |
|---|---|
| **SE-01** | `archiveName` ist der Name, unter dem das Archiv geschrieben wird (`dfdv2` → `dfdv2-2609.7z`), und gleichzeitig der Platzhalter `{archiveName}` in `folderName`, `fileName` und `folders`. Das alte `package` wird **ohne Warnung** weitergelesen; stehen beide im Eintrag, gewinnt `archiveName`. |
| **SE-02** | **Revision = Auslieferungszähler innerhalb eines Cycles** (1–99): in `revision` der `cycle.json`, in der Zeile `Version` der `cycle_info.txt` und in jedem `{revision}` — **nie im Archivnamen**. |
| **SE-03** | `setup.revision` darf die Revision des Export-Manifests überschreiben (Vorrang nach C-02). Eine Zahl **unter** der des Export-Laufs wird gebaut, aber **gewarnt**; gleich oder höher nur Info — dieses asymmetrische Muster ist Vorbild für künftige Einstellungen dieser Art. |
| **SE-04** | `folders` **verschiebt** nur; `exclude` **lässt weg**, und zwar **vor** den Layoutregeln: eine ausgelassene Datei wird nicht kopiert, nicht gezählt und erreicht keine `folders`-Regel. Der **Exportordner bleibt unberührt** — dasselbe Datenset kann in einem Lauf vollständig und beschnitten verpackt werden. |
| **SE-05** | Zwei Formen in `exclude`: **ohne** Platzhalter (`mobile`) = der Ordner **samt allem darunter** oder eine Datei dieses Namens; **mit** `*`/`?` = Mustermaschine, **überschreitet nie ein `/`**. |
| **SE-06** | Muster ohne Treffer → **Warnung, kein Abbruch**. `"*"` wird beim Lesen der Konfiguration abgelehnt, ein Mustersatz, der alle Dateien trifft, beim Bauen — **ein Paket ohne Dateien entsteht nicht**. |

## 21 Neuer Export-Adapter — Checkliste

| ID | Schritt |
|---|---|
| **N-01** | Ordner `adapters/A424Export/<Datenset>/` mit `src/`, `tests/`, `SPEC.md`; Datenset-Name klein mit Bindestrich. |
| **N-02** | `.slnf` in `datasets/` **und** Matrixeintrag in `.github/workflows/build.yml` (R-04, R-05). |
| **N-03** | Jede Abfrage auf eine Regeltabelle wählt **`raw_record`** mit (F-05) — sonst greifen `arincRules` und die Regelfilter nicht. |
| **N-04** | Keine eigene Rechnung für einen Wert, den `A424Export.Core` schon hat (V-01); keine eigene Flugplatz- oder Bahnprüfung (F-07, F-08). |
| **N-05** | Luftstraßen über `EnrouteAirways` ausgeben (AW-03); Suffix nach AW-01/AW-02 entscheiden. |
| **N-06** | Nichtdeterministische Felder als **volatil** deklarieren (F-11); kein Kommentarkopf (F-10). |
| **N-07** | `KnownSettings`, `RetiredSettings`, Filterbeachtung, `settingsFingerprint`-Beitrag und `--show-config`-Zeile ergänzen (C-04, C-06, F-15). |
| **N-08** | `SPEC.md` nach dem Muster der bestehenden Datensets: Format, Felder mit Spaltenlage, Werteregeln, Abweichungen vom Sample mit Begründung, Historie. |
| **N-09** | Selbsttests mit einer Fixture, die **byteexakt** aus dem echten Zyklus stammt; dazu je neue Regel ein Test, der sie für künftige Adapter festnagelt. |
| **N-10** | Für jede Einstellung mit Vier-Ebenen-Vorrang: Eintrag in `KnownProperties`/`KnownGroupProperties`/`KnownPackageProperties`, Wert mit Herkunft, Auflöser **und** manifestfreier Auflöser, Aufnahme in den Fingerabdruck, Warnung bei unbekanntem Namen, `--show-config`, Hilfetext, Log-Zeile, Statistikfeld. |

## 22 Doku, Tests, Lieferung

| ID | Regel |
|---|---|
| **T-01** | Jedes Datenset hat eine **`SPEC.md`**, jeder Adapter ein **`README.md`**, die Gesamtlösung die `docs/*.docx` (deutsch, Inhaltsverzeichnis, Seitenzahlen, jede H1 auf neuer Seite). Eine Regeländerung wird **in allen betroffenen Dokumenten gleichzeitig** nachgezogen. |
| **T-02** | Werkzeug für die docx: `docs/tools/mdinsert.py` fügt ein Markdown-Kapitel in den Konventionen des Dokuments ein (`--replace`, `--before`). |
| **T-03** | **Jede Regel bekommt einen Selbsttest**, der sie auch für künftige Adapter festhält; alle Testprojekte müssen grün sein, bevor geliefert wird. |
| **T-04** | Nachweis einer Lieferung: frisch entpacken, bauen, **alle** Tests laufen lassen, gegen den echten Zyklus vergleichen — nichtdeterministische Felder vorher normalisieren (F-11). |
| **T-05** | Lieferung = **Vollpaket + Delta-ZIP** gegen den letzten gelieferten Stand, dazu `AENDERUNGEN.txt` mit den Änderungen und den **von Hand zu löschenden** Dateien. |
| **T-06** | **Ein Delta-ZIP kann nicht löschen.** Eine Lieferung, die Dateien verschiebt oder entfernt, muss als **Vollpaket** eingespielt werden. |
| **T-07** | Ein Fehler, den das Sample nicht enthält, ist mit einem Referenzvergleich **nicht** zu finden — gemergte Daten brauchen Compiler- bzw. Schemaprüfung. |

## 23 Pflege des Regelwerks

* **Ablage:** `a424-general-rules/README.md` im Repository
  [`RichardStefan/a424-nupkgs`](https://github.com/RichardStefan/a424-nupkgs).
* **Wann nachgetragen wird:** bei **jeder** neuen Regel, jeder Änderung einer bestehenden, jedem neuen
  Export-Adapter, jeder Änderung an Filtern, Exit-Codes, Data-Root oder Vorrangketten — zusammen mit
  `SPEC.md`, `README.md`, `docs/*.docx` und dem Projekt-Dokument der Regel (T-01).
* **Wie:** Regel-ID behalten, Text ersetzen; eine gestrichene Regel wird **nicht gelöscht**, sondern mit
  `(entfallen <Version>)` markiert, damit alte Verweise lesbar bleiben. Neue Regeln bekommen die nächste
  freie Nummer ihres Abschnitts.
* **Kopf anpassen:** Stand und Versionszeile oben.

### Historie

| Datum | Änderung |
|---|---|
| 05.10.2026 | Erste Fassung. Stand Export 2.81.0, Merge 1.27.0, Import 1.16.0, Setup 1.19.0, Watermark 1.8.0, Shared 1.27.0. |
