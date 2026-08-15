# Fast loading

This is a performance-focused fork of ES-DE (EmulationStation Desktop Edition)
whose goal is to make startup and browsing of **large collections** substantially
faster than the official build. The changes so far are all in the collection
loading path: the code that walks your ROM folders, matches file extensions,
parses `gamelist.xml` and sorts the resulting lists.

## What was changed and why

The official loader spends most of its time on redundant work per file, not on
actual work that needs to happen. The changes below remove that redundancy
without changing any behavior.

### 1. Directory scanning no longer re-stats every file

Previously `SystemData::populateFolder()` called
`Utils::FileSystem::getDirContent()` (which returns only paths), and then for
**every** entry called `isDirectory()` and `isSymlink()`, each of which is a
`stat()`/`lstat()` syscall through `std::filesystem`. That is two extra syscalls
per file — hundreds of thousands of syscalls for a large library.

A new `Utils::FileSystem::getDirContentEntries()` returns a `DirEntry` with the
entry type already filled in from the directory iterator (`directory_entry`
caches the `d_type` hint from `readdir()` on POSIX, so no extra syscall is
needed). `populateFolder()` now uses those cached types, only falling back to a
single status lookup for symlinks (to preserve the existing follow-symlink
semantics). The common case drops from ~3 stat calls per file to zero.

### 2. Extension matching is now O(1)

For each file the loader did a linear `std::find()` over the system's extension
list (`mSearchExtensions`). A precomputed `mSearchExtensionsSet`
(`std::unordered_set`) is now stored alongside the list, and both the directory
scanner and the gamelist parser use a hash lookup. Ordering of matches is not
used anywhere, so behavior is identical.

### 3. Name sorting no longer re-uppercases on every comparison

`FileSorts::compareName()` built a fresh uppercased copy of the name/sortname
metadata on **every** comparator invocation. `std::stable_sort` is O(n log n)
comparisons, so a 100k-game list triggered hundreds of thousands of uppercase
string allocations.

`FileData` now caches the resolved, uppercased sort key and invalidates it by
comparing the underlying metadata values. `compareName()` and
`compareNameDescending()` just compare the cached keys, producing the exact same
ordering at a fraction of the cost.

### 4. Gamelist parsing no longer stats every game (the big one)

The single largest win turned out to be hiding in plain sight:
`Utils::FileSystem::getStem()` called `isDirectory()` — a filesystem syscall —
on **every** invocation, and the `FileData` constructor calls `getStem()` for
every game (to derive the display name), as do the arcade-asset checks. On a
library of 115k games this means hundreds of thousands of extra `stat()` calls
against the ROM directory — several tens of seconds when the ROMs live on a
spinning disk or USB drive. `resolveRelativePath()` and
`removeCommonPath()` likewise re-stat'ed the same base directory once per
entry, and `MetaDataList` allocated one `std::map` node per metadata field (24
per game) even for fields that were never set.

Fixes:

- Added stat-free `getStem(path, isDirectory)` and
  `removeCommonPath(path, common, contains, commonIsDirectory)` overloads; the
  hot paths pass the already-known type so no syscall is made.
- `FileData` derives the directory status from its `FileType` instead of
  stat'ing the path.
- `GamelistFileParser` resolves the base directory once per system and skips
  the hidden-file attribute checks in gamelist-only mode (`ParseGamelistOnly`),
  where the gamelist is trusted and the syscalls are pure waste.
- `MetaDataList` no longer stores default values: `get()` falls back to the
  declared default lazily, cutting per-game allocations from ~24 map insertions
  to a handful. `get()` also uses a single lookup instead of `count()+at()`.

Measured on a 115,704-entry, 123-system collection with the ROMs on a USB HDD
(`ParseGamelistOnly` enabled): MAME's 40,309-entry gamelist parses in ~0.5 s
instead of ~40 s, and total gamelist parsing dropped from **86.6 s of CPU to
~1.2 s**.

### 5. Systems are populated in parallel

`SystemData::loadConfig()` previously built every system one after another on a
single core. Each system (directory scan + `gamelist.xml` parse + sort + filter
index) is fully independent, so they are now built across
`std::thread::hardware_concurrency()` worker threads. Theme loading is
deliberately kept on the main thread because it mutates shared
`ResourceManager` font state, and the splash screen / SDL event loop stay
responsive while the workers run. This is the single biggest wall-clock win for
machines with many systems.

## Measuring startup time

Pass `--profile-load` to print a per-phase breakdown to the console and to
`es_log.txt`:

```
--- Startup profile (--profile-load) ---
Theme scanning:       ...
System loading:       ...
  Directory scan:     ...
  Gamelist parsing:   ...
  Sorting:            ...
  Filter indexing:    ...
  Theme loading:      ...
View preload:         ...
Total (themes+systems+views): ...
```

This is what should drive further work: fix whichever phase is actually largest
rather than guessing.

## Measured results (115,704 games / 123 systems, USB HDD)

| Phase | Official build | This fork | Speedup |
|---|---|---|---|
| Gamelist parsing (CPU) | 86,634 ms | ~1,200 ms | **~72x** |
| System loading (wall) | 40,302 ms | ~3,300 ms | **~12x** |
| View preload (default) | 30,632 ms | ~14,500 ms | ~2x |
| **Startup, default** | **70,941 ms** | **~17,500 ms** | **~4x** |
| **Startup, `--no-preload`** | 70,941 ms | **~9,300 ms** | **~7.6x** |

