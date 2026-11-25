# Ghost of Yōtei performance patches

Patches that make Ghost of Yōtei (PPSA26344) run faster on PC PS5 emulators. They use the game's own quality settings to render less. All patches are for game version **02.002.000** only.

They change the game's code in memory or at conversion time, never the game files on disk. The files list instruction addresses and replacement bytes and contain no game code.

## Results

Measured on a Ryzen 5 3600 + RTX 3080 10 GB with 24 GB RAM, game 02.002, in the open world:

| Emulator | Full detail | With the row 15 patch |
|---|---|---|
| AnyKyty `yotei/best`, 1080p output | 15.6 fps | **23.8 fps** (about 21–25 in actual play) |
| SharpEmu main | 3.3 fps | 9.2 fps |

AnyKyty at full detail with the game's normal 4K output runs at 8.1 fps. The 1080p output patch roughly doubles that. In new areas the frame rate drops, to about 10–13 fps, while AnyKyty compiles the shaders it hasn't seen before.

> **It will look worse.** Expect weird, glitchy textures, blocky edges, ghosting and double images on moving characters, and some flicker. That's the price I chose to pay for the frame rate. Still scenes and menus look fine.

## Which file

| Emulator | File | What it does |
|---|---|---|
| AnyPS5 / AnyKyty | [`anyps5/anyps5-yotei-02.002-1080p-ladder15-rtgi-off.toml`](anyps5/anyps5-yotei-02.002-1080p-ladder15-rtgi-off.toml) | **Recommended.** 1080p render and output (shahulmubeen-a's 02.002 patch), ladder row 15, and ray-traced GI kept off so the game doesn't crash in some areas |
| AnyPS5 / AnyKyty | [`anyps5/anyps5-yotei-02.002-1080p-ladder15.toml`](anyps5/anyps5-yotei-02.002-1080p-ladder15.toml) | The same without RTGI off. Crashes in some areas, for example the Snake's house |
| AnyPS5 / AnyKyty | [`anyps5/anyps5-yotei-02.002-rtgi-off.toml`](anyps5/anyps5-yotei-02.002-rtgi-off.toml) | RTGI off only, to combine with other patches |
| AnyPS5 / AnyKyty | [`anyps5/anyps5-yotei-02.002-ladder15.toml`](anyps5/anyps5-yotei-02.002-ladder15.toml) | Ladder row 15 only |
| AnyPS5 / AnyKyty | [`anyps5/anyps5-yotei-02.002-ladder11.toml`](anyps5/anyps5-yotei-02.002-ladder11.toml) | Ladder row 11 only (less detail lost, less speed gained) |
| SharpEmu | [`sharpemu/GhostOfYotei.xml`](sharpemu/GhostOfYotei.xml) | **Performance mode** (row 15), **Balanced mode** (row 11), **Output at 1080p** |

Use one ladder patch at a time. The relinker takes one `--patches` file, which is why the combined files exist.

### AnyPS5 / AnyKyty

Pass the file to the relinker when you convert the game, with `--patches <file>` (for example together with `--windows`). Check it first with `--verify-patches`: every site must match your unwrapped `eboot.elf` (32 for the recommended file). The relinker prints each patch it applies.

### SharpEmu

Put `GhostOfYotei.xml` in SharpEmu's `user/patches/` folder, then turn on the patch you want in the game's patch list in the SharpEmu window. The ladder patches have also been submitted to [sharpemu/ps5patches](https://github.com/sharpemu/ps5patches). **Output at 1080p crashes on SharpEmu main** when the world loads ("The image view is invalid"). It only works on [my SharpEmu fork](https://github.com/DarthPotato/sharpemu), which fixes that bug.

## What the patch does

Yōtei has its own internal quality ladder: a table of 22 rows of {LOD scale, sun shadow view delta, small shadow caster cull, …}, plus one int that holds the current row. Row 0 is full quality.

| Row | LOD scale | Sun shadow views | Small shadow caster cull |
|---|---|---|---|
| 0 | 1.0 | all | 0 |
| 11 | 0.2 | 2 fewer | 0 |
| 15 | 0.1 | 3 fewer | 0.2 |

At row 15 the game itself sends about 40–45% fewer draws and dispatches. The game also rewrites the row while it runs, so the patches change all 17 instructions that read it (`movsxd r64, [rip+row]` → `mov r32, <row>; nop`). Rows 16–21 also stop drawing the player's body, so row 15 is the lowest useful row. The game looks rougher: lower-detail models, sparser grass and fewer shadows.

Addresses, the render-size settings and other notes are in [FINDINGS.md](FINDINGS.md). Measurements of the AnyKyty shader compile freezes are in [anykyty-shader-compile-report.md](anykyty-shader-compile-report.md).

## Known issues on AnyKyty

- **Long freezes with only music playing.** A few compute shaders can't be compiled in the background, so the game waits for the NVIDIA driver to compile them: usually 5–25 s, and 2–3 minutes for a few very large ray-tracing shaders (they still run with the ray tracing option off). Wait it out; the game carries on. AnyKyty keeps every compiled shader, so each freeze happens only once per machine and they get rarer the more of the map you've played. They don't go away completely, because the game keeps producing new versions of the same shaders (mostly with a different number of textures bound).
- **`FATAL: Insufficient RTGI memory` crash.** The game turns on ray-traced global illumination by itself in some places, for example walking into the Snake's house, even with ray tracing off in the menu. It also turns on if you enable ray tracing in the graphics menu, and that setting is saved. With the 1080p patch, the memory block RTGI needs doesn't fit (345 MB free, 364 MB needed), so the game stops itself. Even when it fits, AnyKyty can't translate one of the RTGI compute shaders yet. **Use the recommended file with RTGI off**: it keeps RTGI off everywhere, and I played through the Snake's house and the boss fight with it. Indoor lighting is a bit flatter, and you get a few frames back.
- **Changing patches resets the shader cache.** The NVIDIA driver keeps a separate shader cache for each converted executable. After you relink with a different patch file, the first session starts cold again: about 12 minutes of shader compiling in the background after the game starts, plus the long freezes. Pick your patch file once and keep it.
- **Memory.** A first run pushed committed memory near 40 GB. On a 24 GB machine, leave the Windows page file system-managed.
- **Settings for playing.** Keep the default `KYTY_ASYNC_PIPELINES=1`. With `0` it freezes far more often, and `2` crashed for me.
- **Xbox controller.** View presses the touchpad. To draw on the touchpad (for example when writing names), start with `ANYPS5_MOUSE_TOUCHPAD=1` and drag with the left mouse button.

## What the RTGI patch does

The game decides whether RTGI runs by reading one byte (app+0x53c20b0). It reads it in 10 places, all `cmp byte [rip+flag], imm8`, and never writes it directly. The patch points each of those reads at a constant zero byte in read-only data (app+0x1a4e666), so the game always sees RTGI as off. The memory check is at app+0x13ee3e0. It asks for 364 MB, or 397 MB when a second flag at app+0x464a5e4 is set, which is probably the PS5 Pro mode.

## Credits

- The 1080p output sites for AnyPS5 are shahulmubeen-a's 02.002 relinker patch.
- AnyKyty `yotei/best`: https://github.com/Celegans12/AnyPS5-Ghost-of-Yotei
- The patches and measurements were made with AI assistance (Claude Code and Codex). I checked the patch sites and ran the tests on my machine.

Released under CC0: use, copy and adapt freely.
