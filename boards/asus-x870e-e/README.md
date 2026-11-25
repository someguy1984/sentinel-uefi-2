# boards/asus-x870e-e

ASUS ROG Strix X870E-E Gaming WiFi — AMD AM5, AMI Aptio, AMD AGESA. This is the board
that v1 (`../sentinel-uefi`) was built on, and the first board folder of Sentinel UEFI v2.

Everything that is true of this board, and only of this board, lives here. The framework
(`sentinel/`, `tools/`, `uefi/`) never depends on this folder; this folder depends on the
framework. See §4.16 of the main [`PLAN_v2.md`](../../PLAN_v2.md) for the rules every board
folder follows, and [`PLAN.md`](PLAN.md) here for this board's own plan.

> **Status: planning.** Only this file and `PLAN.md` exist. Once `run.py board map`
> exists, the map below is generated into `MAP.md` and kept current by a test.

## Map of this folder

| Entry | What it holds | State |
|---|---|---|
| `README.md` | This file | present |
| `PLAN.md` | The board plan: decisions, images, evidence, regressions, lineage, board track B0–B5 | present |
| `MAP.md` | Generated contents map with counts of images, recipes, modules, bundles and regressions | planned (B0) |
| `board.yaml` | Identity and matching rule (`$FID` tag `A5557`, SMBIOS), silicon, firmware vendor, write route (USB BIOS Flashback), which event-log entries identify the image | planned (B0) |
| `profile.yaml` | Platform profile: Zen 5 CPU family gates, Setup and PCD defaults, SMRAM assumptions | planned (B0) |
| `known_bad.yaml` | SHAs that must never be used as a build base, with reasons | planned (B0) |
| `recovery.yaml` | The chip-recovery artefact: chip-sized bytes for an external programmer, by SHA | planned (B0) |
| `images.yaml` | Records of this board's images, on disk or not, by capsule SHA and measured digests | planned (B1) |
| `images.built.json` | Identity of every capsule built for this board (generated) | planned (B4) |
| `pcr0.yaml`, `assertions.yaml` | Measured ranges; whole-image assertions | planned (B4) |
| `contrib/` | Board-level plugin data, merged after the ASUS OEM data | planned |
| `tools/evidence/legacy/` | Decoders for the record layouts of v1's older captures | planned (B2) |
| `recipes/` | This board's recipes; `legacy/` holds v1's revs distilled into data | planned (B4) |
| `edk2/` | Scaffolded EDK2 workings: platform DSC/FDF, the board modules (AcpiNeuter, XhcS0w), `derive/` for image-derived constants, board PCD values, `assertions.yaml`, the module manifest, DEPEX pins | planned (B5) |
| `evidence.yaml` | Index of this board's evidence bundles (the bundles themselves are never tracked) | planned (B2) |
| `docs/` | `operator-loop.md` (Flashback procedure and hazards), board history and decisions | planned (B0) |
| `tests/` | `fixtures.yaml` (hashes and locations, never binaries), `regression/` (the five v1 regressions), frozen v1 oracles | planned (B1, B3) |

Firmware binaries, captures and anything that identifies the machine never enter this
folder: it is tracked in git.
