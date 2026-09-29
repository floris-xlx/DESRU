# In-game overlay (top right)

The patch in the top right is DOOM Eternal’s own performance-metrics block. DESRU does not inject a DLL, create a layered window, or draw pixels. After it has a handle to `DOOMEternalx64vk.exe`, it overwrites the C format strings that block already prints, and writes the metrics-mode byte so the engine keeps the block on screen.

The game still owns the draw. Each frame it reads those strings and `sprintf`s them. DESRU only changes the bytes, on a timer, for as long as it stays hooked.

## How the process is attached

`MainWindow` finds `DOOMEternalx64vk.exe` and constructs `MemoryHandler` with that `Process`. That is the “hook”: open the process, resolve addresses, start a WinForms timer. There is no `LoadLibrary` into the game and no remote thread for the overlay.

`MemoryHandler` stores `MainModule.ModuleMemorySize` and looks the build up in `GameVersion.GetVersionByModuleSize`. `Initialize()` then:

1. Picks `KnownOffsets` whose `Version` matches (`SetCurrentKnownOffsets`), from the list loaded out of `DESpeedrunUtil/Resources/offsets.json`.
2. If the version is missing, runs `SigScans()` and appends the result to `scannedOffsets.json`.
3. Wraps each integer offset in a `DeepPointer` aimed at module `DOOMEternalx64vk.exe`.

The memory timer is created in the constructor:

- Interval: `Program.TimerInterval`, default **16 ms** (`Program.cs`). A command-line interval overrides it.
- Tick: `MemoryTick()`.

`MemoryTick` starts with `DerefPointers()`, which turns each `DeepPointer` into an `IntPtr` in the game. For the metric rows the pointer has no extra offsets, so the address is `module base + offsets.json["Row1"]` (same for `Row6`, `GPUVendor`, `GPUName`, `Metrics`, `MetricsFontSize`). The CPU slot is different: `CreateDP(CPU, 0x0)` follows one pointer, so `_cpuPtr` is whatever that address points at.

Writes go through `Process.WriteBytes` in `Memory/ProcessExtensions.cs`, which is `WriteProcessMemory` on the game handle. Before the string writes, `ModifyMetricRows` calls `VirtualProtectEx` on 1024 bytes at the row (and at the CPU / GPU strings) with `PAGE_READWRITE`, because those format strings live in a page that is not writable by default. The old protection is not restored.

## Finding the strings

`SigScans` runs only when `offsets.json` has no entry for this module size. It scans `MainModule` from base to `ModuleMemorySize`.

FPS anchor, ASCII for `"%i FPS\0%.2fms\0Frame : %u"`:

```text
25 69 20 46 50 53 00 00 25 2E 32 66 6D 73 00 00 46 72 61 6D 65 20 3A 20 25 75
```

That hit is row 1. A second pattern, `"DLSS: %s"` followed by `"Vulkan %s"`, is row 6 on builds that have the DLSS line. `GetOffset` stores `address - module base`. A miss logs `Could not find the perf metrics rows in memory` and leaves the pointers unset, so later ticks no-op on `CheckNonZeroPtr`.

Known builds skip the scan. `Row1` / `Row6` in `offsets.json` are those same relative offsets.

`DeepPointer.DerefOffsets` (`Memory/DeepPointer.cs`) adds the module base, then walks any extra offsets by reading a pointer at each step. Row pointers have an empty offset list (the class still inserts a trailing `+0`), so they stay on the static string.

## One tick

`MemoryTick` (`Memory/MemoryHandler.cs`), in order, for the overlay:

1. `DerefPointers()` so a rebasing or a failed read does not leave a stale address. A `Win32Exception` aborts the deref and the rest of the tick still runs against the last good pointers.
2. On `1.0 (Release)` only, read the slopeboost byte and set the `slopeboost` flag when it is off.
3. `ReadTrainerValues()` when the position pointer is non-zero. Velocity is always stored. Position, yaw, and pitch are stored only when Meathook or cheats are on.
4. Font size. If `EnableFontSizeChange` is set, write `5 + OSDFontSize * 0.5` into `MetricsFontSize` (`con_fontSize`). Otherwise write `10`, the game default. The write is skipped when the current float already matches.
5. Clear `_row1` … `_row9`, `_cpu`, `_gpuV`, `_gpuN`.
6. Set `_row1` to `%i FPS`, and append `*` when the `outofdate` flag is set.
7. Branch on `EnableOSD`, the external-trainer flag, and the in-game trainer flag. Build the other rows. Call `SetMetrics` and `ModifyMetricRows`.

Rows 4 and 5 are cleared at the start of the tick and, while the OSD is on, never filled again. The write still happens, so those stock lines (`HDR`, `Vulkan`, and on older layouts the raytracing line) are replaced with empty strings. That is what removes the hardware block and leaves the status lines.

## What each line becomes

Constants are in `Define/Constants.cs`.

### Row 1 stays a format string

