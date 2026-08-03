# FaladorScape Self-Updating Launcher

One download. Everything else updates itself.

Your friend downloads **`FaladorScapeLauncher.exe`** once, runs it, and from
then on every bot/client/library/resource update you publish is fetched
automatically. No Java install, no re-downloads, no source code.

```
FaladorScapeLauncher.exe   <-- the ONE file your friend downloads
        |
        | checks https://your-host/.../version.json every run
        v
Update location (launcher/web/)
        +-- version.json              manifest: versions, file list, SHA-256
        +-- FaladorScapeBot.jar       the bot
        +-- client.jar                the game client
        +-- api/*.txt                 game definition lists
        +-- resources/...             fonts + sprites
        +-- config-defaults/...       installed only if absent (user config wins)
        +-- bot/profiles/...          preinstalled profiles (same rule)
        +-- runtime.zip               bundled JRE (jlink) - no Java needed
        +-- FaladorScapeLauncher.exe  next launcher version (self-update)
        +-- Updater.exe               swap helper that replaces the running exe
```

## On the friend's PC (first run)

1. Put `FaladorScapeLauncher.exe` in an empty folder.
2. Run it once.
3. First run asks for the update location
   (`Enter the update manifest URL (https://host/path/version.json)`) - paste
   the manifest URL once, it is saved to `launcher-config.json`, then the
   run continues normally. If `launcher-config.json` is already present the
   prompt is skipped.
4. It downloads everything, verifies every SHA-256, launches the bot.
5. Every later run = check manifest -> download only changed files -> launch.

No admin rights are used. The install is fully contained in that folder.


## How a run works

```
START
  read launcher-config.json (manifest URL)
  fetch version.json
  compare local files against SHA-256 + sizes
  if launcher version changed:
      stage new exes, start Updater.exe, exit
      Updater.exe verifies + swaps + relaunches --updated
  download + verify + atomically replace ONLY changed files
  extract runtime.zip when the bundled JRE changed
  launch: <runtime>\bin\java.exe -cp FaladorScapeBot.jar;client.jar;. bot.ui.DashboardFrame
```

Notable behaviour:
- **Protected files** (`config-defaults/...`, `bot/profiles/...`) install only
  when absent; a user-modified copy is NEVER overwritten.
- **Every download is verified** against the manifest SHA-256 before install;
  a mismatch aborts that file and reports it.
- **Atomic installs**: the old file is moved aside, the new one moved in, the
  backup deleted. Locked files are retried with backoff.
- Any failed update leaves the previous install intact; rerun to retry.

## Build requirements (only on YOUR machine)

- Maven (`mvn`) for `mvn package`.
- A JDK with `jlink` (this repo's build uses `C:\Program Files\Java\jdk-25.0.2`).
- The in-box Windows .NET Framework C# compiler (`csc.exe`) - NOT needed on
  the friend's PC.

## Files

| File | Purpose |
|------|---------|
| `BuildInfo.cs` | Shared identity (launcher version, exe/file names) |
| `Launcher.cs`  | Launcher: manifest fetch, delta sync, verify, self-update handoff, launch |
| `Updater.cs`   | Swap helper that replaces running launcher exes |
| `build_launcher.ps1` | Compiles both exes with the in-box csc |
| `build_game_dist.ps1` | Packages the game + bundles JRE + writes `version.json` + `launcher/web/` |
| `web/` | The generated update location - upload this whole folder |
