# AnyKyty: Ghost of Yōtei compute pipelines that take 1–4 minutes to compile

## Setup
- AnyKyty `yotei/best` 4fb7613b70 (AnyKyty 7de61e3451 + Yōtei commits), DLL build, Windows 11.
- Ryzen 5 3600, RTX 3080 10 GB, NVIDIA driver 610.88, Ghost of Yōtei 02.002.000.
- Default settings for playing (`KYTY_ASYNC_PIPELINES=1`), with `KYTY_PIPELINE_STALL_LOG_MS=1000`.

## What happens in play
Buffer-writing compute dispatches wait for their pipeline (`compute-sync` / `compute-wait`, `why=writes-buffers`). Playing hits freezes of 5–25 s in new areas, and some of 2–3 minutes with only music playing (for example `[kyty-gpl] stall 119.598 s compute-wait 0x9d7131775f0cbbda`, `stall 184.559 s compute-sync 0x73b6a2fc1ccbbf27`).

## Cold start
With `AppData\LocalLow\NVIDIA\GLCache` and `_PipelineCache\PPSA26344.bin` moved aside, startup took 724 s to create the 3516 stored pipelines on 6 threads. 258 compute pipelines took ≥ 1 s. The slowest:

| Shader hash | Driver compile | SPIR-V words | Note |
|---|---|---|---|
| 0x85345fc443c5178f | 255 s | 158,451 | |
| 0x88ed2a5653800e9c | 180 s | 143,075 | |
| 0x73b6a2fc1ccbbf27 | 179 s | 144,449 | |
| 0x1644eb398013a14e | 173 s | 167,823 | |
| 0x9d041667a02720e3 | 150 s | 619,505 | BVH traversal left to emulation, 6 of 6 intersections run empty |
| 0xc2bf8069153c6232 | 141–144 s | 191,897 / 195,464 | two variants |
| 0x9d7131775f0cbbda | 124 s | 596,789 | BVH traversal left to emulation, 6 of 6 intersections run empty |
| 0xfa1d388fee13854b | 69–75 s | 535,072 / 536,978 | two variants |
| 0x9343b120cf56a68c, 0x954024f4f04f71f9, 0x6540d35b864ba0a8, 0x228fc43eeb7014aa | 30–35 s | 276k–340k | 2–3 variants each |

These times are with 6 compiles in parallel, so a single compile is somewhat faster. In play, the same hashes took 109–184 s.

## Observations
- Compile time does not follow module size. The four slowest (143k–168k words) compile 2–3x slower than modules of 535k–620k words.
- The slow non-ray-tracing modules form two near-identical pairs (0x73b6 ≈ 0x88ed, 0x1644 ≈ 0x8534), plus 0xc2bf and 0x69c9a94015c9c4c5 (27 s, 59k words). Each has two small helper functions with a 77-case `OpSwitch` (called once each) and 116–204 `OpControlBarrier`. The fast large modules (0xfa1d, 0x9343, 0x9d04) have no `OpSwitch`.
- The two ray-tracing modules mark every BVH intersection empty, but the emulated traversal loop and the triangle function are still emitted.
- Variants: in `PPSA26344_02.002.000.kpm`, 196 of 687 compute shaders have more than one variant. Most differ in the number of images bound (`si` length), then buffer count, image dimension (2D/3D), buffer empty vs real and buffer format. New modules per session were 129, then 187, then 428.

I did not get further than this because `kyty_compute_pipeline_replay` is only built when `ANYPS5_KYTY_DLL` is off.

Measured on 2026-11-24. Measurements and write-up made with AI assistance (Claude Code).
