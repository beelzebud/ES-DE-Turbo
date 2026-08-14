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

## Expected impact

These three changes primarily speed up **initial system population** (the
directory scan), **gamelist parsing** (extension checks) and **list sorting**.
The win is largest for:

- Collections with many systems and/or very large flat ROM directories
  (hundreds of thousands of files), where the syscall reduction dominates.
- Large flat gamelists being sorted by name.

They do not change what gets loaded or how it is ordered — they only make the
existing logic do less redundant work.

## Building

Build exactly as the upstream project; the only touched translation units are
`es-core` and `es-app` sources that are already part of the normal build.

```bash
cmake --preset release
cmake --build --preset release
```

See [INSTALL.md](INSTALL.md) for per-platform instructions.

## Roadmap for further speedups

These are the next highest-value, higher-risk items (kept out of this first
pass because they change execution flow, not just per-file cost):

- **Parallel system loading** across CPU cores. Each `SystemData` constructor
  (directory scan + `gamelist.xml` parse + sort) is independent, and this is the
  single biggest wall-clock win for machines with hundreds of systems. Needs
  careful thread-safety review of `Settings`, `Log`, `ResourceManager` and the
  splash-screen rendering, plus a main-thread-only event/theme phase.
- **Caching the game list / gamelist index on disk** so unchanged systems can be
  re-hydrated without re-scanning or re-parsing XML.
- **Lazy media and theme resolution**: defer image/video path lookups until a
  game is actually shown rather than during list construction.
- **Instrumented startup profiling** (a `--profile-load` mode that prints
  per-phase timings) so future changes are driven by numbers, not guesses.