Runs shown are from this machine; the official numbers are the same build
before the changes above. The remaining startup time is split roughly evenly
between system loading (gamelist parse + sort + filter index, parallel) and the
main-thread view preload, which is dominated by building theme elements for the
system-select screen.

## Skipping the gamelist-view preload (`--no-preload`)

During startup ES-DE eagerly builds the gamelist view for **every** system so
there is no texture pop-in when switching between them. With large collections
this is several seconds of main-thread work (theme components for 100+ systems,
plus list population for the biggest systems).

This fork adds a persistent setting **`PreloadGamelistViews`** (default `true`,
upstream behavior) and a `--no-preload` command-line flag. When disabled, the
gamelist views are built lazily on first visit instead of during startup:

```bat
ES-DE.exe --no-preload
```

or add this to `es_settings.xml`:

```xml
<bool name="PreloadGamelistViews" value="false" />
```

The trade-off is some texture pop-in the first time you enter a system. On the
reference collection this saves ~10 s of startup. The system-select view itself
is still built at startup (it is the first screen) and remains the largest
remaining fixed cost on slow storage.

## Expected impact

These changes primarily speed up **initial system population** (the directory
scan), **gamelist parsing** (extension checks) and **list sorting**, and let
that work scale across CPU cores. The win is largest for:

- Collections with many systems (parallel loading), with the biggest gains on
  multi-core machines.
- Very large flat ROM directories (hundreds of thousands of files), where the
  syscall reduction dominates.
- Large flat gamelists being sorted by name.

They do not change what gets loaded or how it is ordered — they only make the
existing logic do less redundant work, and do it concurrently.

## Building

Builds exactly like upstream ES-DE. Windows requires MSVC (MinGW is no longer
supported upstream); Linux/macOS use system package-manager dependencies. Full
instructions are in [INSTALL-DEV.md](INSTALL-DEV.md).

Verified on this tree with MSVC 2022 (toolset 14.44) and Windows 11 SDK
10.0.26100:

```bat
:: From the "x64 Native Tools Command Prompt for VS 2022":
tools\Windows_dependencies_setup.bat
tools\Windows_dependencies_build.bat

:: Configure and build. Ninja is used here instead of NMake/JOM for parallel builds:
cmake -G Ninja -DCMAKE_BUILD_TYPE=Release .
cmake --build . -j 12
```

Notes:

- The dependency setup script ends by trying to install OpenSSL 4 (interactive,
  requires admin). OpenSSL is **not** required to build or link ES-DE on Windows
  (libcurl uses Schannel and libgit2 uses WinHTTP), so that step can be skipped;
  install it only if you want the optional runtime DLLs copied next to the
  executable.
- Ninja build artifacts are gitignored; the in-source build directory is the
  repository root.

## Application updater

The in-app updater originally checked es-de.org for official ES-DE releases.
It now checks this fork's own `latest_release.json` (served from the root of the
`beelzebud/ES-DE-Turbo` GitHub repository), so fork users are never offered the
official build.

To cut a new release:

1. Bump `PROGRAM_VERSION_*` and `PROGRAM_RELEASE_NUMBER` in
   `es-core/src/ApplicationVersion.h`. The release number **must increase**,
   because the updater only reports a newer version when the feed's release
   number is greater than the running build's.
2. Update `latest_release.json` in the repo root: set `stable.version`,
   `stable.release` and `stable.date`, and point the `WindowsPortable` package
   at the new GitHub release asset
   (`https://github.com/beelzebud/ES-DE-Turbo/releases/download/<tag>/<asset>`)
   with its MD5.
3. Tag and publish the release on GitHub. The file at the root of the default
   branch is what the running application downloads.

The updater can still be disabled entirely, either per-user (`CHECK FOR
APPLICATION UPDATES` → `NEVER` in the menu, which sets
`ApplicationUpdaterFrequency` to `never`) or at build time with
`cmake -DAPPLICATION_UPDATER=off`.

## Remaining costs and roadmap

On the reference collection the ~9 s (with `--no-preload`) breaks down as:

- **System loading ~3.3 s** — already parallel across cores; split between
  gamelist parsing (~1.2 s), sorting (~1 s) and filter-index building (~2.3 s
  CPU). Further wins here need intra-system parallelism or a binary cache.
- **System-select view ~4.3 s** (warm cache, more when cold) — building theme
  elements (backgrounds, videos, texts) for all 103 systems, mostly reading
  theme media from disk on the GL-bound main thread. Lazy per-system image
  loading (only load the visible carousel entries) is the next target.

Higher-value, higher-risk ideas further out:

- **Caching the game list / gamelist index on disk** so unchanged systems can be
  re-hydrated without re-scanning or re-parsing XML. Keyed on a directory
  fingerprint (mtime + entry count) and stored as a flat binary snapshot, not a
  database.
- **Intra-system parallel gamelist parsing** for single gigantic systems
  (split the XML node list across workers; needs care with shared tree
  mutation).
- **Lazy media and theme resolution**: defer image/video path lookups until a
  game is actually shown rather than during list construction.
- Keep profiling with `--profile-load` (plus the per-system
  `GamelistFileParser::parseGamelist(): Parsed N entries ...` and
  `ViewController::preload(): ... built in ... ms` log lines) and target
  whichever phase dominates on your hardware.
