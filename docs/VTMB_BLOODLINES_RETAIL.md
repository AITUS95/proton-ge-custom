# Vampire: The Masquerade - Bloodlines (retail, no patches)

The retail release, run without the unofficial patch, exits at startup with:

```
Engine Error
Available memory less than 15MB!!! -1
```

## Cause

`engine.dll` calls `GlobalMemoryStatus()` and compares `dwTotalPhys` with
15 MB as a **signed** 32-bit int. With more than 4 GB of RAM Wine reports
`0xFFFFFFFF`, which reads as `-1`. The engine then uses half of that value,
clamped to 80-200 MB, as its heap, so reporting less RAM costs nothing.

## Fix

`patches/protonfixes/0007-gamefix-vtmb-bloodlines-retail-memory-check.patch`
adds `exe_fixes.py` to protonfixes. It is called from the Steam and umu
default fixes and matches fixes by the launched executable name, because a
retail game added as a non-Steam shortcut has a random app id and a umu game
without a database entry runs as `umu-0`.

For `vampire.exe` (skipped for the Steam release, app id 2600) it sets:

- `WINE_RAM_REPORTING_BIAS` = total RAM in MiB - 1536, so the game sees
  1.5 GB of RAM (29715 on a 31251 MiB machine). An existing value is kept.
- `version=n,b`, so the SafeDisc `version.dll` loader next to `vampire.exe`
  is loaded instead of Wine's builtin.

It is the same as adding these launch options in Steam:

```
WINE_RAM_REPORTING_BIAS=29715 WINEDLLOVERRIDES="version=n,b" %command%
```

Do not disable DXVK's `d3d9.memoryTrackTest` for this game: its built-in
profile limits video memory to 1024 MB, and without it the game keeps
allocating textures until tens of GB are in use.
