# Speedometer (m/s)

The number shown as `M/S` is the player's **horizontal** velocity: `sqrt(vx² + vy²)`. Vertical (`vz`) and total (`sqrt(vx² + vy² + vz²)`) are the optional second line. The game floats are already in meters per second; nothing converts units before display.

There is no file export or HTTP endpoint in this project. The values are updated in memory and painted on the form. The sections below name the functions to call if you add either.

## Where the number is produced

| Step | Location |
| --- | --- |
| Pointer chain | `DESpeedrunUtil/Memory/MemoryHandler.cs`, offset setup around `_velocityDP = CreateDP(...)` |
| Offset tables | `DESpeedrunUtil/Define/Constants.cs` — `VEL_OFFSETS_CURRENT` (game major ≥ 3) and `VEL_OFFSETS_OLD` |
| Base address per build | `DESpeedrunUtil/Resources/offsets.json`, key `"Velocity"` |
| Read + math | `MemoryHandler.ReadTrainerValues()` |
| Public snapshot | `MemoryHandler.GetPlayerVelocity()` → `(velX, velY, velZ, horizontal, total)` |
| UI tick | `MainWindow.cs`, trainer block inside the form timer (`_formTimer.Interval` = `Program.TimerInterval`, default **16 ms**) |
| Paint as `M/S` | nested class `MainWindow.Speedometer` (`TEXT_FORMAT` / `TEXT_FORMAT_PRECISION`) |

`ReadTrainerValues` does this:

```csharp
_velocityX = _game.ReadValue<float>(_velocityPtr);
_velocityY = _game.ReadValue<float>(_velocityPtr + 4);
_velocityZ = _game.ReadValue<float>(_velocityPtr + 8);
_velocityHorizontal = (float) Math.Sqrt((_velocityX * _velocityX) + (_velocityY * _velocityY));
_velocityTotal = (float) Math.Sqrt((_velocityX * _velocityX) + (_velocityY * _velocityY) + (_velocityZ * _velocityZ));
```

The big number on the speedometer is `_velocityHorizontal`. The bar and the smaller line use total velocity unless the "vertical" radio is checked, in which case they use `_velocityZ`. Cheats are not required for this read; position and rotation are skipped when cheats and `meathook` are both off, but velocity is always read.

The pointer is:

`DOOMEternalx64vk.exe + offsets.json["Velocity"]` then the four offsets in `VEL_OFFSETS_*`.

Current (major version ≥ 3): `0x1510, 0x5C0, 0x1D0, 0x3F40`.  
Older: `0x1510, 0x598, 0x1D0, 0x3F40`.

`DeepPointer` resolves that chain into `_velocityPtr`. The three floats are contiguous: x, y+4, z+8.

## File export

Hook the same place the UI already samples, inside the trainer block of the form timer in `MainWindow.cs`, right after:

```csharp
var (velX, velY, velZ, hVel, totalVel) = Memory.GetPlayerVelocity();
```

Write one line per tick. The timer is ~16 ms, so throttle if you do not want ~60 lines per second (for example only when `hVel` changes, or every 50–100 ms).

CSV example, append-only so a crash does not lose earlier samples:

```csharp
File.AppendAllText(
    Path.Combine(AppContext.BaseDirectory, "velocity.csv"),
    $"{DateTime.UtcNow:O},{hVel:0.00},{velZ:0.00},{totalVel:0.00}{Environment.NewLine}");
```

`File.AppendAllText` opens and closes the file each call, so another process can read it between writes. For a single latest value, overwrite a small file instead:

```csharp
File.WriteAllText(
    Path.Combine(AppContext.BaseDirectory, "velocity.txt"),
    hVel.ToString("0.00", CultureInfo.InvariantCulture));
```

Keep the write on the form timer thread. `GetPlayerVelocity` only returns the last values `ReadTrainerValues` stored; it does not touch the game process itself.

## Local API

The app is a WinForms process with no listener. A loopback server has to be started from `MainWindow` after `Memory` exists (constructor / load), and stopped on form close.

Minimal read-only endpoint, `GET http://127.0.0.1:9271/velocity`:

```csharp
var listener = new HttpListener();
listener.Prefixes.Add("http://127.0.0.1:9271/");
listener.Start();
_ = Task.Run(async () => {
    while (listener.IsListening) {
        var ctx = await listener.GetContextAsync();
        var (x, y, z, h, total) = Memory.GetPlayerVelocity();
        var json = $"{{\"x\":{x:0.00},\"y\":{y:0.00},\"z\":{z:0.00},\"horizontal\":{h:0.00},\"total\":{total:0.00}}}";
        var bytes = Encoding.UTF8.GetBytes(json);
        ctx.Response.ContentType = "application/json";
        ctx.Response.ContentLength64 = bytes.Length;
        await ctx.Response.OutputStream.WriteAsync(bytes);
        ctx.Response.Close();
    }
});
```

`HttpListener` on `127.0.0.1` does not need a URL ACL. Bind only to loopback. Format the floats with `CultureInfo.InvariantCulture` so the JSON uses `.` as the decimal separator.

`Memory.GetPlayerVelocity()` is a tuple of plain floats written by the memory timer and read by the form timer. A single float read on another thread is acceptable for a display feed; if you ever add writes next to it, take a lock around the five fields.

Check it with:

```text
curl http://127.0.0.1:9271/velocity
```

Expected body:

```json
{"x":0.00,"y":0.00,"z":0.00,"horizontal":12.34,"total":12.40}
```

`horizontal` is the same value the speedometer prints as `M/S`.
