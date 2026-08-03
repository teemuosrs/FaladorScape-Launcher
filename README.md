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
3. It downloads everything, verifies every SHA-256, launches the bot.
4. Every later run = check manifest -> download only changed files -> launch.

No admin rights are used. The install is fully contained in that folder.