`METRICS_FPS_TEXT` is `"%i FPS"`. DESRU does not write the FPS number. The engine keeps a `%i` on that line and fills the integer itself. The optional `*` is a literal character after the format, so an out-of-date DESRU shows as `123 FPS*`. `Program.UpdateDetected` sets the `outofdate` flag when the hook is established (`MainWindow` around the `Memory.SetFlag` calls after a successful hook).

### Full OSD (`minimal` flag off, metrics byte `2`)

Row 2 is the game version from `GameVersion.Name`, passed through two replacements:

- `" Rev "` becomes `"r"` (`6.66 Rev 2` → `6.66r2`)
- `" (Gamepass)"` is removed
- `"1.0 (Release)"` becomes `"Release"`

If any status flag is set, ` (` + letters + `)` is appended. Letter order is fixed:

| Letter | Flag         | Set from                                                                                                                                        |
| ------ | ------------ | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| `M`    | `macro`      | Freescroll macro enabled and a key is bound (`MainWindow` form timer)                                                                           |
| `F`    | `firewall`   | A firewall rule exists for the game exe                                                                                                         |
| `R`    | `reshade`    | ReShade files found next to the game at hook time                                                                                               |
| `S`    | `slopeboost` | `1.0 (Release)` only, when the ramp-jump byte reads as slopeboost disabled                                                                      |
| `L`    | `limiter`    | The stored flag is **false**. The checkbox path sets `limiter` to true when “enforce 250 FPS” is on, so `L` means the cap is not being enforced |

Example: `6.66r2 (MFL)`.

The line under that is one of, first match wins:

1. `_resetRunString` (`RUN RESET`) while a reset script is on screen. It is set when the reset script runs, cleared 5 seconds after cutscene id `1` is seen, and it replaces the cheat line until then.
2. `RESTART GAME` when the `restart` flag is set.
3. `CHEATS ENABLED` when `EnableCheats` is on.
4. `MEATH00K` when `XINPUT1_3.dll` was found in the game directory at hook time.
5. Nothing, in which case the resolution-scale readout is used instead (below).

`MODDED CLIENT` replaces an empty cheat line when the `modded` flag is set (UWP / modded client detected at hook). If a cheat line is already present and the text is not `RESTART GAME`, ` (MOD)` is appended.

That string is written to row 3 when `_cpuPtr` is zero. When the CPU pointer resolved, row 3 is cleared and the same text is written to the CPU string (64-byte buffer). The engine prints the CPU line as part of the same metrics stack, which is why the status still shows up under the version on those builds.

Scroll pattern (`SetScrollPatternString`, fed by the form timer) replaces that status line while it is non-empty. The form shows it for about 2 seconds: scroll count and the average milliseconds between wheel clicks.

Resolution scale is shown for 3 seconds after `CurrentResScaling` changes (`_scalingTime`). Format is `{scale:0.00}x [{target}]`. The bracket is `S` when `rs_forceResolution` is in use, otherwise the dynamic target FPS from `rs_raiseMilliseconds` (`1000 / (ms / 0.95)`). If dynamic and static scaling are both off, the text is `rs-off`. It only occupies the status line when the cheat / Meathook / reset line is empty.

### Minimal OSD (`minimal` flag on, metrics byte `1`)

The version and flags are folded into the first two printed lines so a one-line metrics mode still shows them. The game prints row 1, then row 2 directly under it, so the split reads as one sentence.

- `MODDED CLIENT` with no other cheat text is rewritten to a `MOD-` prefix on the version.
- `(` is stripped from the version, then re-inserted: on row 1 when DESRU is current, on the front of row 2 when the `*` is present (the `*` already consumed the spare byte).
- Row 2 gets a `)` if it does not already have one.
- Cheat text wins over the version. The first character is written into row 1 at index 7 (`_row1[..7]`), and the rest goes to row 2. With the out-of-date `*`, the whole cheat string stays on row 2, because index 7 is already `*`.
- If there is no cheat text but a scale readout is active, it is inserted before the closing `)`: `6.66r2 (MF [0.75x S)`.

`CHEATS ENABLED` / `MEATH00K` become `CHEATS (MOD)` / `MEATH00K (MOD)` in this mode when `modded` is set.

Metrics mode `0` in `offsets.json` forces the `minimal` flag on (`Initialize`), because that build has no mode byte to switch to the taller layout.

### Trainer text (`Show In-Game Display`)

The checkbox sets the `trainer` flag every form tick. The trainer branch runs only when that flag is set **and** Meathook or cheats are on. It replaces the version block and calls `SetMetrics(2)`:

| Slot                                            | Text                                                                                |
| ----------------------------------------------- | ----------------------------------------------------------------------------------- |
| Row 2                                           | `vel: {total:0.00}` — `sqrt(vx² + vy² + vz²)`, not the speedometer’s horizontal m/s |
| GPU name string                                 | `x: … y: … z: …`                                                                    |
| CPU string, or row 3 if there is no CPU pointer | `h: {horizontal:0.00} v: {vz:0.00}`                                                 |
| Row 8, or row 6 if `Row6` was not found         | `yaw: {yaw:0.0}`                                                                    |
| Row 9, or row 7                                 | `pitch: {pitch:0.0}`                                                                |

Yaw and pitch are normalized with `(360 + raw) % 360` in `ReadTrainerValues`.

