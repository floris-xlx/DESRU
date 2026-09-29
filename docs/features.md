# DESRU feature matrix

DOOM Eternal Speedrun Utility (`DESpeedrunUtil`) is a WinForms process that attaches to `DOOMEternalx64vk.exe` and edits cvars and format strings in that process. It does not inject a DLL. Rows below are the user-facing features. Implementation names point at the type that owns the behavior.

| Feature | What it does | Where | Depends on |
| --- | --- | --- | --- |
| Game hook | Finds `DOOMEternalx64vk.exe`, reads `ModuleMemorySize`, and maps it to a known build | `MainWindow` hook path, `GameVersion.GetVersionByModuleSize` | Game process running |
| Offset table | Loads per-version addresses from `Resources/offsets.json`. Unknown builds are signature-scanned and saved to `scannedOffsets.json` | `MemoryHandler.Initialize`, `SigScans` | Successful hook |
| On-screen display | Rewrites the game’s top-right performance metrics: version, flag letters (`M` `F` `R` `S` `L`), cheat / Meathook / restart status, scroll pattern, resolution scale | `MemoryHandler.MemoryTick`, `ModifyMetricRows` | Hook. See `docs/osd-overlay.md` |
| Minimal OSD | Packs the same status into the first two metric lines and sets metrics mode to `1` | `minimal` flag, `SetMetrics(1)` | OSD enabled |
| OSD font size | Writes `con_fontSize`. Also changes dev-console text size | `SetOSDFontSize`, Settings page | Hook. Optional |
| FPS limit | Writes `com_adaptiveTickMaxHz` to the Max FPS field (default 250) | `MemoryHandler.SetMaxHz` | Version has a `MaxHz` offset |
| Enforce 250 FPS | Keeps the cap applied. When off, the OSD shows `L` | `enforce250FPSCheckbox`, `limiter` flag | FPS limit offset |
| FPS hotkeys | Up to 15 global keys, each bound to an FPS value in `fpskeys.json` | `HotkeyHandler`, `FPSHotkeyMap` | Global hotkeys enabled, game hooked |
| Freescroll macro | Plays the bound up-scroll and down-scroll keys on a timer so a freescroll bind repeats | `Macro/FreescrollMacro.cs`, `macro/bindings.txt` | Macro checkbox and a bound key. OSD letter `M` |
| Resolution unlock | Lowers the minimum resolution scale from the game’s 50% floor toward 1% by rewriting `rs_minimumResolutionScale` or the 32-float scale table | `UnlockResScale`, `SetResScales` | Game past the intro (`rs_raiseMilliseconds` under 16). Alt-tabs to apply unless “Prevent automatic alt-tab” is set |
| Dynamic scaling | Sets `rs_enable` and the raise/drop frametimes from a target FPS (up to 1000) | `EnableDynamicScaling` | Unlock scheduled or already unlocked |
| Static scaling | Sets `rs_forceResolution` and turns dynamic scaling off | `EnableStaticScaling` | Same as dynamic. Game will not go much under ~10% in static mode |
| Auto enable scaling | Turns the chosen scaling mode on when the unlock finishes | `ScheduleResUnlock(auto, …)` | Unlock on startup or the Unlock button |
| Res. scale hotkeys | One key toggles scaling. Four keys set a minimum percent | `HotkeyHandler` res-scale keys | Hotkeys enabled, unlock finished |
| Cheats toggle | Flips the console-cheats and binds-cheats bytes so the dev console can run cheat commands | `EnableCheats`, cheat offsets in `MemoryHandler.Initialize` | Known cheat signature for the build. OSD line `CHEATS ENABLED` |
| Meathook detect | Treats `XINPUT1_3.dll` next to the exe as Meathook. Install and uninstall copy or remove that DLL | `MainWindow.CheckForMeathook`, `InstallMeathook` | Game directory known. OSD line `MEATH00K` |
| Reset-run hotkey | Double-tap (500 ms) runs the keyboard script that resets a run, then shows `RUN RESET` on the OSD for about 5 seconds | `HotkeyHandler`, `MemoryHandler` reset script | Setting “Enable Reset Run / Kill Script Hotkey”. Cheats may be turned on for the script and restored after |
| Firewall rule | Adds or removes a Windows Firewall rule for the game exe so leaderboard traffic can be blocked | `Firewall/FirewallHandler.cs` | OSD letter `F` while the rule exists. A change asks for a game restart |
| Version swap | Switches the installed game to another build from the version list and can download the downpatcher | `changeVersionButton`, `downpatcherButton`, `gameVersions.json` | Game not left in a bad hook. “Version Swapped” is shown after a change |
| profile.bin replace | Overwrites the Steam profile (`782330/remote/PROFILE/profile.bin`) on patch 3.1 | `replaceProfileCheckbox`, `UserdataDialog` | Patch 3.1 only |
| ReShade detect | Checks for ReShade beside the game at hook time | `MainWindow` hook, `reshade` flag | OSD letter `R`. Detection only; DESRU does not load ReShade |
| Slopeboost flag | On `1.0 (Release)`, reads the ramp-jump byte and shows `S` when slopeboost is disabled | `MemoryTick` | That build only |
| Modded-client flag | Marks UWP / modded clients and shows `MODDED CLIENT` or `(MOD)` | `modded` flag at hook | OSD |
| Restart-required flag | If Cheat Engine or `DoomEternalTrainer` is seen, shows `RESTART GAME` and stops trusting the session. An external trainer also freezes further OSD writes | `RestartTick` (every 2.5 s) | DESRU must be running before the game for a legal run |
| Out-of-date mark | Appends `*` to the FPS line when an update was detected | `outofdate` flag, `Program.UpdateDetected` | Update check |
| Trainer readout | With cheats or Meathook, reads position, yaw, pitch, and velocity into the form, or into the metrics block when “Show In-Game Display” is on | `ReadTrainerValues`, trainer branch of `MemoryTick` | Position pointer non-zero for the build |
| Speedometer | Form panel showing horizontal m/s (`sqrt(vx²+vy²)`), plus total or vertical. Works without cheats | `MainWindow.Speedometer` | Hook. See `docs/speedometer.md` |
| Scroll pattern | Counts a mouse-wheel burst and shows `count (avg ms)` on the OSD for 2 seconds | Form timer, `SetScrollPatternString` | OSD enabled |
| Info panel | Live status: game, FPS cap, macro, slopeboost, balance updates, cheats, ReShade, dynamic scaling, hotkeys, RTSS | `MainWindow.StatusTick` | — |
| CVARs | Disable anti-aliasing (no effect under DLSS), disable the Ultra-Nightmare quit-out delay, auto-continue through loading screens | Settings page, `SetCVAR`, written in `MemoryTick` | Offsets for that build |
| Console hotkey | Moves the dev console bind to Ctrl+Shift+~ so nearby keys do not open it, and lets you pick the console key | Settings page, `con_AdvancedKeypress` | Hook |
| Launch RTSS | Starts RivaTuner Statistics Server with DESRU when it is not already running | `StatusTick`, `launchRTSSCheckbox` | RTSS installed |
| Self-update | `Updater` downloads a new DESRU build and relaunches it. The in-game `*` means the running copy is behind | `Updater/Program.cs`, `UpdateDialog` | Network |
| Global hotkeys | Raw input so the FPS, res-scale, and reset keys work while the game is focused | `HotkeyHandler`, `Enable Global Hotkeys` | — |

## OSD letters

These are the only flags printed in parentheses on the version line. Detail is in `docs/osd-overlay.md`.

| Letter | Means |
| --- | --- |
| `M` | Freescroll macro is enabled and has a key bound |
| `F` | Firewall rule is present for the game exe |
| `R` | ReShade was found next to the game |
| `S` | Slopeboost is disabled (`1.0 (Release)` only) |
| `L` | The 250 FPS cap is not being enforced |

`*` on the FPS counter is separate: DESRU itself is out of date.
