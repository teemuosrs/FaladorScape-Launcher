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
        +-- RSCMultiClient.jar        the dock window (opens on launch)
        +-- jnidispatch.dll           native helper - sits next to the jar
        +-- bot-core.zip              bot classes -> bot-core\ (embedded per client)
        +-- api/*.txt                 game definition lists
        +-- resources/...             fonts + sprites
        +-- Profiles/*.properties     starter profiles (installed only when absent)
        +-- Scripts/*.java            script sources (installed only when absent)
        +-- runtime.zip               bundled JRE (jlink) - no Java needed
        +-- FaladorScapeLauncher.exe  next launcher version (self-update)
        +-- Updater.exe               swap helper that replaces the running exe
```

## Release versioning

The project release version is a **single integer** (1, 2, 3, ...), stored in
**one authoritative place**: `launcher/RELEASE_VERSION.txt` (currently `1`).

- `build_game_dist.ps1` reads that file and writes `"releaseVersion": <int>`
  into `version.json`.
- The launcher writes the integer it installed into `<installDir>\version.txt`
  and compares it against the manifest's `releaseVersion` on every start.
- `UpdateGithub.bat` prompts "Current version: N, New version: M" (default
  N+1), confirms with [Y/N], updates `RELEASE_VERSION.txt`, then builds and
  publishes. No editing ten files.

The launcher's own executable identity stays the string `launcherVersion`
(the self-update string); the new integer is the USER-facing release version
(the "new version X available" prompt).

## On the friend's PC (first run)

1. Put `FaladorScapeLauncher.exe` anywhere - Desktop, Downloads, USB. It is
   location independent.
2. Run it once.
3. First run asks **where to install**:
   - `[Yes]` -> `Desktop\FaladorScape` (resolved via the OS Desktop folder,
     never a hardcoded `C:\Users\...` path),
   - `[No]`  -> choose a folder (FolderBrowserDialog; the directory is
     created if missing),
   - `[Cancel]` -> exit.
   The chosen directory is remembered in
   `%APPDATA%\FaladorScape\install-path.txt`, so a moved exe keeps working.
4. If no `launcher-config.json` exists next to the exe, the first run also
   asks for the update location
   (`Enter the update manifest URL (https://host/path/version.json)`) - paste
   the manifest URL once; it is saved to `launcher-config.json`, then the
   run continues normally. If `launcher-config.json` is already present the
   prompt is skipped.
5. It downloads the COMPLETE package into the chosen directory, verifies
   every SHA-256, extracts `runtime.zip` + `bot-core.zip`, writes
   `version.txt` with the installed release integer, and launches the bot.
6. Every later run = resolve install dir -> check manifest -> compare local
   `version.txt` vs `releaseVersion` -> download only changed files -> launch.

No admin rights are used. The install is fully contained in the chosen folder.

## Main launcher window

- **[Launch]** - start the bot (after the usual update/login flow).
- **[Check for updates]** - re-fetch the manifest and show the version prompt.
- **[Change location]** - re-run the install-location prompt and store the
  new path (files are then synced into the new folder).
- **[Reinstall]** - full download of the complete current remote version into
  the same install dir (protected files preserved).
- **[Exit]** - close.

Version prompts:
- Newer remote version: `New version X available. Download it? [Y/N]`
- Up to date: `You are running the latest version (N). Continue? [Y/N]`
- Declining an update launches the game with the current (still working)
  local files.

## Release workflow (YOU, after every bot update)

**One click:** double-click `UpdateGithub.bat` at the repo root. It:

0. reads the current release from `launcher/RELEASE_VERSION.txt`,
1. prompts `Current version: N, New version: M` (default N+1),
2. confirms with `[Y/N]`,
3. updates `launcher/RELEASE_VERSION.txt` (the ONE authoritative file),
4. rebuilds the launcher exes (`build_launcher.ps1` - failure is non-fatal),
5. rebuilds the whole distribution + `version.json` (`build_game_dist.ps1`,
   which now contains `releaseVersion`),
6. mirrors `launcher/web/` into `Desktop\FaladorScape-Launcher` (the clone of
   `teemuosrs/FaladorScape-Launcher`),
7. commits (message includes the release number) + pushes to GitHub,
8. commits a second PIN that rewrites `version.json`'s `baseUrl` to the
   pushed commit tree URL (CDN consistency) + pushes again,
9. verifies every manifest file is anonymously reachable on
   `raw.githubusercontent.com`,
10. syncs the freshly built bot classes into `Multiscreen\bot-core\bot`.

Or run the same steps manually:

```
# 1. Rebuild the launcher exes (only needed when Launcher.cs/Updater.cs changed)
powershell -NoProfile -ExecutionPolicy Bypass -File launcher\build_launcher.ps1

# 2. Build the game distribution + version.json (reads launcher\RELEASE_VERSION.txt)
powershell -NoProfile -ExecutionPolicy Bypass -File launcher\build_game_dist.ps1 -LauncherVersion "1.2.1"
```

`build_game_dist.ps1`:
1. Reads `launcher/RELEASE_VERSION.txt` and writes `releaseVersion` into
   `version.json`.
2. `mvn package` -> builds `FaladorScapeBot.jar` from `target/`.
3. Packages `FaladorScapeBot.jar`, `client.jar` (lib/RSCFalador-client-451.jar),
   `api/*.txt`, `resources/`, fresh `config-defaults/` + `bot/profiles/`.
4. `jlink` -> bundles a trimmed JRE (currently JDK 25) -> `runtime.zip`.
5. Computes SHA-256 for every file, writes `version.json`.
6. Verifies every manifest entry exists (including `releaseVersion`).

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
  read launcher-config.json (manifest URL, lives next to the exe)
  resolve INSTALL dir:
     1. --app-dir <dir>            (tests; never persisted)
     2. %APPDATA%\FaladorScape\install-path.txt  (if valid install)
     3. the exe folder itself      (legacy installs keep working)
     4. first-install prompt: Desktop\FaladorScape / Choose folder / Cancel
  fetch version.json
  read local release version from <installDir>\version.txt
  if released releaseVersion > local:
      prompt "New version X available. Download? [Y/N]"
  if equal:
      prompt "You are running the latest version (N). Continue? [Y/N]"
  if launcher version changed:
      stage new exes, start Updater.exe, exit
      Updater.exe verifies + swaps + relaunches --updated
  download + verify + atomically replace ONLY changed files
  extract runtime.zip + bot-core.zip when they changed
  write version.txt = releaseVersion
  write multiclient_config.json (absolute paths to the installed files)
  launch: <install>\runtime\bin\javaw.exe -Djna.boot.library.path=<install> \
          -Djna.nounpack=true -Dsun.java2d.uiScale=1 \
          -Dsun.java2d.uiScale.enabled=false -jar RSCMultiClient.jar
  RSCMultiClient opens the dock window; each client runs with the bot core
  embedded (java -cp bot-core;client.jar bot.bridge.Starter --port <N> ...)
```

Notable behaviour:
- **Location independence**: the install dir is stored in
  `%APPDATA%\FaladorScape\install-path.txt`, never next to the exe. If the
  stored path is missing/invalid, the launcher offers to reinstall / choose
  again.
- **Safe updates**: download -> verify SHA-256 -> extract to staging ->
  validate -> replace -> write new version -> relaunch. A failed
  download/extract/verify NEVER destroys the existing installation; the old
  version stays usable.
- **Protected files** (`multiclient_config.json`,
  `Profiles/*.properties`, `Scripts/*.java`) install only when absent;
  a user-modified copy is NEVER overwritten. Updates never delete profiles,
  accounts, config, settings or saved user data - only package files are
  replaced. Starter Profiles and Scripts arrive once and then belong to the
  friend.
- **Starter content**: every publish ships the owner's `Profiles/*.properties`
  (BotSettings only - NEVER passwords, which live in `config/accounts.txt`
  and stay on the owner's machine) and `Scripts/*.java` sources. The compiled
  script classes ride inside `bot-core.zip`; the sources ship for
  visibility/backup.
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
| `BuildInfo.cs` | Shared identity (launcher version, exe/file names, release-version file names) |
| `Launcher.cs`  | Launcher: install-dir resolution, manifest fetch, version compare, delta sync, verify, self-update handoff, launch |
| `Updater.cs`   | Swap helper that replaces running launcher exes |
| `build_launcher.ps1` | Compiles both exes with the in-box csc into `target\launcher\` |
| `build_game_dist.ps1` | Packages the game + bundles JRE + writes `version.json` (with `releaseVersion`) + `launcher/web/` |
| `RELEASE_VERSION.txt` | **Authoritative release version** - a single integer (1, 2, 3, ...) |
| `publish_to_github.ps1` | Reads `RELEASE_VERSION.txt`, builds, mirrors `launcher/web` to the GitHub clone, commits/pushes (release + pin), verifies raw access |
| `web/` | The generated update location - upload this whole folder |