The form’s `Speedometer` panel is a separate WinForms control. It is not this overlay. See `docs/speedometer.md`.

### OSD checkbox off

`EnableOSD` is set from `enableOSDCheckbox` at hook time and in `DisableOSD_CheckChanged`. While it is false, the tick runs the restore path **once** (`_osdReset` starts false):

- Row 1: `%i FPS` (the `*` is removed)
- Row 2: `%.2fms`
- Row 3: `%d x %d (%s)`
- If `Row6` was not found (older layout): rows 4–6 become `HDR: %s`, `Vulkan %s`, `VRAM %llu MB%s`
- If `Row6` was found: rows 4–8 become `RT: %s`, `HDR: %s`, `DLSS: %s`, `Vulkan %s`, `VRAM %llu MB%s`

`ModifyMetricRows` writes that, then `_osdReset` is set so later ticks do not keep clobbering a metrics block the user turned off. Turning the checkbox back on clears `_osdReset` because the enable path sets it false after every write.

`ClosingDESRU` stops both timers and writes `DESRU CLOSED` into the status slot (row 3, or the CPU string; in minimal mode it is split across row 1 and row 2 the same way as cheat text). Unknown versions write it at `_row1Ptr + 0x7`, or `+ 0x8` when the `*` is present.

## Memory layout of the rows

`ModifyMetricRows` writes fixed-length ASCII. `ToByteArray` allocates `length` zero bytes and copies the string in, truncated to that length. The first `0` after the text is the C terminator. A string that fills the whole buffer has no terminator inside the buffer.

| Field   | Address                                        | Bytes written | Stock contents                             |
| ------- | ---------------------------------------------- | ------------- | ------------------------------------------ |
| `_row1` | `_row1Ptr`                                     | 20            | `%i FPS`                                   |
| `_row2` | `_row1Ptr + 0x8`                               | 18            | `%.2fms`                                   |
| `_row3` | `_row1Ptr + 0x58`                              | 19            | `%d x %d (%s)`                             |
| `_row4` | `_row1Ptr + 0x70`                              | 7             | `HDR: %s`                                  |
| `_row5` | `_row1Ptr + 0x78`                              | 34            | `Vulkan %s`                                |
| `_row6` | `_row6Ptr` if non-zero, else `_row1Ptr + 0x98` | 34            | `DLSS: %s` or `VRAM %llu MB%s`             |
| `_row7` | row 6 + `0x10`                                 | 34            | next stock line                            |
| `_row8` | `_row6Ptr + 0x30`                              | 34            | only when `Row6` resolved                  |
| `_row9` | `_row6Ptr + 0x40`                              | 34            | only when `Row6` resolved                  |
| `_cpu`  | `_cpuPtr`                                      | 64            | CPU name                                   |
| `_gpuV` | `_gpuVendorPtr`                                | 64            | vendor                                     |
| `_gpuN` | `_gpuNamePtr`                                  | 64            | GPU name (trainer position uses this slot) |

Row 1’s 20-byte write runs past row 2, which starts 8 bytes later. The row 2 write at `+0x8` repairs that. Minimal mode uses the overlap on purpose: `"%i FPS"` is 6 bytes, index 7 is the first byte of the next stock string, and the splice puts one status character there so the two lines read continuously.

`SetMetrics` writes one byte to `_metricsPtr` (`offsets.json` key `Metrics`):

- `1` — short metrics, used by minimal OSD
- `2` — taller block, used by the full OSD and by the trainer text
- The method returns without writing if the value is above `6`

The tick calls this every 16 ms while the OSD is on, so a change in the game’s own menu does not stick.

## When the overlay is left alone

`RestartTick` runs on a 2500 ms `System.Timers.Timer`, not the 16 ms tick, because the process walk was measured at about 3 ms.

- A process whose name contains `DoomEternalTrainer` sets `_externalTrainerFlag`. While that is true, `MemoryTick` skips the whole OSD rewrite (`if (!_externalTrainerFlag && EnableOSD)`), and sets `restart` unless Meathook is already loaded. The help text tells the player to restart the game: DESRU will not vouch for a session where the external trainer was seen.
- A process whose name contains `cheatengine` (case-insensitive) sets `restart` and `_cheatString = "RESTART GAME"`, unless Meathook is loaded or `restart` is already set. The rows keep updating; the status line becomes `RESTART GAME`.

Flags are stored on `MemoryHandler` and read only when a line is built. `SetFlag` / `GetFlag` take the string names above (`meath00k`, `firewall`, `limiter`, `macro`, `outofdate`, `reshade`, `restart`, `slopeboost`, `minimal`, `trainer`, `modded`). Unknown names are ignored.

## Why it looks like an overlay

The metrics cvar positions that text in the top right and picks the font (`con_fontSize`, written by `SetOSDFontSize`). DESRU never chooses screen coordinates. Emptying rows 4 and 5, and replacing rows 2 and 3, is what makes the block look like a DESRU patch instead of FPS / frametime / resolution / GPU. Putting `%i FPS` back, and restoring the `%` format strings, is what hands the block back to the game.
