# Sentinel UEFI v2

An automated UEFI firmware analysis service. Drop a firmware capsule into a folder, run
one command, and get a full workspace for it: the extracted image, an interactive map of
every module, emulation-driven defect hunting across SMM, DXE and PEI, and the ability to
build modified capsules from declarative recipes.

v2 supersedes [`sentinel-uefi`](../sentinel-uefi) (v1), a long-running research and
hardening project against a single ASUS / AMD / AMI board. v1's tooling and lessons are
being reorganised here into one capsule-agnostic tool. AMD is supported first; Intel is
planned.

> **Status: planning.** Nothing below is implemented yet. This README describes the
> intended tool; [`PLAN.md`](PLAN.md) is the full project plan and roadmap.

## How it will be used

```
cp ~/Downloads/A5557.cap firmware_dropzone/      # any name, any extension
python run.py                                    # ingest, analyse, map, emulate
xdg-open firmware/A5557_20261124-130512/map/index.html
```

The dropzone accepts **any file extension** — `.cap`, `.bin`, `.bios`, `.rom`, or none.
The file type is decided by its content, never its name. A file with no recognisable
firmware structure is moved to `firmware/_rejected/` with the reason.

| Command | Purpose |
|---|---|
| `python run.py` | Process everything in `firmware_dropzone/` |
| `python run.py doctor` | Check dependencies and report which stages they disable |
| `python run.py list` / `show <ws>` | List workspaces / summarise one |
| `python run.py rerun <ws> [stage]` | Re-run a stage on an existing workspace |
| `python run.py map <ws> --open` | Rebuild and open the module map |
| `python run.py findings <ws>` | List leads with their evidence tier |
| `python run.py emulate <ws>` | Run the emulation hunters |
| `python run.py diff <wsA> <wsB>` | Compare two images |
| `python run.py evidence add <ws> <bundle>` | Attach evidence captured from a live board |
| `python run.py build <recipe.yaml>` | Build a modified capsule from a recipe |
| `python run.py watch` | Keep watching the dropzone |

## What a workspace contains

Each dropped file gets its own folder, `firmware/{name}_{datetime}/`, holding all workings:

```
manifest.json     identity (SHA-256), original name, container chain
input/            the original file, read-only
image/            the unwrapped firmware image
extract/          browsable tree of volumes, files and sections
modules/          one folder per executable module
model/            the parsed image as JSON: regions, volumes, modules, dependency graph
analysis/         output of each analysis plugin
emu/              emulation run records and coverage
evidence/         evidence bundles captured from a live board
findings/         leads, coverage report, summary
overlay/          hand-written corrections and annotations
map/index.html    the interactive module map (opens offline)
build/            capsules built from recipes
```

## Repository layout

```
run.py                 entry point
sentinel/              shared core library (generic UEFI; no vendor knowledge)
tools/                 analysis, emulation, evidence and build plugins
uefi/                  Sentinel firmware module sources (EDK2-based)
collect/               read-only capture kit for a live board
recipes/               capsule build recipes
data/                  platform profiles, board data, GUID name databases
firmware_dropzone/     drop firmware here
evidence_dropzone/     drop evidence bundles here
firmware/              workspaces (git-ignored)
docs/  tests/
```

Plugins are filed by phase, then by what they apply to:

```
tools/<phase>/common/            generic UEFI
tools/<phase>/amd|intel/         silicon-specific
tools/<phase>/ami/               firmware-vendor specific
tools/<phase>/<silicon>/<oem>/   OEM extension, e.g. tools/smm/amd/asus/
```

Firmware module sources follow the same idea, filed by what the code is compatible with:
`uefi/<phase>/{common,amd,ami,board/<name>}/`. Upstream EDK2 itself is not kept in this
repository; a bootstrap step fetches the pinned release and applies the project's patches.

## Principles

These come from v1's experience and are enforced in code, not left as convention.

- **Leads, not verdicts.** Every automated result is a lead with an evidence tier and a
  confidence level. A human decides.
- **Not run is not clean.** A check that could not run is reported as such, and the
  report always lists what was not covered.
- **Identity is the hash.** A firmware image is identified by its SHA-256, never by its
  file name.
- **Measured beats static.** Evidence captured from a real board overrides static and
  emulated conclusions, and the disagreement is reported.
- **Corrections go in overlays.** Generated output is never edited by hand.
- **It never flashes.** The tool has no code path that writes to hardware. A built
  capsule comes with a statement of what was and was not verified, and a checklist for
  the person who decides whether to flash it.

## Supported platforms

| Platform | State |
|---|---|
| AMD (AGESA / PSP), AMI Aptio firmware, ASUS boards | First target |
| Intel | Planned; the layout is designed so it can be added without restructuring |

## Requirements

Python 3 with [`uv`](https://docs.astral.sh/uv/) for the environment. Planned
dependencies are `capstone`, `unicorn`, `pyyaml`, `pydantic` and `brotli`. Optional
external tools include `iasl`, `objdump` and `psptool`. `python run.py doctor` will
report what is present and what each missing piece disables.

## Relationship to v1

v2 is self-contained: everything worth keeping from `sentinel-uefi` and its precursors
is copied in, in a layout designed for reuse, and v2 does not depend on those
repositories at run time. v1's history, rules and failure lessons will be distilled into
`docs/LESSONS.md`. See [`PLAN.md`](PLAN.md) for the research findings, architecture,
migration map and phased roadmap.
