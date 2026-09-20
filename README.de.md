# VSCplugin (VSCpy) – Technische Dokumentation

## Überblick

VSCplugin ist ein Total-Commander-Dateisystem-Plugin (WFX), das die
Volume Shadow Copies (Wiederherstellungspunkte) von Windows durchsuchbar
macht. Es zeigt sie als virtuelles Netzlaufwerk "VSCpy" an:

```
\\VSCpy\<Laufwerk>\<Zeitstempel-des-Snapshots>\<normale Ordnerstruktur>
```

Aus diesen Snapshots lassen sich Dateien schreibgeschützt herauskopieren
(z. B. um eine versehentlich überschriebene oder gelöschte Datei
wiederherzustellen), ohne dass ein vollständiges Systemwiederherstellen
nötig ist.

Zusätzlich zeigt eine optionale Spalte **"VSCStatus"** pro Datei an, ob sie
sich gegenüber dem vorherigen Snapshot geändert hat.

## Architektur

Das Plugin besteht aus einer einzigen DLL (`VSCPlugin.wfx64`), aufgeteilt
in folgende Quellmodule:

- `common/vss_enum.*` – Ermittelt die vorhandenen Shadow Copies per WMI
  (`Win32_ShadowCopy`, Namespace `ROOT\CIMV2`).
- `common/pathutil.*` – Übersetzt virtuelle TC-Pfade
  (`\<Laufwerk>\<Zeitstempel>\<Rest>`) in echte Gerätepfade
  (`\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\<Rest>`).
- `common/diff.*` – Die eigentliche Vergleichslogik (siehe unten).
- `common/debuglog.h` – Optionales, per INI zuschaltbares Debug-Logging.
- `wfx/vscplugin.cpp` – Die eigentliche WFX-Implementierung
  (`FsInitW`, `FsFindFirstW`/`FsFindNextW`/`FsFindClose`, `FsGetFileW`)
  sowie die Status-Spalte (`FsContentGetSupportedField`,
  `FsContentGetValueW`).

# 

## Wie die Diff-Logik genau funktiert

Die Vergleichslogik lebt in `common/diff.cpp`, Funktion `ComputeStatus`.

**Grundprinzip: Vergleich immer mit dem unmittelbar vorherigen (nächst-
älteren) Snapshot desselben Laufwerks – nicht mit einem festen "ersten"
oder "Original"-Snapshot.**

1. Alle Snapshots eines Laufwerks werden absteigend nach Erstellungsdatum
   sortiert (neuester zuerst) – `ListShadowCopiesForDrive`.
2. Die Position des gerade betrachteten Snapshots in dieser Liste wird
   ermittelt.
3. Der **Vorgänger** ist der nächste (also zeitlich davorliegende) Eintrag
   in dieser sortierten Liste.
4. Die Datei/der Ordner wird an derselben relativen Position im
   Vorgänger-Snapshot gesucht.

**Beispiel mit drei Snapshots A (ältester) → B → C (neuester):**

| Du betrachtest | Vergleich gegen                        |
| -------------- | -------------------------------------- |
| C              | B                                      |
| B              | A                                      |
| A              | – (kein Vorgänger, Spalte bleibt leer) |

**Bewertungskriterien** (nur für Dateien, nicht für Ordner):

- Datei fehlt im Vorgänger → **"Neu"**
- Datei existiert in beiden, aber Größe oder `LastWriteTime` unterscheiden
  sich → **"Geändert"**
- Größe und `LastWriteTime` identisch → **"Unverändert"**

Für **Ordner** wird nur die reine Existenz im Vorgänger geprüft (kein
rekursiver Inhaltsvergleich, aus Performance-Gründen) – ein vorhandener
Ordner gilt immer als "Unverändert", ein neuer Ordner als "Neu".

## Bekannte Einschränkung: keine 32-Bit-Unterstützung

Das Plugin gibt es bewusst **nur als 64-Bit-Version** (`VSCPlugin.wfx64`).

Grund: Die WMI-Klasse `Win32_ShadowCopy`, über die wir die vorhandenen
Snapshots ermitteln, ist von Microsoft offiziell als **nicht verfügbar
für 32-Bit-Anwendungen auf 64-Bit-Windows** dokumentiert. Ein 32-Bit-
Prozess bekommt dabei keinen Fehler, sondern still ein leeres
Ergebnis zurück – genau das haben wir beim Testen unter einer 32-Bit-
Total-Commander-Installation auch beobachtet (WMI-Abfrage erfolgreich,
aber 0 Treffer, obwohl `vssadmin list shadows` echte Snapshots zeigte).

Quelle: Microsoft Learn, *Win32_ShadowCopy class*, Abschnitt "Remarks":

> "This class is unavailable for 32-bit applications on Windows Server
> 2008 x64."

<https://learn.microsoft.com/en-us/previous-versions/windows/desktop/vsswmi/win32-shadowcopy>

Ein 64-Bit-Hilfsprozess, der die WMI-Abfrage stellvertretend für eine
32-Bit-Total-Commander-Installation ausführt, wäre technisch möglich
gewesen, wurde aber bewusst nicht umgesetzt (Komplexität/Nutzen-
Abwägung).

## Debug-Logging

Standardmäßig deaktiviert (kein Datenträgerzugriff). Zum Aktivieren:
Datei `VSCDebug.ini` im selben Ordner wie `VSCPlugin.wfx64` anlegen:

```ini
[Debug]
Enabled=1
```

Total Commander neu starten (die Einstellung wird nur beim Laden der
DLL geprüft). Das Log landet als `VSCPlugin_debug.log` im selben Ordner.

## Build

Cross-Compile mit MinGW-w64 (`x86_64-w64-mingw32-g++`), C++17.

Wichtige Linker-Flags:

```
-static-libgcc -static-libstdc++
```

Ohne diese Flags hängt die DLL dynamisch von `libgcc_s_*.dll` /
`libstdc++-6.dll` ab, die auf einem normalen Windows-System nicht
vorhanden sind – TC meldet dann "Plugin nicht erkannt" statt eines
klaren Ladefehlers.

Benötigte Bibliotheken beim Linken: `-lwbemuuid -lole32 -loleaut32`
(für die WMI-Abfrage).
