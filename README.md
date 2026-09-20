# VSCplugin (VSCpy) – Technical Documentation

## Overview

VSCplugin is a Total Commander file system plugin (WFX) that makes
Windows Volume Shadow Copies (restore points) browsable. It shows them
as a virtual network drive called "VSCpy":

```
\\VSCpy\<drive>\<snapshot-timestamp>\<normal folder structure>
```

Files can be copied out of these snapshots read-only (e.g. to recover an
accidentally overwritten or deleted file) without doing a full system
restore.

An optional **"VSCStatus"** column additionally shows, per file, whether
it changed compared to the previous snapshot.

## Architecture

The plugin is a single DLL (`VSCPlugin.wfx64`), split into these source
modules:

- `common/vss_enum.*` – Enumerates existing shadow copies via WMI
  (`Win32_ShadowCopy`, namespace `ROOT\CIMV2`).
- `common/pathutil.*` – Translates virtual TC paths
  (`\<drive>\<timestamp>\<rest>`) into real device paths
  (`\\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\<rest>`).
- `common/diff.*` – The actual comparison logic (see below).
- `common/debuglog.h` – Optional, ini-toggleable debug logging.
- `wfx/vscplugin.cpp` – The actual WFX implementation
  (`FsInitW`, `FsFindFirstW`/`FsFindNextW`/`FsFindClose`, `FsGetFileW`)
  plus the status column (`FsContentGetSupportedField`,
  `FsContentGetValueW`).

# 

## How the diff logic actually works

The comparison logic lives in `common/diff.cpp`, function
`ComputeStatus`.

**Core principle: always compare against the immediately preceding
(next-older) snapshot of the same drive - never against a fixed
"first" or "original" snapshot.**

1. All snapshots of a drive are sorted newest-first -
   `ListShadowCopiesForDrive`.
2. The position of the currently viewed snapshot in that list is found.
3. The **baseline** is the next entry in that sorted list (i.e. the one
   immediately older in time).
4. The file/folder is looked up at the same relative path in the
   baseline snapshot.

**Example with three snapshots A (oldest) → B → C (newest):**

| You're browsing | Compared against                    |
| --------------- | ----------------------------------- |
| C               | B                                   |
| B               | A                                   |
| A               | – (no baseline, column stays empty) |

**Evaluation criteria** (files only, not folders):

- File missing in the baseline → **"New"**
- File exists in both, but size or `LastWriteTime` differ →
  **"Modified"**
- Size and `LastWriteTime` identical → **"Unchanged"**

For **folders**, only plain existence in the baseline is checked (no
recursive content comparison, for performance reasons) - an existing
folder always counts as "Unchanged", a new one as "New".

## Known limitation: no 32-bit support

The plugin is deliberately shipped **64-bit only** (`VSCPlugin.wfx64`).

Reason: the WMI class `Win32_ShadowCopy`, which we use to enumerate
existing snapshots, is officially documented by Microsoft as
**unavailable to 32-bit applications on 64-bit Windows**. A 32-bit
process gets no error, just a silent empty result - exactly what we
observed testing against a real 32-bit Total Commander install (the WMI
query succeeded but returned 0 hits, even though `vssadmin list shadows`
showed real snapshots).

Source: Microsoft Learn, *Win32_ShadowCopy class*, "Remarks" section:

> "This class is unavailable for 32-bit applications on Windows Server
> 2008 x64."

<https://learn.microsoft.com/en-us/previous-versions/windows/desktop/vsswmi/win32-shadowcopy>

A 64-bit helper process that performs the WMI query on behalf of a
32-bit Total Commander install would have been technically possible, but
was deliberately not implemented (complexity/benefit tradeoff).

## Debug logging

Disabled by default (zero disk I/O). To enable, create a file named
`VSCDebug.ini` in the same folder as `VSCPlugin.wfx64`:

```ini
[Debug]
Enabled=1
```

Restart Total Commander (the setting is only checked when the DLL
loads). The log is written as `VSCPlugin_debug.log` in that same folder.

## Build

Cross-compiled with MinGW-w64 (`x86_64-w64-mingw32-g++`), C++17.

Important linker flags:

```
-static-libgcc -static-libstdc++
```

Without these, the DLL dynamically depends on `libgcc_s_*.dll` /
`libstdc++-6.dll`, which a normal Windows system doesn't have - TC then
reports "plugin not recognized" instead of a clear load error.

Libraries needed at link time: `-lwbemuuid -lole32 -loleaut32` (for the
WMI query).
