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

## Release workflow (YOU, after every bot update)

Run these two commands from the repo root:

```
# 1. Rebuild the launcher exes (only needed when Launcher.cs/Updater.cs changed)
powershell -NoProfile -ExecutionPolicy Bypass -File launcher\build_launcher.ps1

# 2. Build the game distribution + version.json
powershell -NoProfile -ExecutionPolicy Bypass -File launcher\build_game_dist.ps1
```

`build_game_dist.ps1`:
1. `mvn package` -> builds `FaladorScapeBot.jar` from `target/`.
2. Packages `FaladorScapeBot.jar`, `client.jar` (lib/RSCFalador-client-451.jar),
   `api/*.txt`, `resources/`, fresh `config-defaults/` + `bot/profiles/`.
3. `jlink` -> bundles a trimmed JRE (currently JDK 25) -> `runtime.zip`.
4. Computes SHA-256 for every file, writes `version.json`.
5. Verifies every manifest entry exists.

### Upload

Upload the ENTIRE **`launcher/web/`** folder to your web host (any static
host works - GitHub Pages, Netlify, your own domain, Dropbox public link...).

HTTPS is strongly recommended. The launcher supports plain HTTP too, but
your friend's account credentials travel through the game client's own
connection - keep the transport for the app files at least HTTPS.

### Point the launcher at your host

The launcher does not ship with a baked-in URL. Your friend (or you, before
sending it) points it at your manifest once:

```
FaladorScapeLauncher.exe --set-url https://YOUR-HOST/path/to/web/version.json
```

That writes `launcher-config.json` next to the exe. Alternatively ship a
`launcher-config.json` in the same folder:

```json
{ "manifestUrl": "https://YOUR-HOST/path/to/web/version.json" }
```

The address can be a folder (`https://host/falador/`) or the full
`version.json` URL - both work.

## Publishing a NEW LAUNCHER (self-update)

1. Bump `launcherVersion` in `BuildInfo.cs` (and the default in
   `build_game_dist.ps1` to match).
2. `build_launcher.ps1`, then `build_game_dist.ps1`.
3. Upload `launcher/web/` (the new exes are in the manifest with
   `"launcher": true`).
4. Your friend's next run downloads the new exes, `Updater.exe` swaps the
   running ones, and the new launcher relaunches. Zero interaction.

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
