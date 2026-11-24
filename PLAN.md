# Sentinel UEFI v2 — project plan

> **Status:** plan, 2026-11-24, revised the same day after an end-to-end coverage review
> against v1. Nothing is implemented yet; work follows the phased roadmap in §6.
>
> Built from eleven research and design agent reports on `../sentinel-uefi` (v1),
> `../uefi` and `../uefi_framework`, plus decisions made by the project owner during planning.
> The coverage review (ten area readers over v1, four adversarial verifiers, three
> independent planners) drew the scope boundary of §2.1, corrected about thirty statements
> about v1 and added build, gate and analysis mechanisms throughout §4. Each factual
> correction was checked against v1's files or against the bytes of `A5557.cap`.

## 1. Context

`../sentinel-uefi` (v1) is a 327-commit (Mar–Nov 2026) project that replaced the ASUS/AMI
DXE and SMM service layer of one board (ASUS ROG Strix X870E-E, Ryzen 9950X, AMI Aptio,
AMD AGESA) with edk2 code and custom "Sentinel" modules across ~125 firmware revisions.
It holds ~477 Python scripts and a large body of hard-won lessons, but it grew
investigation by investigation: tooling is scattered over a dozen trees, 61 of the 75
Python files in `firmware/builds` (57 of them builders) hardcode a path that no longer
exists, there is no dependency manifest, and almost every script assumes this one board
and a particular revision.

v2 turns that into an automated, capsule-agnostic service. A firmware file of any name
is dropped into `firmware_dropzone/`; `python run.py` moves it into its own workspace
`firmware/{capsulename_datetime}/`, identifies and extracts it, produces an interactive
UEFI module-map web page, emulates the major segments (SMM, DXE, PEI) hunting for
defects, and can build new capsules from declarative recipes. Tooling is organised as
phase / architecture / vendor plugins so Intel can be added later. v2 is self-contained:
everything worth keeping from v1 and its precursors is copied in, in a layout designed
for reuse, and v2 never depends on the old repositories at run time. v2 ends at a gated
capsule, its generated documents and imported board evidence: flashing and the board's
release bookkeeping stay with the human (§2.1).

## 2. Requirements and decisions

Stated by the user:

- Entry point `python run.py`; input `firmware_dropzone/`; workspace
  `firmware/{capsulename_datetime}/` holding all workings.
- **Any file extension** is accepted (`.cap`, `.bin`, `.bios`, `.rom`, none). Type is
  decided by content, never by extension.
- Pipeline order: analyse → module-map page → emulate segments hunting defects → build.
- Plugins: `tools/smm/(amd|intel)/*.py`, `tools/pei/(amd|intel)/`, OEM extensions such as
  `tools/smm/amd/asus/`. AMD first.
- Preserve v1's history and teachings; streamline.

Decided by the user during planning:

| Decision | Choice |
|---|---|
| Firmware-vendor (AMI) code | **Sibling folder** `tools/<phase>/ami/`, beside `amd/`, `intel/` and `common/` |
| Board evidence | **First-class from the start** (§4.7) |
| Other projects | Everything worth keeping is **copied into v2** in an optimal layout; v2 is self-contained |
| Sentinel EDK2 module sources | Copied into **`uefi/<phase>/<vendor-or-chipset>/`** for reuse (§4.12) |
| Upstream EDK2 | **Kept external and pinned**: fetched by a bootstrap step into a git-ignored folder, patches applied; never committed |
| Precursors `../uefi`, `../uefi_framework` | **Research them too** (§3.6) |

Decided by the user during the coverage review (2026-11-24):

| Decision | Choice |
|---|---|
| D1 Scope | **Analysis + build only.** v2 ends at a gated capsule, its generated documents and imported evidence. The post-build operator loop stays with the human and with documents (§2.1) |
| D2 v1 capsules | None of v1's built capsules (rev5–rev124) is or will be available as a fixture. Validation uses what v1 tracks in git (§7, §8) |
| D3 Board build line v1 → v2 | **Left open** (§2.2). No cut-over gate is designed |

Conventional defaults chosen during planning (open to change):

- Workspace folder is exactly `{capsule file stem}_{YYYYMMDD-HHMMSS}`; a same-second
  collision gets `-2`. **Identity is the SHA-256** (v1: every Flashback image is named
  `A5557.cap`), stored in `manifest.json` and `firmware/index.json`.
- Re-dropping an identical file creates a new dated workspace linked to earlier ones with
  the same SHA; a content-addressed result cache keeps the re-run cheap.
- Unrecognised files move to `firmware/_rejected/{name}_{datetime}/` with `reason.json`.
- `run.py` processes the dropzone and exits; `run.py watch` is the optional service loop.
- v2 **never flashes** and contains no flash-write code path.
- No command deletes build outputs, workspaces or module artefacts; clean-up touches
  `firmware/_cache/` only. (v1's Makefile carries a `clean-builds` target that removes
  every built capsule, against v1's own never-delete list.)

### 2.1 Scope: v1's loop, stage by stage

v1's end-to-end process is longer than v2's pipeline. Every stage is either automated,
supported by a generated document, preserved as a document, or out of scope:

| # | v1 stage | v2 treatment |
|---|---|---|
| 1 | Acquire and baseline: vendor capsule, chip dump, golden copy, security-state baseline | **Automated**: dropzone ingest by content, SHA identity; chip dumps ingest as images (P1); baseline readings arrive as an evidence bundle (P2) |
| 2 | Understand: extraction, DEPEX, SMM / PEI / boot-services maps, ACPI, NVRAM, crypto, option ROMs | **Automated**: extract → static plugins → map → emulation (P1–P5) |
| 3 | Decide what to swap, remove, keep or harden (plans, charters, prune lists with the owner's dated decisions) | **Human.** v2 supplies leads with consumer evidence (§4.9); each recipe operation records `why` and `decided` (§4.11). No planning or decision store |
| 4 | Develop firmware modules (Sentinel packages, derive scripts, host tests, QEMU gates) | **Automated**: `uefi/`, `edk2 bootstrap`, `uefi build`, `uefi gates` (§4.12, P7) |
| 5 | Build a rev (base + operations, lineage, registries, bisect variants) | **Automated**: recipes, operations, gates, child re-ingest (§4.11, P6) |
| 6 | Pre-flash: preflight checklist, independent refutation, backup and flash-medium preparation | **Generated documents**: `FLASHABILITY.md`, `CHECKLIST.md`, `REFUTER_BRIEF.md` per build (§4.11). Carrying them out, and the go / no-go, are human; v2 never reads them back |
| 7 | Flash (USB BIOS Flashback, by hand) | **Out of scope** by design: v2 never flashes |
| 8 | First-boot readings: which image runs, reader output, per-rev snapshot | **Partly automated**: capture kit, import, binding by measurement, observed layer (§4.7). Judging a boot against its build is **out of scope** |
| 9 | Declare state: live / stable / escape image, rollback ladder, never-flash list, capsule retention | **Out of scope.** Only the known-bad list survives, as a build refusal (§4.11) |
| 10 | Investigate anomalies: dated studies, bisects, root-cause synthesis | **Tools only**: findings, overlays, evidence, `diff`, `emulate`. No study store; method and conclusions go to `docs/LESSONS.md` |
| 11 | Report: disclosure-grade reports, a backlog of candidates | **Partly**: leads with evidence tiers, overlay status and an embargo flag (§4.6). Dossiers are written by hand; no disclosure workflow |
| 12 | Track upstream: EDK2 pin, revocation lists, vendor releases, advisories | **Partly**: a new vendor release is dropped in and `diff`ed; pinned `edk2 bootstrap`; sourced advisory matching (§4.9, P9) |

**Not covered, by decision D1.** v2 keeps no board ledger or image states (live / stable /
escape), no rollback ladder, no capsule-retention list, no verdict of a captured bundle
against a build's expectations, no flash-medium check, no board status command and no
investigation or decision store; no code reads or writes such state. v1's knowledge of
these stages is preserved in `docs/operator-loop.md`, written by hand in P0 from v1's
`CLAUDE.md`, `/flash-preflight`, `firmware-refuter.md` and `tests/boot_checklist.md`: the
Flashback procedure and its side effects, what to capture before and after a flash, the
first-boot readings, rollback hazards, retention discipline, the refuter's nine failure
classes and the boot checklist. It holds procedure and hazards only, never live board
state, which stays with the board's build line (§2.2).

### 2.2 Open decisions (owner)

- **Board build line (D3).** Keep building the board's revs in v1 until v2's P7 reproduces
  v1's module set; adopt a recovered image (a chip dump or the flash-medium copy) as a
  recipe base (`adopt`, §4.11); or re-derive the line from A5557, which needs inputs v1
  never tracked. No phase depends on the answer; C-line recipes stay plan-only until it
  is given (P7c).
- **Vendor-line Sentinel SMM modules** (SMI shield, MmAudit W^X stages, vendor-line
  census): frozen in `uefi/legacy/` as planned, or kept alive as a reusable "harden the
  vendor's SMM from outside" path for foreign AMI capsules. Their record decoders are
  needed either way (§4.7).
- **Prebuilt old-line binaries** (v1 `edk2-modules/`, 40 `.efi`): fixtures by SHA so that
  old-line legacy recipes can be exercised, or those recipes declared unbuildable.
- **Hand-written map commentary** (v1 `interactive_map/data/commentary.json`: phase
  prose, module notes, a glossary): import the module notes and glossary, or drop it.
- **Raw third-party scan files** (the CycloneDX SBOM/CBOM files in v1 git): kept outside
  git with only v1's triage verdicts imported (default), or importable data.
- **Embargoed material.** Expected-lead entries and overlay text about undisclosed
  findings stay out of tracked files by default (§4.6). This plan itself already names
  the subject of v1's embargoed report (§4.10, regression 1): whether that wording may
  stay in a pushed file is the owner's call.
- **Emulator evidence in hand-written dossiers**: whether a dossier may cite SMRAM
  guard-page hits (v1's proofs of concept deliberately touched only benign pages).

## 3. Research headline findings

### 3.1 What v1 has and lacks

| Area | State in v1 | Consequence for v2 |
|---|---|---|
| Ingestion | None. At least 13 files carry their own in-memory FV/FFS walker; capsule header stripped by assumption | One extractor, written once, that writes a tree |
| Module map | Static page, 70 KB vanilla JS, opens over `file://`; no dependency graph | Lift the shell; rewrite views; add graph, SMI, NVRAM, findings |
| SMM static analysis | **Strongest asset**: `trace.py` → `smi_registry.py` (130 registrations) → protocol graph → handler audit | Lift; rewrite the taint model |
| SMM emulation | Two proofs of concept on BIOS 2503 (a hook write, 5 scenarios; NvramSmm store rules, 19 steps) sharing one Unicorn helper, at hardcoded RVAs | New general harness; the PoCs become regressions together with their controls |
| PEI / DXE emulation | **None.** A C harness runs modules natively, x64 only, unsandboxed | New; port its stub semantics only |
| PEI dispatch simulator | **Does not exist** (the cited scripts were never committed) | New, shared by analysis and build gates |
| DXE dispatch model with cycle detection | Exists: `firmware/builds/dispatch_model.py`, a DEPEX fixed-point model run as a gate on C-line builds | Rewrite to cover PEI, DXE and MM (§9) |
| Capsule build | Home-grown, hardware-proven byte splicing: 57 builders (50 `build_rev*.py`, `build_sentinel_mm.py`, 6 `rev1xx_steps.py`) among 75 scripts | 13 operation types + declarative recipes |
| Build gates | Grew over 125 revs; several are stubs that always pass | Gate framework where "not run" ≠ "pass" and every gate must be shown to fail |
| Firmware module pipeline | 13 host and QEMU gates sharing one build directory in a hand-kept order; a 710-line per-rev board-build assertion script; module bytes that change with a random stack cookie | Per-variant builds, assertions as data, artefact records (§4.12) |

### 3.2 Environment (verified on this machine, 2026-11-24)

- Python 3.14.4, `uv` available. Present: capstone 5.0.7, pyyaml, pydantic 2.12, jinja2,
  networkx, brotli 1.2.0, pytest. **Missing**: unicorn, pefile, lief.
- On PATH: iasl, objdump, r2, rizin, chipsec_util, psptool, binwalk, gcc, nasm.
  **Not on PATH**: UEFIExtract/UEFIReplace, GenFfs/GenSec/GenFw, flashrom.
- `qemu-system-x86_64` exists but fails to load (`libcacard.so.0` missing).
- `../UEFITool` is a source checkout (A74-6); `common/guids.csv` there has 12,182 GUID
  names (BSD-2-Clause).

### 3.3 Firmware images on disk

| Image | SHA-256 | Notes |
|---|---|---|
| `../sentinel-uefi/A5557.cap` | `33f80b07…` | Aptio capsule, 33,558,528 B. Same as `firmware/baseline/*`. **Provenance unclear**: v1 docs also call this SHA `sentinel-phase1-rev4` and name `1f067b99…` as the true ASUS release — to be settled in P1 |
| `../sentinel-uefi/02102026.bin` | `b61802d8…` | Raw 32 MiB SPI **board dump running rev70** (FV_MAIN digest `370de8c5…` matches the registry); contains live NVRAM |
| `../sentinel-uefi/efivars/rev_84/rom_armor/flashwin_ff000000.bin` | — | 16 MiB upper half of SPI from the board running **rev84** (FV_MAIN `cd0c39b6…`); a partial image |
| `/home/j/Videos/ROG-STRIX-X870E-E-GAMING-WIFI-ASUS-2503/…2503.CAP` | `0ed48d24…` | Vendor BIOS 2503. **Both v1 SMM PoCs are pinned to this image, not A5557** |
| `../uefi/CarbonX1.bin` | `08852717…` | 32 MiB Intel image: flash descriptor, ME 12.5 MiB, BIOS region with 23 volumes; Alder Lake-P, Lenovo ThinkPad (BIOS `N3JET41W`), Lenovo in-house codebase with no AMI/Insyde/Phoenix markers. Intel fixture — exercises "unknown IBV" too |

None of v1's built capsules (rev5–rev124; rev125 was only planned) is on disk, and none
will be (D2). A5557's FV_MAIN digest (`a0326063…`) is the registry's vendor-baseline
entry (which also lists `sentinel-phase1-rev4` as identical in the measured range).

What v1's git does still track, usable as fixtures and oracles (§8): the 108 module
bodies of its SMM static corpus (cut from rev45; all 108 occur byte-for-byte in A5557,
checked 2026-11-24); 57 FFS vectors, 43 of them in `c5_work/`, which also holds 42 of
rev124's 43 module bodies; 38 per-rev `modules.sha256` (rev86–rev124); the PCR0 and PEI digest
registries; 40 prebuilt old-line `.efi`; and nine BIOS 2503 module files (six distinct
bodies) under `reports/`.

A data point for the A5557 provenance question (checked 2026-11-24): v1's
`guid_registry.json` records eight edk2 SMM swaps with `33f80b07…` as their output. The
seven of those bodies that can be compared — they are in v1's SMM corpus, hence in
A5557 — do not match the recorded edk2 hashes, and the SMM CPU driver is present under
the vendor's GUID, not the edk2 variant's. This image therefore does not carry those
swaps; whether it is the untouched vendor release (v1 names `1f067b99…` as a
"sanitized" vendor source) is settled in P1 as a `vendor_release` fact (§4.3).

### 3.4 Facts checked against the bytes

- A5557 capsule header: GUID `4A3CA68B-…` (UEFITool: `APTIO_SIGNED_CAPSULE_GUID`),
  `HeaderSize=0x1000`, `RomImageOffset=0x1000` at +0x1C. The payload offset must be read
  from `RomImageOffset`, not assumed.
- All 8 LZMA sections in A5557: dictionary 16 MiB, props 0x5D, true size stored; no
  LZMA-F86 sections. FV_MAIN is outer FV `5473C07A…` at SPI 0x1000000, size 0xB50000.
- The PSP-loaded PEI image sits **inside** a firmware volume: FFS `3E3B6DC0` in each of
  two outer FVs holds an AMI GUID-defined section `CE3233F5` — a 0x100-byte header, zero
  except the compressed size at +0x14, then a zlib stream — and the `$BL2` type-0x62
  entries point into it. Both copies inflate to the same image, the one v1's PEI builds
  started from (`0bfe931b…`, 0x500020 B). Re-running zlib level 9 does **not** reproduce
  the vendor stream (828,184 B against the vendor's 825,097 B).
- `$FID` (tag `A5557`, date, `ALASKA A M I`) is readable at 0x1C012EC; `AGESA!` and the
  board name are inside compressed sections ⇒ identification must run after extraction.
- `unicorn 2.1.4` ships a `cp37-abi3` wheel (should install on 3.14 — to be confirmed in
  P0). Read from its source during the review, and to be confirmed by a P0 test: the x86
  instruction hook accepts only IN, OUT, SYSCALL, SYSENTER, CPUID, RDTSC and RDTSCP, so
  RDMSR/WRMSR cannot be hooked directly.
- Qiling's PyPI release is stale (1.4.6, 2023-08) although the project still tags
  releases; its SMM model is Intel-only (Intel save-state layout, software SMIs only, no
  AMD code), and it brings ~31 dependencies.

### 3.5 Known v1 bugs to fix while lifting

- `pei_research/tools/ppi_graph.py:58` drops every terminating PPI descriptor (flags `0x80000010`).
- `tools/depex/analyze.py` parses section type 0x18 (FREEFORM_SUBTYPE_GUID) as an SMM
  DEPEX and leaves out 0x1B (PEI DEPEX); 0x1C is handled. It also ignores AND/OR/NOT and
  mis-computes GUID_DEFINED offsets for 8-byte headers.
- `scan_smm_handlers.collect_files` reads large-file size at +20 instead of +24.
- `analysis/_tools/fw_walk.py` sends EFI-compressed sections to LZMA.
  `interactive_map/tools/fv_inventory.py` decodes Tiano only by shelling out to a
  `TianoCompress` binary that is no longer present, so in practice nothing decodes it.
- `smi_registry.py:220-222` skips dispatch protocols that are not PI-standard. That
  explains only `395B0181` (SW 0x55). `06BE9C49`, the real FchSmm (0xBB), has an FCH
  Locate the tracer never resolves; `34F85EE2`, the ASUS WMI back-end (0x90–0x92),
  carries no dispatch GUID at all; `271E684F` registers 0x99 only under a condition. All
  were recovered by hand, and their registry entries exist but are empty.
- `protocol_graph.py` read boot-services slot 0xA8 (RegisterProtocolNotify) in DXE
  modules as an SMM install until `call_scan.py` replaced its scanner.
- `handler_audit.py` treats SW-SMI `r8` as a CommBuffer (it is `EFI_SMM_SW_CONTEXT`); taint
  dies at joins and calls; `GetVariable` output is not a taint source.
- `fv_inventory.py` aborts a whole FV walk on one bad FFS size.
- `tests/depex_verify.py` and `tools/spi/verify_psp.py` are stubs that report success.
- `tests/build_sentinel_mm_board.sh` writes `modules.sha256` before its verdict, and the
  capsule builder takes the newest such file without checking that the board build passed.
- The crypto known-answer tests under `encryption/tools/` load firmware PE code into the
  host process through a native harness: the one place v1 runs firmware code natively.

### 3.6 Precursor projects `../uefi`, `../uefi_framework`

Both date from February 2026 and are mostly superseded; almost nothing is lifted as code.

- **PSP emulation is a sketch built on a header misparse** and never executed (trace log
  is 0 bytes). It used stock `qemu-system-arm -M virt` with nothing stubbed, hardcoded
  directory offsets, and assumed a 0xA0-byte `$PS1` header — the real header is 0x100
  bytes (ARM vector table verified at +0x100). Not a basis for `tools/psp/amd/`; PSP-stage
  emulation, if done, is built fresh on Unicorn ARM (P9).
- `psp_forensics.py` is weaker than v1's `tools/spi/psp_inventory.py` (wrong EFS offsets,
  no signature check). Ideas worth porting into the v2 PSP layout plugin: whole-image
  **gap / non-empty-padding map**, entry-overlap detection, reference-image diff with
  variable regions masked, per-entry entropy as information only.
- `uefi_framework` is plans plus unrun scaffolding. Ideas carried into v2, rebuilt
  properly: a **crypto bill of materials** per module, an **advisory matcher keyed on
  module hash/version with a sourced database** (not name substrings), a protocol-orphan
  report, and the removal-cascade simulation from `depex_analyzer.py` as a recipe pre-flight.
- Lessons confirmed: whole-firmware QEMU boot is a dead end (its own guide says so);
  byte-pattern detectors need decompressed x86 and real disassembly; "sanitising" by
  patching bytes inside modules breaks hashes — recipes replace or remove whole files.
- **Claims v2 must not inherit**: the precursor's "nation-state implant", PNG/TTF
  steganography and CsmDxe CosmicStrand conclusions do not hold up (v1's own
  `pe_entry_anomaly` found nothing). v1 never repeats the implant or CosmicStrand
  wording, but `sentinel-uefi/docs/architecture.md:34-37` still lists four of the
  precursor's items as findings: a UPX-packed PE, 408 high-entropy PNGs, a
  "steganographic" TTF font and "suspicious" NVRAM variables. `docs/LESSONS.md` records
  all of them as retracted.

### 3.7 Hygiene issues found in the existing repositories (not v2 work — for the owner)

- `../uefi/modules/key/PK.key` is an **unencrypted Secure Boot private key tracked in git
  and pushed to a GitHub remote**. If that PK is enrolled anywhere it should be treated as
  compromised.
- v1's git history contains machine identifiers: board serial and system UUID
  (`dmi.txt`), Wi-Fi BSSIDs/SSIDs (`wifi_scans.csv`), the TPM EK public key
  (`efivars_dump/`), MAC addresses and drive serials in ~25 kernel logs, two flash dumps
  with live NVRAM, and Windows attestation data.
- `CarbonX1.bin` is a **machine dump** (NVRAM log entries dated 2024-12-20), not a vendor
  image; it may hold device-unique data.
- v1's git also tracks firmware that must not be copied into v2's: `A5557.cap`, the
  `02102026.bin` dump with its live NVRAM, about 150 module bodies
  (`smm_research/modules/`, `edk2-modules/`, `reports/`), and investigation notes that
  quote board identity data.
- v1's `firmware/baseline/MANIFEST.md` describes chip recovery by writing the
  33,558,528-byte `.cap` to a 32 MiB chip, and calls the platform "PSB-hardened"
  although the PSP reported `boot_integrity` = 0 (§4.11 defines the recovery artefact).

v2 is designed so none of this class of data enters its git history (§4.7 privacy; the
P0 hygiene tests of §8).

## 4. Architecture

### 4.1 Repository layout

```
run.py                       thin entry: sentinel.cli.main()
pyproject.toml               uv-managed; deps: pyyaml, pydantic, capstone, brotli, unicorn, cryptography
CLAUDE.md, .claude/          project agent doctrine and harness rules (P0, §6)
sentinel/                    shared core library (generic PI/UEFI; no vendor knowledge)
  cli.py workspace.py doctor.py profile.py hashing.py guid.py diff.py log.py
  fmt/        capsule fv ffs section pe(PE32/PE32+/TE, Authenticode) depex esl hob magic
              compress/{lzma_bcj, tiano, brotli_, ami_zlib, registry}
  ingest/     dropzone, sniff, index
  extract.py  the single walker → model + tree
  model/      schema, store, graph, dispatch (PEI+DXE+MM evaluator, cycles), observed
  names/      resolver, providers, harvest, importers/{edk2, uefitool, v1}
  plugins/    api, loader, contrib
  pipeline/   dag, runner, worker (subprocess isolation), cache
  findings/   schema, overlay, coverage, report
  evidence/   bundle schema, attach, override rules                      (§4.7)
  disasm/     trace (from v1 trace.py), callscan, guid_refs, cfg, provenance (gBS/gSmst by origin)
  emu/        engine, loader, memory, stubs, layers/{pei,dxe,smm}, devices, hunters, coverage
  build/      recipe, lineage, editor, ffs wrap, compress, ops, gates
  map/        build.py, static/{index.html, assets/{core.js, style.css, views/*.js}}
tools/<phase>/{common,amd,intel,ami}/[<oem>/]     plugins                (§4.2)
uefi/<phase>/<compat>/…      Sentinel firmware module sources            (§4.12)
collect/                     read-only capture kit run on a live board   (§4.7)
recipes/<board>/*.yaml       build recipes
data/        vendors.yaml, profiles/<id>.yaml,
             boards/<board>/{images,assertions,known_bad,pcr0,recovery}.yaml + generated images.built.json,
             guids/{edk2,vendor,learned}/, names/overrides.yaml, annotations/global.yaml,
             catalogues/ (sourced lists, each with its source and date: advisories, Secure Boot
             objects and revocations, pattern rules, vendor releases)
firmware_dropzone/           input (contents git-ignored)
evidence_dropzone/           evidence bundles in (contents git-ignored)
evidence/<board>/<bundle>/   normalised bundles, each stored once (git-ignored, §4.7)
firmware/                    workspaces, _rejected/, _cache/ (the only clean-up target),
                             _artefacts/ (built uefi/ modules by SHA, §4.12), index.json (git-ignored)
ext/edk2/                    pinned upstream checkout made by `edk2 bootstrap` (git-ignored)
docs/        LESSONS.md, operator-loop.md, architecture.md, plugin-authoring.md, findings-doctrine.md, safety.md
tests/       unit/, synth/ (fixture generators), golden/<fixture>/, regression/ (emulation specs),
             oracles/v1/ (frozen v1 outputs and check functions), fixtures.yaml, fixtures/bin/ (git-ignored)
```

Hand-written data is YAML validated by pydantic; generated data is JSON.

### 4.2 Plugin model

Directory = placement convention; applicability is declared and checked.

```
tools/<phase>/common/             generic PI/UEFI
tools/<phase>/amd|intel/          silicon-specific
tools/<phase>/ami/                firmware-vendor (IBV) specific; insyde/, phoenix/ later
tools/<phase>/<silicon>/<oem>/    OEM extension of a silicon dir   (tools/smm/amd/asus/)
```

Phases: `ingest layout identify nvram hii acpi oprom crypto image sec pei dxe smm psp
platform evidence build gates`. `platform/` holds phase-neutral device models (MSR,
CPUID, FCH/PCH MMIO) used by emulation. One extra vendor folder exists only under
`tools/evidence/`: `sentinel/`, for decoders of telemetry published by the Sentinel
firmware modules in `uefi/` (they share an ABI and move together, §4.12).

A plugin is one Python file exporting `PLUGIN` and an entry function:

```python
@dataclass(frozen=True)
class Match:  silicon="*"; ibv="*"; oem="*"; board: str|None=None; require: tuple[str,...]=()
@dataclass(frozen=True)
class PluginSpec:
    phase: str
    kind: Literal["sniffer","decoder","identify","analyzer","hunter","importer","op","gate","emitter"]
    applies: Match = Match()          # may only narrow what the path implies
    needs: tuple[str,...] = ()        # "model:modules", "out:smm.common.registry", "profile:cpu.family"
    produces: tuple[str,...] = ()
    extension_points: dict[str,type] = {}
    replaces: str|None = None; requires_py=(); requires_bin=(); version="1"; timeout_s=600
def run(ctx: Context) -> Result
class Status(Enum): NOT_APPLICABLE; RAN; COULD_NOT_RUN; ERROR; SKIPPED_UPSTREAM; CACHED
```

- **Vendor extensions are mostly data.** A consumer declares an extension point
  (`smm.dispatch_protocols`, `smm.sw_smi_decoders`, `emu.msr_models`); any directory may
  hold `*.contrib.yaml` contributing entries; `ctx.contrib()` merges common → ami → silicon
  → OEM → board with provenance. `tools/smm/amd/fch_dispatch.contrib.yaml` declares the
  FCH dispatch protocols as data. That closes only part of v1's registry gap (§3.5): one
  module's FCH Locate is not resolved by the tracer, one carries no dispatch GUID and one
  registers only under a condition, so the registry also needs better Locate resolution,
  wrapper-aware dispatch (§4.9) and the emulated registration census (P5a).
- **Three outcomes never collapse.** *Not applicable* (match failed), *could not run*
  (missing dependency, unknown profile field, undecodable input, upstream failure), *ran*.
  `RAN` with an empty coverage list is recorded as `RAN_NO_COVERAGE` and reported as
  "unknown", never "clean" — the guard against v1's unrun-scan-reads-as-clean failure.
- **Isolation.** Each analyzer runs in a subprocess (`python -I -m sentinel.pipeline.worker`)
  with timeout and `RLIMIT_AS`; a crash is an `ERROR` for that plugin only.

### 4.3 Ingest and identification

- Dropzone scan: top-level regular files, any extension; files still being copied
  (size/mtime changed in the last 3 s) are skipped until stable. File names are untrusted
  input: the workspace name is the sanitised stem (control characters, separators and
  leading dots removed, length capped).
- Sniffers score the buffer; the best unwraps it and sniffing repeats (container chain,
  max depth 4). None uses a fixed image size.

| Sniffer | Test |
|---|---|
| `ingest/ami/aptio_capsule` | Aptio GUIDs at 0; sizes consistent; payload at `RomImageOffset` |
| `ingest/common/efi_capsule`, `fmp_capsule` | `EFI_CAPSULE_HEADER` with a catalogued GUID; FMP payload items |
| `ingest/amd/efs_spi` | `AA55AA55` at candidate EFS offsets **with field checks** (pointers land on `$PSP`/`$BHD`) |
| `ingest/intel/ifd_spi` | `5AA5F00F` at 0x10 / 0x0, FLMAP0 sane |
| `ingest/common/bare_fv` | `_FVH` at +0x28 with sane lengths |

- Identification runs **after** extraction; each fact is `{value, confidence, evidence[]}`
  with `conflicts[]`: silicon (EFS/PSP vs IFD), IBV (Aptio GUID, NVAR store GUID, `$FID`),
  OEM/board (`$FID` tag, SMBIOS defaults), AGESA version; microcode patch level per
  container; versions of PSP components and trustlets; embedded crypto-library versions
  (from the crypto census); default key stores (PK/KEK/db/dbx containers: hashes and
  counts); the vendor signature (whether the Aptio header chain verifies over the range
  it covers; payload coverage unknown); Platform Secure Boot indicators (BIOS-directory
  OEM-key and RTM-signature entries, and `boot_integrity` when evidence supplies it);
  and `vendor_release`, a match against `data/catalogues/releases/<vendor>.yaml`
  (version, date, source, capsule SHA), which is how the A5557 provenance question is
  settled. v1 tracked exposure by microcode, AGESA and trustlet versions, by hand.
- **Platform profile** (`data/profiles/*.yaml`) supplies what a capsule cannot: CPU family
  and SoC gates, default Setup/PCD values, boot mode, SMRAM assumptions. Most specific
  match wins; no match → `generic-<silicon>` with every field `unknown`, and plugins that
  need a gate report *could not run* (never assume both per-family twins are live). The
  resolved profile is frozen as `profile.lock.yaml`; its digest is in every cache key.

### 4.4 Extractor, workspace and model

One walker (`sentinel/extract.py`) merging v1's `interactive_map/tools/fv_inventory.py`
recursion with `encryption/tools/fwcorpus.py`'s on-disk dump, fixing: resync after a bad
FFS size; GUID_DEFINED data offset from the header; LZMA-F86 (pure-Python BCJ); Tiano/EFI
decompression (pure-Python port); Brotli; CRC32 sections; TE images; FFS3 large files;
and the AMI GUID-defined section `CE3233F5` (`compress/ami_zlib`: a 0x100-byte header
with the compressed size at +0x14, then a zlib stream). On Aptio AMD images that section
is the PEI container: it sits inside FFS `3E3B6DC0` of an outer FV, in two copies that
must inflate to the same image, and the AMD layout plugin links each `$BL2` type-0x62
entry to the section it points into. A second pass walks only roots that layout plugins
find genuinely outside any FV. Every PE/TE image, whether from an FV or an option ROM,
records its file SHA-256, its Authenticode SHA-256 and SHA-1, and whether it is signed
(v1's measured driver events and its option-ROM allowlist name images by Authenticode
digest, not by file hash).

Firmware is untrusted input. Declared sizes are checked before allocation, recursion
depth and total extracted bytes are capped, names taken from the image (UI sections,
variable names) are sanitised before they become paths, and decoders run isolated (§4.2).

```
firmware/{name}_{datetime}/
  manifest.json          sha256, size, original_name, container_chain, parent, same_image_as
  input/<original file>  + SHA256SUMS            (read-only)
  image/payload.bin      unwrapped image; capsule_header.bin
  profile.lock.yaml
  blobs/<sha[:2]>/<sha>  content-addressed bodies
  extract/<region>/<fv>/<NNN>_<GUID>_<Name>/…    browsable tree (hardlinks into blobs)
  modules/<GUID>_<sha8>/ image.efi|.te, manifest.json, strings.txt
  model/                 image regions volumes files sections modules census graph names identity observed
  analysis/<plugin-id>/  result.json + outputs
  emu/                   per-module run records, coverage, proposed plugin data
  evidence/<bundle-id>.json  binding to this image (match method, confidence); the bundle
                         itself is stored once in top-level evidence/     (§4.7)
  findings/              findings.jsonl, coverage.json, summary.md
  overlay/               annotations.yaml, names.yaml, profile.yaml       (hand-edited)
  map/                   index.html, assets/, data/*.js
  build/<recipe>_<datetime>/                                              (§4.11)
  logs/  state/
```

Model files carry `schema_version`; ids are paths (`fv@0x1000000/f12/s2/fv0`).
`regions.json` lists unclaimed byte ranges explicitly. `graph.json` edges carry
`role` — depex_requires, installs, locates, notifies, signals / waits (event group),
config_table, communicates (an MM Communicate header GUID → the handler registered for
it), loads_file (a FILE_GUID named by a loader), names_variable (name + GUID, ASCII or
UTF-16, from decoded bodies), references — `method` (depex, literal, immediate (a GUID
assembled on the stack from immediates, which a 16-byte search misses), trace, emulation,
measured), `tier` and `confidence`. The core emits only DEPEX and reference edges because
byte-proximity scans confuse install with locate; trace edges come from `callscan`
(§4.9), and unresolved call sites and GUID arguments are counted per module into
coverage. Static, emulated and measured facts live in separate layers and never
overwrite each other.

`model/dispatch` lists its **assumed providers**: a protocol that some DEPEX needs but no
traced installer provides is assumed present and shown as such (view 4). SoC gates
declared by the profile are never assumed, the rule v1's dispatcher model follows. A
disagreement between the model and observed dispatch is a finding about the model.

### 4.5 Pipeline and CLI

Stage order: ingest → extract → layout → extract (pass 2) → identify → profile → names →
index → evidence attach → static analyzers → findings → **map** → emulation → findings →
**map**. The map is rendered as soon as static results exist and refreshed after
emulation, `evidence add` and `annotate`: that keeps the order asked for in §2 (analyse →
map → emulate → build) and puts a page in front of the user before slow emulation
finishes. Build runs on demand. The DAG comes from `needs`/`produces`; cache key = plugin
source hash + inputs + profile digest + contribution digests.

```
run.py [process] [--file F] [--profile ID] [--only S…] [--until S] [--quick|--deep] [-j N]
run.py rerun <ws> [STAGE | --from STAGE] [--force]
run.py list | show <ws> [--gates] | plugins [--for <ws>] | map <ws> [--open]
run.py findings <ws> [--coverage | --across F]
run.py diff <wsA> <wsB>          run.py watch
run.py evidence add <ws> <bundle> | evidence list
run.py emulate <ws> [--module G] [--hunter H]
run.py build <recipe.yaml> [--variant V] [--plan]
run.py check <ws> --recipe R     R's gates and assertions on any ingested image (§4.11)
run.py annotate <ws> <finding> --status S --reason … [--evidence P] | --note …
run.py guids import … | guids harvest
run.py edk2 bootstrap            fetch pinned EDK2 + apply uefi/patches   (§4.12)
run.py uefi build [--platform P] [--module M] | uefi gates [--only G]     (§4.12)
run.py doctor [--json]
```

`--quick` stops before emulation; `--deep` adds the fuzzing hunters. `doctor` reports
every capability and which stages it disables (today: unicorn missing, BaseTools missing,
qemu broken). Exit codes: 0 all ran, 1 partial, 2 usage, 3 everything rejected, 4 core
environment broken. Findings never change the exit code unless asked.

Every read command takes `--json` (schema-versioned). Read commands write nothing outside
`firmware/<ws>/`; `data/` changes only through explicit import or harvest commands, which
print a diff. (Two of v1's "checkers" rewrote the registries they were run to check.)

**Ground truth, one command each.** An agent runs the command and quotes its output; it
never infers the answer.

| Question | Command |
|---|---|
| What is this file, and which workspaces hold the same image? | `show <ws> --json` |
| Does image X carry change Y? | `check <ws> --recipe R --json` |
| What differs between two images? | `diff <wsA> <wsB> --json` |
| Which images can this evidence bundle come from? | `evidence list --json` (candidates, method, confidence) |
| Is this built capsule's verdict complete? | `show <child> --gates --json` (verdict, `not_run`, waivers, refusals) |
| What was not checked? | `findings <ws> --coverage --json` |
| What can this host run? | `doctor --json` |

### 4.6 Findings, coverage and overlays

v1's doctrine becomes the schema: **every automated result is a lead, never a verdict.**

- Finding: `id` (stable fingerprint of rule + module SHA + normalised location), `class`,
  `subject`, `location` as (module SHA, RVA), `tier` (behavioral / descriptive /
  structural), `confidence` (certain / likely / unknown), `evidence[]`, `fidelity[]`
  (models, phantoms, unmodelled hardware on the path), `sensitivity`, `status = "lead"`.
- Corrections go in overlays (`overlay/annotations.yaml` per workspace,
  `data/annotations/global.yaml` keyed by module SHA so they follow identical modules into
  other capsules). Generated files are never hand-edited. Overlay status is set by a
  human: `lead` (initial), `confirmed(static | emulated | measured)`,
  `dismissed(false_positive | not_reachable | by_design | mitigated)` or `accepted`, each
  with a reason and an evidence path. The global overlay is seeded from v1's "considered,
  not reported" list and from its triage of a third-party scan (27 of 37 reported IDs
  were false positives), so settled leads are not triaged again. There are no disclosure
  states: reporting is outside v2 (§2.1).
- `coverage.json` is a plugin × subject matrix; the report and the map list everything
  that was **not** checked: plugins not applicable / could not run / errored, undecodable
  sections, modules never emulated, profile fields left unknown.
- **Analysis horizon.** Static analyzers report, per subject, what they could not follow:
  taint leaving the analysed code, unresolved call or install sites, stack-built GUIDs.
  Above a threshold the subject is `partial` in `coverage.json`, the static counterpart of
  §4.10's coverage threshold: "nothing found" always has a stated scope.
- **Siblings.** A finding raises linked *sibling* leads on code clones (`clones.py`, §4.9)
  and on identical bodies in every workspace and known-but-absent image record;
  `run.py findings <ws> --across F` reports identical / differing body / absent. (v1
  found a second and third instance of one known-vulnerable accessor this way.)
- **Sensitivity.** Findings and overlay entries are `public` or `embargoed`; `map/` and
  `summary.md` leave embargoed items out unless asked, and embargoed overlay text stays
  out of tracked files. Key-material plugins emit fingerprints, offsets and sizes, never
  key bytes; a test fails if a private key appears under `analysis/`, `findings/`, `map/`
  or in any tracked file.

### 4.7 Board evidence — first-class

v1 keeps board evidence in 26 `efivars/rev_*` directories and, beyond them, in
`smm_live_audits/` (the only measurements for rev82–85), `hob_analysis/`, `census/`,
`efivars_dump/`, `pei_research/rev125/evidence/` and the `board/` and flashed-review
directories under `smm_research/studies/`; the two `efivars/store_*` directories are
key-store snapshots, not captures. Layouts differ and there is **no common manifest**
(newer captures carry `captured_utc.txt`, `c6_state.json` and, from rev101,
`capsule_sha256.txt`). Identity is a rev number in a folder name; a measured FV_MAIN
digest exists from rev82, saved binary event logs only from rev97. Many captures hold
text already decoded by v1's readers rather than raw records. v2 normalises this.

**Bundle.** `evidence/<board>/<boot_utc>_<fvmain8>/` with `manifest.json`, `raw/`,
`decoded/`, `redacted/`. Manifest fields: `bundle_id`, `schema_version`, `captured_utc`,
`boot_id`, `boot_kind` (cold / warm / after another OS / first after a flash; claimed),
kernel/OS, lockdown and Secure Boot state, `virtualized` (such a capture never binds by
measurement and is never a reference), `board_profile`, capture tool + version, and

- `identity`: `pcr0_live`, `pcr0_replay_match`, `blob_events[]` (description, base,
  length, digest), `candidate_images[]` (capsule SHA, match method), `confidence`
  (**measured / claimed / inferred**);
- `items[]`: type, path, SHA-256, source interface, decoder + version, `read_status`
  (ok / absent / denied_lockdown / no_decode_all_ones / short_copy, checked against the
  decoder's record size), `about_boot` (this / previous / last OS / dump N: such an item
  describes another boot and binds to that boot's image, not to the capturing one),
  `payload_required` (true when it only exists because a telemetry module was flashed),
  `sensitivity` (public / identifier / secret);
- `active_actions[]`: anything during capture that was not a pure read.

A denied or all-ones read counts as *not run* in coverage, never as a negative result.

**Attaching to an image — generic, with board knowledge as data.** The TCG event log's
`EV_EFI_PLATFORM_FIRMWARE_BLOB(2)` events are self-describing (description, base, length).
v2 hashes each described range in every ingested image under two interpretations of
`base` (SPI offset; memory-mapped address) and records matches; a partial image (a
flash-window read) carries its origin so that ranges are translated first. Further match
methods: `module_digest` (an event digest equals a module's SHA-256 — Sentinel's MM
loader measures each MM image), `container_digest` (the PEI-container events that v1's
images extend into PCR0 from rev124) and Authenticode digests for the driver and
application events of PCR2/PCR4. Event logs are read in binary and in `tpm2_eventlog`
YAML form. The board profile says which interpretation applies and which events identify
the image versus runtime. **What is measured depends on the image, not only on the
board**: the vendor image measures FV_MAIN but neither the PEI volume nor NVRAM, while
rev124 measures both PEI containers itself. Because unmeasured regions mean several
images share one digest (v1's registry records such collisions, and every PEI-only
variant shares its FV_MAIN digest with its sibling), a bundle binds to a **set of
candidate images**, never silently to one.

**Known-but-absent images.** Every v1 capture of a Sentinel rev was taken on a capsule
that is not on disk (D2). v2 keeps image records keyed by capsule SHA and measured digest
in `data/boards/<board>/images.yaml`, seeded from v1's 66-entry `pcr0_registry.json`,
from `pei_registry.json` (container and stream digests, inflated SHA, build id) and from
the per-rev `modules.sha256` of v1's board builds, so that module-digest events bind too.
Those bundles import now and bind when an image is ingested. Attachable today: the rev84
measurement (`smm_live_audits/runs/2026-10-27`: a YAML event log and `audit.json`;
`efivars/rev_84` itself holds no event-log file) against the rev84 flash-window image,
and the rev70 dump — by claim only, since no rev70 capture carries a measurement.

**Importer plugins** (`tools/evidence/…`), each producing facts in the model's `observed`
layer with a join key:

| Evidence | Importer | Join key | Needs payload? |
|---|---|---|---|
| TCG event log, PCRs | `common/tcg_log` (from `tools/security_state/tcg_log_events.py`; binary and `tpm2_eventlog` YAML) | digest ↔ image range, module, PEI container, Authenticode | no |
| Windows measured-boot log | `common/wbcl` (from `analysis/_tools/wbcl_decode.py`) | as `tcg_log`; `about_boot` = that Windows boot | no |
| v1 `smm_audit.py` JSON (rev82–85) | `common/smm_audit_json` | FV_MAIN blob digest | no |
| Secure Boot stores, mode bytes, MOR, variable listing | `common/uefi_vars` | variable name + GUID | no |
| Live ACPI tables | `common/acpi_live` | table signature + OEM table id; diffed against the image's own tables | no |
| Runtime map, Memory Attributes Table | `common/memory_map` | address range | no |
| Kernel log, reset reason, DMI | `common/host` (redacted on import; a decoded reset status carries a "value is not cause" caveat) | — | no |
| MSR snapshot, PSP sysfs, FCH and IOMMU registers | `amd/security_state` (from `read_security_state.py`, `c6_validate.py`) | MSR number, MMIO address | no |
| Flash-window / SPI dump | re-ingested as an image (with its origin), linked to the bundle | — | no |
| HOB list | `common/hob` | HOB GUID + address range | yes |
| PEI dispatch trail | `sentinel/pei_trail` | CRC-32 of the 16 GUID bytes, resolved against the bound image's PEIM inventory; unresolved keys are kept | yes |
| MM audit / census ("did not load") | `sentinel/mm_audit` | full FFS GUID (v1 compares full GUIDs but prints 8-digit prefixes: text imports are resolved against the bound image and ambiguity is flagged) | yes |
| Crumbs, boot dumps | `sentinel/crumbs`, `sentinel/boot_dump` | `about_boot` = previous / last OS / dump N | yes |
| Older record layouts (shield, census and audit versions of the vendor line) | `sentinel/legacy/*` | as the current form; needed to convert the rev73–85 captures | yes |

**Override rule.** Static, emulated and measured facts are separate layers. Where they
disagree the measured fact wins *for that board and boot*, the disagreement is reported
as its own finding class (e.g. "statically 'runs', never dispatched"), and the caveats
travel with it (one boot only; APRIORI modules run before a trail listener exists, so
absence there proves nothing).

**Absence rule.** A missing record, event or trail entry counts as evidence only when its
producer's own "ran" mark is present; otherwise it imports as *unknown*. Which Sentinel
records to expect is derived from the bound image (each `tools/evidence/sentinel` decoder
declares its producer FILE_GUID), never from revision thresholds as in v1's reader. Per
image, the observed layer aggregates its bundles ("not dispatched in k of k boots"), and a
single bundle's absence keeps the one-boot caveat. v2 reports observed facts and how they
disagree with static and emulated ones; it does not judge whether a boot or a build
passed (§2.1).

**Capture kit** (`collect/`): one script producing a bundle with the generic items using
stock Linux tools (TCG log, PCRs, efivarfs listing and stores, ACPI tables, runtime map,
kernel log, DMI, `tpm2_getcap`, MSRs; AMD extras; the memory-mapped flash window where
lockdown allows, which becomes a partial image with its origin). efivarfs files are read
whole with a single read (v1 found that copying them could leave an empty file), every
item is size-checked against its decoder, and a virtualisation probe sets `virtualized`.
Strictly read-only by construction.
**Excluded from v2 entirely**: v1's deliberate SMI test scripts (they write `/dev/mem` and
fire SW SMIs), the NVRAM pruner's `--execute` mode, and key-enrolment scripts. Results
those scripts already saved import as evidence with `active_actions[]`.

**Privacy.** `raw/` never enters git and never reaches a generated page. Import redacts
serials, UUIDs, MAC addresses, SSIDs/BSSIDs, hostnames (shell prompts in captured text
included), user names and home paths, volume serials, personal certificate subjects, the
TPM EK and disk GUIDs; only `redacted/` and derived facts are publishable. Windows logs
and boot-partition manifests default to `identifier`. An ingested image that is itself a
machine dump (live NVRAM, SMBIOS serials: `02102026.bin`, `CarbonX1.bin`) is marked
`live_state`, and generated pages show its variable names and sizes, not values, unless
asked. `evidence/`, `firmware/` and both dropzones are git-ignored; a test fails if a
tracked file matches identifier patterns.

### 4.8 Interactive map

Self-contained, offline, opens over `file://` (classic scripts, lazy-loaded data shards;
no framework). Lifted from v1's `interactive_map/assets/app.js`: hash routing, theme,
drawer, search. Views:

1. Overview — identity with evidence, profile, container chain, counts, coverage, top
   leads, and the source of each datum (plugin or evidence item; for a measured fact,
   which boot)
2. Flash layout — derived from `regions.json` (nothing hardcoded), nested FV tree, unclaimed space
3. Modules — table with phase, name source, PE hardening, signature, clone family, findings
4. Boot timeline and dispatch — lanes PSP → SEC → PEI → DXE → MM → BDS →
   ExitBootServices → runtime from the emulator's phase clock (v1's map led with this
   view); per module the lifecycle events it references (references, not proof),
   unsatisfied DEPEX, cycles, assumed providers and profile assumptions; a module
   observed never to dispatch shows the missing DEPEX term and who holds it
5. Dependency graph — layered ego graph (1–2 hops) across PPI / DXE / MM; edge style = evidence tier
6. SMI surface — type, key, module, handler, ACPI / WMI caller, reachability class,
   static-vs-emulated registration difference, leads
7. NVRAM variables with referencing modules and class · 8. Setup/HII questions joined to
   offset, default and, where evidence exists, observed value
9. Findings with overlay status and coverage · 10. Emulation results
11. Evidence — measured vs static differences
12. Diff against another workspace (parent vs built child, or any two) · 13. Glossary
14. Crypto and keys — algorithms and libraries per module, key material by fingerprint,
    key stores with certificate validity, known-answer-test results, parent-vs-child delta

### 4.9 Static analysis plugins (lifted from v1)

| Plugin path | From v1 | Action |
|---|---|---|
| `sentinel/disasm/trace.py` | `smm_research/tools/trace.py` | lift |
| `sentinel/disasm/callscan.py` | `smm_research/tools/call_scan.py` | lift: slot meanings chosen by FFS file type (boot services / MM / the traditional-SMM mix), offsets re-checked against the pinned edk2 headers; records installs, locates, notifies, event groups, configuration tables and one level of wrapper, and counts unresolved sites. The source of every trace edge (§4.4) |
| `sentinel/disasm/guid_refs.py` | `smm_research/tools/file_guid_refs.py`, the reference scans of the vendor-census study | new primitive — who names a GUID: 16-byte literal, stack-built immediates, RIP-relative address, traced argument; the method is recorded on each hit |
| `tools/smm/common/registry.py` | `smi_registry.py` | lift; dispatch table becomes contributed data; Locate resolution and wrapper-aware dispatch added (§3.5) |
| `tools/smm/amd/fch_dispatch.contrib.yaml` | — | new: the seven FCH siblings of the PI dispatch protocols installed by the dispatcher `4E348EF9`/`D6CC7A4D` (`881B4AB6`, `87E2A6CF`, `A365240E`, `736102F1`, `FBBB2EA9`, `7051AB6D`, `91288FC4`) plus the unpaired `13BD659B` |
| `tools/smm/common/{protocol_graph,handler_audit}.py` | same names | lift graph (on `callscan`); rewrite audit taint |
| `tools/smm/ami/` | AMI SMM core layout, RT shim `395C33FE`, NvramSmm rules, the ACPI-mode module `750890A6` (SW 0xA0/0xA1, with its OEM hook table), `6B35FEF5` (0xDA), the SwDispatch wrapper `E53734A3` | parameterise as data |
| `tools/smm/amd/asus/` | `7627B2DE` (SW 0x96), `875486C2` (0x98), `271E684F` (0x99, conditional), the WMI back-end `34F85EE2` (0x90–0x92) | data + decoders |
| `tools/image/common/consumers.py` | `smm_research/tools/comm_consumer_scan.py`, the `caller_check` and `second_look` readers of the vendor-census study | new: for a set of modules, every kept consumer of what they install, of their MM Communicate GUIDs and of their FILE_GUIDs, with the code after each Locate classed status-tested / ignored / unchecked. v1's triage verdicts become overlay data. Feeds the removal gates (§4.11) |
| `tools/{smm,dxe}/common/surface.py` | `minimal_closure.py`, `reachability_survey.py`, `tools/depex/triage_*.py` | new, leads only: installs nobody consumes; loaded modules with no registration and no consumer; modules outside the closure of declared seed modules; code callable after ExitBootServices; the profile-inactive twin of a per-family pair. Each lead carries its consumer evidence |
| `tools/image/common/clones.py` | `smm_research/tools/code_similarity.py` | rewrite: normalised per-function hashes, per workspace and across the cache; drives sibling leads (§4.6) |
| `model/census.json` (one record per module SHA, computed once) | `bs_census`, `analysis/_tools/{pe_review,event_sites,rt_calls}.py`, `iommu/tools/audit_iommu.py` | new: service-call histogram, writes into the service tables, event types and GUIDs, port I/O with the port resolved against a data catalogue, W+X sections. A lone linear-sweep hit stays a lead until a function boundary confirms it |
| `tools/acpi/common/aml_smi.py` | `smm_live_audits/smm_audit.py --from-dsl`, `aml_smi_callers.py` | lift |
| `tools/acpi/common/{tables,wmi_surface}.py` | v1's `acpi/` review and `SMI_CMD_INVENTORY.md` | new: WSMT and FACP flags against the fixed buffers named in AML; `_WDG` and BMOF decode; AML methods that reach MSR / port / PCI / EC; joins SMI command → AML caller → WMI method → handler module for view 6 |
| `tools/dxe/common/{bs_census,bs_table_map,win_strings}.py` | `bootservices/tools/*` | lift; `bs_table_map`'s ExitBootServices sequence becomes data for the DXE emulation layer |
| `tools/dxe/amd/iommu_posture.py` | `iommu/tools/*` | lift: IVRS, IOMMU-protocol producers and consumers, users of the gating PCD (`pcd_db` gains getter and setter sites) |
| `tools/pei/common/ppi_graph.py`, `pcd_db.py` | `pei_research/tools/ppi_graph.py`, `iommu/tools/decode_pcd_db.py` | lift, fix bug |
| `tools/pei/common/hob_checks.py` | `hob_analysis/*/analyze.py` | lift, over observed HOBs: SMRAM against TSEG, MM-input HOBs, resource gaps, duplicate or zero GUIDs |
| `tools/pei/amd/` | AGESA chain; CPUID family gates, decoded into a per-module `applies_to_family` and twin families; runtime PPI installers as contributed data; 0x62 container | parameterise as data |
| `tools/layout/amd/psp_dirs.py` | `tools/spi/psp_inventory.py` | parameterise (walk from EFS, no fixed offsets) |
| `tools/nvram/ami/nvar.py` | `read_nvram.py`, `nvar_dump.py` | rewrite: locate by store GUID |
| `tools/nvram/common/var_refs.py` | the variable-name scan of `build_smm_map.py`; `tools/sentry/classify.py` | new: `names_variable` edges from decoded bodies, nested compression included; v1's read-only variable classes (never-touch, tiers, memory-training cache) become contributed data; used by view 7 and by the `ami.edit_nvar` guard |
| `tools/hii/common/ifr.py` | `tools/hii/ifr_export.py` | lift |
| `tools/hii/common/setup_map.py` | `analysis/_tools/{ifr_map,nvar_value}.py`, `iommu/tools/{hii_find_question,nvar_defaults}.py` | new join: question → varstore, offset, width → default byte in the NVAR defaults store → file offset → observed value; flags variables that AMD CBS builds from its own defaults |
| `tools/crypto/common/{census,key_material,esl}.py` | `encryption/tools/{crypto_census,canon,key_material_scan,esl}.py` | lift |
| `tools/crypto/common/keystore_audit.py` | `encryption/tools/secureboot_store.py` | lift: default PK/KEK/db/dbx located by content; per certificate the algorithm, key size and validity against a reference date; test or untrusted platform keys; dbx compared with a sourced catalogue; baked versus observed stores through the override rule |
| `tools/crypto/common/kat.py`, `tools/crypto/ami/`, `tools/crypto/amd/asus/` | `cryptodxe_kat`, `ami_digsig_kat`, `pss_probe`; `fw_key_compare` | re-host the known-answer and forgery tests on `sentinel/emu` in harness-entry mode (P5d), since v1 ran firmware code natively for them; parameterise the rest |
| `tools/image/common/{pe_hardening,entry_anomaly}.py` | `pe_entry_anomaly.py` | lift, add TE |
| `tools/oprom/common/`, `tools/oprom/amd/atom.py` | `fw_walk.find_oproms`, `atom_disasm.py`, `analysis/oprom-*` | lift; scan every leaf section, raw sections of non-PE files included (how v1 found a fourth ROM); per ROM: container chain, IDs, code type, checksum and length, signature and Authenticode digest of an embedded EFI image, file-GUID readers and override modules |
| `tools/image/common/cve_patterns.py` | hand-applied byte/instruction patterns | new, catalogue-driven: an own evaluator for a public rule schema (FwHunt's; its rule corpus was CC0-licensed when checked on 2026-11-24, re-checked per rule on import), with rules anchored to the SMI registry rather than scanned image-wide |
| `tools/image/common/exposure.py` | v1's hand-kept exposure ledger (`smm_research/plans/03-cve-remediation-2026-10.md`) | new: version- and hash-keyed matching of the identity facts (§4.3) against `data/catalogues/advisories.yaml`; each entry carries its source, date and a match level (confirmed / inferred / class) |

Dropped: Windows-driver analysis trees; per-rev narrative (per-module verdicts are
imported by SHA, §4.13); v1's hand-written lineage keys as code (P6 maps each to an
assertion, guard or gate, §4.11).

### 4.10 Emulation harness and hunters

**Engine.** Own harness on Unicorn 2.1.4 behind an `Engine` interface (IA32 + x64), with
the CPU model pinned. Qiling rejected (Intel-only SMM model with no Communicate path,
about 31 dependencies, a stale PyPI release — §3.4); ideas borrowed. Whole-image boot is impossible (signed PSP/AGESA stage), so emulation is
per module and per handler under modelled services. Firmware never runs natively.

**Core components** (`sentinel/emu/`):

- Loader — PE32+, PE32, TE (offset = RVA − StrippedSize + 0x28), relocation types
  0/1/2/3/4/10. Every reported address is (module SHA, RVA).
- Memory — region kinds (SMRAM image/heap/stack, save state, BS/RT data, ACPI NVS, COMM,
  flash, MMIO, canary, guard) with a policy per (kind, CPU mode, boot phase); non-SMRAM is
  non-executable while in SMM.
- Stubs — declarative service tables; unfilled slots auto-stub as `UNIMPLEMENTED` and are
  counted; callbacks via continuation frames; every call logged.
- Devices — port I/O, MSR, MMIO, CPUID, PCI config; unmodelled access is recorded and any
  lead that depended on it is footnoted. Port I/O, CPUID and RDTSC use Unicorn's
  instruction hooks; MSR access uses per-site hooks planted from a capstone pre-scan of
  each loaded module, because the instruction hook does not cover RDMSR/WRMSR (§3.4).
- Determinism — RDTSC = instruction count, seeded RDRAND, fixed time, bump allocator;
  identical inputs give byte-identical run summaries.
- Snapshots — load and run the entry once, snapshot after ReadyToLock and after
  ExitBootServices, restore per handler case.

**Environment layers**, driven by one phase clock (`PEI → DXE → END_OF_DXE →
SMM_READY_TO_LOCK → EXIT_BOOT_SERVICES → RUNTIME`):

- SMM/MM — gSmst, SMM protocol database, SmmAccess2/Control2/Communication/Cpu, PI dispatch
  protocols as core models, vendor dispatch protocols as plugin data, CommBuffer and
  SW-SMI trigger paths, save state in the layout the silicon plugin declares. Dispatchers
  not known to any plugin are detected and written to `emu/proposed_dispatch_protocols.json`.
- DXE — gST/gBS/gRT/gDS, handle and protocol database, events that actually fire, a
  variable store seeded from the image's NVRAM defaults, PCD database, emulated dispatch
  in computed order with per-module rollback.
- PEI (IA32) — PEI Services, PPI database with notify, HOB list (its migration at
  permanent memory is modelled, so a stale HOB pointer is flagged), PCD PPI, FFS services,
  and a new PEI dispatch simulator. PPI installers that the static scan misses are
  written to `emu/proposed_pei_installers.json`.
- Vendor contributions: `tools/platform/amd/` (CPUID per profile, HWCR SmmLock,
  SMM_ADDR/MASK, FCH ACPI-MMIO, PSP mailbox), `tools/smm/amd/` (FCH dispatch, AMD save
  state), `tools/smm/ami/` (AMI core services `DA473D7F` — required, or vendor handlers
  fail closed and hide defects), `tools/smm/amd/asus/`.

**Hunters** — input technique is *canary taint*: attacker-controlled pointer-sized fields
are filled with values in an unmapped canary window; a hit identifies the field, then a
second run aims it at an SMRAM guard page; a guard access is the lead, a rejection is
recorded as positive evidence.

| Priority | Hunter | Oracle |
|---|---|---|
| 1 | SMM callout | In SMM: fetch from non-SMRAM, or code pointer read from boot-services memory after ExitBootServices |
| 2 | Attacker-controlled pointer (CommBuffer, NVRAM variable, fixed mailbox) | Guard-page access via an input field; access beyond CommBufferSize |
| 3 | TOCTOU / double fetch | Same non-SMRAM offset read twice, confirmed by serving a different second value |
| 4 | Hardening state | Named predicates checked at the SMM_READY_TO_LOCK and EXIT_BOOT_SERVICES snapshots: SMM page tables read-only, CR0.WP set, nothing executable outside SMRAM, the memory-attribute protocol gone, SmmLock and the SMM_MASK valid bits set, CommBuffer copied before use, WSMT flags matched by behaviour. Each predicate names its measured counterpart (§4.7) |
| later | Dispatcher quirks, dead code | Duplicate SW values; static-vs-dynamic registration diff |

Embedded keys and known-CVE patterns stay static (§4.9).

Leads carry their **gating preconditions**: every variable, Setup byte and attribute check
on the path is recorded, the case is re-run with each gate flipped, and factory values
come from the image's defaults store through `tools/hii/common/setup_map.py`. The result
is a "holds by default / blocked by" table, which v1 built by hand for its one report.
The SMM layer's variable store is seeded from the same defaults as the DXE layer's.

**Honesty.** Each module is labelled `EMULATED_COMPLETE`, `EMULATED_PARTIAL(reasons)` or
`NOT_EMULATED(reason)`; a hunter below a coverage threshold reports "insufficient
coverage", never "clean". The static-vs-dynamic registration difference is itself a signal.
Entering at an internal function (an address named by a global annotation for that module
SHA) with declared modelled state is labelled `EMULATED_PARTIAL(harness-entry)`: such a
run may satisfy a regression or a known-answer test, but never counts as full coverage.

**Output policy.** Results are emulator evidence (source field, sink, rule site, trace).
v2 does not generate stand-alone exploit payloads for real hardware.

**Regressions** are specifications (`tests/regression/<id>.yaml`: image SHA, entry,
modelled state, positive cases, negative controls, expected outcome and site). They are
semantic: addresses appear only through global annotations. Module bodies come from
fixtures and are never committed.

1. BIOS 2503 — rediscover the `WpBufAddr` variable-supplied pointer write in `750890A6`
   via SW SMI 0xA0, together with the negative controls of v1's proof of concept (cases
   that must produce no write). A5557 has the same writer statically — v1 found it
   there and patched it out in rev82 — so emulating it confirms a known result.
2. BIOS 2503 — reproduce the NvramSmm store-rule refusal matrix (19 steps) in
   harness-entry mode, as v1 did; each step asserts the status **and** the refusing rule
   site, named by global annotations for that module SHA.
3. A5557 — the cached boot-services pointer path in `34DF9766`/`2EF2DEE2` under the Zen 5
   profile. Hardware measured only its precondition (the cache unset at hand-off);
   whether it is reachable after ExitBootServices is still open. Under a Phoenix profile
   the twin `EF9A853E` registers instead.
4. Dispatch what-ifs on A5557 (static; P4) — the rev86–90 DEPEX cycle and the lost
   MonotonicCounter architecture protocol of rev86/87, built as model overlays from v1's
   study records. The cycle must appear under the Zen 5 profile, and restoring
   MonotonicCounter alone (the rev88 shape) must still fail to dispatch the MM loader.
5. PEI dispatch (static in P4; repeated in P5e with installers proposed by emulation) —
   A5557's PEI with rev124's change set applied as a model overlay, compared with the
   one measured rev124 trail: 79 PEIM begins and ten non-APRIORI PEIMs that never began.
   Every disagreement is reported and attributed to a profile gate, a missing runtime
   installer or a model bug.

### 4.11 Capsule builds, gates and safety

**Recipes** (`recipes/<board>/<id>.yaml`, pydantic-validated): base pinned by SHA-256;
inputs pinned by SHA-256; **explicit volume selectors** that must resolve to exactly one
node (the "largest LZMA section" heuristic is banned) and address nested sections by
path; operations with guards; expected file-set delta; waivers; `exceptions` for
protected regions; variants.

**Lineage model.** A recipe declares its build `line` (v1 ran two, the old line and the C
line) and its `base`: an ingested image pinned by SHA-256, or `adopt: {sha, asserts:
<recipe>}` — an image built elsewhere whose ancestry is checked (the ancestor recipe's
assertions run on it) and never re-applied. `extends:` flattens the parent's operations
by stable operation `id`; each operation carries the lines it applies to and a `status`:
`active`, `withdrawn(reason)`, `superseded_by(id)` or `amends(id)`. Withdrawn and
superseded operations are still evaluated and reported as `WAIVED` / `SUPERSEDED`, never
dropped silently. `modules:` is a set of artefact SHAs (§4.12) that replaces the
parent's set wholesale. Each operation records `why` and `decided` (by, date, quoted
source); legacy recipes take both from v1's prune lists. Plain flattening would have
re-applied a change v1 withdrew after a boot loop, and old-line operations whose targets
the C line removed.

**Assertions** come from four sources: operation post-conditions (derived); feature
`markers` declared with each `uefi/` module (tag strings, variable names, GUIDs,
constants) and checked wherever its body appears; whole-image `absent` / `keep_once`
scans, from the recipe and from `data/boards/<board>/assertions.yaml`, over every decoded
section; and the structural gates below. Of v1's 91 hand-written invariants about half
are markers inside Sentinel-built bodies; others are whole-image absence scans,
structural guards or pins on vendor bytes, and only a minority are post-conditions of an
image operation. P6 ships a table mapping every v1 key to derived / marker / scan / gate
/ dropped (with the reason).

**Operations** (thirteen types cover everything v1 ever did):

| Home | Operations |
|---|---|
| `sentinel/build/ops` (generic PI) | `insert_ffs`, `replace_body`, `replace_file`, `remove_ffs` (→ same-size pad), `patch_pe`, `replace_raw`, `edit_apriori`, `grow_lzma_section` |
| `tools/build/ami/` | `ami.edit_nvar`, `ami.replace_default_key` |
| `tools/build/amd/` | `amd.psp_edit`, `amd.apcb_token`, `amd.pei_container` (0x62, both copies, PE→TE rebase) |

- FFS wrapping is **pure Python** (BaseTools absent; GenFfs output needed patching anyway);
  GenSec/GenFfs/GenFw are optional test oracles. State byte and pad-header checksum follow
  the volume's erase polarity (generic PI, not AMI-specific).
- Placement and guards: `insert_ffs` puts a file after the last file or in a named pad
  slot, which is split into the file and a residual pad (at least 0x18 bytes, 8-aligned);
  growth is absorbed only by the following pad and no surviving file moves (refused
  outright in an execute-in-place volume). `replace_body` reaches a file inside nested
  compression through its selector path. `patch_pe` covers PE32, PE32+ and TE, guards on
  the expected original bytes, refuses a span touched by a base relocation and recomputes
  the checksums. `ami.edit_nvar` has two modes: the defaults store, and the main NVRAM
  bank (which needs an exception).
- **Compressor rules.** LZMA: LZMA1 alone-format, lc3 lp0 pb2, dictionary from the
  original stream unless the recipe overrides with a reason, true size patched,
  decompressed length preserved, section never shrunk, overflow refused. zlib, only
  inside `amd.pei_container`: the vendor stream passes through untouched unless the
  inflated content changes or the recipe asks for a re-encode; then level 9, an exact
  round trip, the header's size field rewritten, and growth or shrinkage only into the
  container's trailing pad. The vendor stream cannot be regenerated (§3.4), so
  pass-through is what keeps a zero-operation rebuild byte-identical. The build manifest
  records the liblzma and zlib versions, because volume and container digests depend on
  them. Edits inside LZMA-F86, Tiano or Brotli are refused at first.
- `amd.pei_container` works on section `CE3233F5` in FFS `3E3B6DC0` of each outer FV,
  found through its `$BL2` entries: both copies are rewritten identically, inserted PEIMs
  are converted PE → TE and rebased to the load address, and the `$BL2` entries stay
  byte-identical. It carries a scoped exception covering only the container and its
  trailing pad, and needs the AMD PEI precondition below.
- Proven kernel to lift: `firmware/builds/ffs_replace.py:38`, `ffs_replace_batch.py:26,66`,
  `build_rev42.py:40`, `build_rev81.py:232,261`, `rev123_steps.py:128-389`,
  `rev124_steps.py:127-215` with its `check_geometry` and `check_xip`,
  `build_rev118_z0.py` (container rewrite), `rev111_steps.py:97-193`,
  `rev118_steps.py:66-225`, `build_sentinel_mm.py:341-411`, `lineage_invariants.py:59,1602`.

**Gates** return `pass | fail | not_run | inconclusive | waived | error`; any `not_run` or
`inconclusive` makes the verdict `INCOMPLETE`.

- *P6a, structural (generic PI)*: same size; container header passthrough; no byte
  outside declared spans; unchanged files byte-identical; file-set delta; FFS integrity;
  rewritten sections decode; execute-in-place identity for every image in an XIP volume
  (for a TE: ImageBase + StrippedSize − 0x28 = load base + offset); markers and `absent` /
  `keep_once` scans (board defaults include build paths, debug records and test-probe
  GUIDs); reproducible double build.
- *P6b, semantic*: architecture protocols still installed; dispatch model with cycle
  detection (for PEI: the dispatched set equals the parent's minus the removals); the
  removal gates; ACPI legality for changed or inserted AML (disassemble, recompile with
  zero errors, no Device without `_HID` / `_ADR` — the shape that bugchecked Windows on
  v1's rev103 while Linux booted); derived lineage; AMD PSP signatures, dual EFS, PEI
  copies identical, PEI container shape.
- *P7a, module-coupled*: artefact verdict, ABI pairs, bindings (§4.12).

**Removal gates** (mandatory on `remove_ffs`, `replace_file` and variable removal):
producer parity over protocols, PPIs, event groups, DEPEX and GUID HOBs; MM Communicate
consumers — a kept module that talks to a removed handler's GUID (v1's rev94 halted
entering Setup this way); FILE_GUID readers; NVRAM variable references (v1's rev46/47
boot-looped after deleting variables classed as safe); and liveness — refused when the
profile marks the target as the live member of a per-family pair or cannot tell, or when
a bundle bound to an image holding the same volume observed the module dispatch. Waivers
are `{guid, consumer FILE_GUID, reason}`; unused and orphaned waivers are reported.

**A gate must be able to fail.** Each gate declares a negative control — the parent
image, a mutated fixture, a fault switch or an older source ref — that must FAIL; a gate
whose control has not been seen to fail reports `inconclusive`. Each operation's verifier
first passes on a positive control (the XIP identity on the vendor's own images, for
example). Derived lineage checks that the parent image fails exactly the new assertions.
A pin that only restates the code's own constant is not evidence. v1's record shows
gates that could not fail: scripts that aborted yet reported success, a gate that exited
0 on FAIL, a negative control that ran positive.

Board-specific data (whole-image assertions, known-bad SHA list, PCR0 ranges, recovery
path) lives in `data/boards/<board>/`.

**Output** lands in `firmware/<base-ws>/build/<recipe>_<datetime>/`: the capsule; the
resolved recipe (flattened, with operation statuses) and a `build_id` derived from it and
stamped into any module that carries a build-id marker; input and artefact SHAs; tool and
encoder versions; gate results; a lineage report (PASS / FAIL / WAIVED / SUPERSEDED per
operation, marker and scan); intermediates and variants; the digests of every range the
board profile says is measured; and a `DRAFT — NOT FLASHED` banner. The image's identity
record (capsule SHA, measured-range and container digests, build id, recipe id) is
appended to the generated `data/boards/<board>/images.built.json`, which carries identity
only, never lifecycle state. The capsule is then **re-ingested automatically as a child
workspace** with a parent link, so the map shows parent-vs-child and hunters re-run only
on changed modules.

`run.py build --plan` resolves the recipe, selectors, placement, pad and compression
headroom and writes nothing; for a recipe whose base is absent it is the only check
available. Variants are edits of the resolved recipe by operation id (drop some
operations; re-encode only), which turns v1's hand-built bisect and probe images into one
line each. `run.py check <ws> --recipe R` runs R's gates and assertions on any ingested
image — v1's most-used command, `lineage_invariants.py <capsule>`.

**Safety (code-enforced).** No flash-write path exists (a lint test enforces it). Refused
without an explicit recipe exception: any write inside PSP/BIOS directories, EFS,
PSP-referenced blobs, microcode, NVRAM or the boot block; a size change; a base on the
known-bad list; an ambiguous selector; a removal without the removal gates; an artefact
whose verdict is not PASS or whose sources were dirty; a module set that mixes ABI header
digests; duplicate GUIDs; unpinned inputs. The known-bad list holds SHAs and reasons
only: it is a refusal list, not lifecycle state (§2.1).

**AMD PEI precondition.** `amd.pei_container`, and any edit of a PSP-referenced blob,
also requires that the base carries no BIOS-directory OEM-key or RTM-signature entry and
that the profile records Platform Secure Boot as not enforced, citing evidence. Unknown
state means *could not run*: on a part that enforces PSB the same edit can leave the
board unbootable. v1 edited PEI only because both held on its board and two
single-question flashes had shown the PSP accepting first a re-encoded, then a changed
stream.

Every build emits `FLASHABILITY.md` stating: which gates ran and what static checks can
never prove; that the vendor capsule header is passed through unchanged, so its signature
— which covers only the header — still verifies and **vouches for nothing in the modified
payload**, while no signature is known to cover the edited bytes (v1 left payload-signature
coverage unresolved); the board's declared write route; and the chip-recovery artefact
(none declared → `NOT FLASHABLE`). The **chip-recovery artefact** (`recovery.yaml`) is
chip-sized bytes for an external programmer, never a `.cap`: A5557.cap is 4 KiB larger
than the 32 MiB chip, an error v1's own restore manifest makes. The preferred artefact is
a full dump of this very board, identified by SHA and kept outside git; the vendor
`payload.bin` restores the code but neither NVRAM, enrolled keys nor the board's own
identity data.

v1's flash-preflight and refuter become generated, stateless documents for a human; v2
never reads them back. `CHECKLIST.md` lists the gate verdict and refusals; the known-bad
check by SHA; the recovery artefact and whether this image writes the NVRAM bank; what to
capture before and after the flash; the flash-medium file name and the SHA it must have;
the digests a first boot should measure, and which operations touch regions nothing
measures; and, from `uefi/abi.yaml`, the switch defaults, unsafe values and post codes of
every inserted Sentinel module. `REFUTER_BRIEF.md` states the claim, at most six evidence
paths and v1's nine failure classes, each marked with the gate that covers it or "no
mechanism" — never the supporting argument.

### 4.12 `uefi/` — Sentinel firmware module sources

**What exists in v1**: 155 INFs — 31 Sentinel board modules (17 DXE, 4 runtime, 8
Standalone MM, 1 MM core, 1 PEIM), 55 libraries + 3 mocks, 18 test probes, 37 host unit
tests, 10 old-line modules (7 DXE, 3 Traditional SMM) and one QEMU-only Standalone MM
variant; ~84k lines of C in `SentinelMmPkg`, of which about 30k (36k with headers) are
**generated from upstream by 12 `derive_*.py` scripts**. The board image ships 43
modules: those 31 plus 12 upstream edk2 modules built with Sentinel libraries and PCDs,
one of which keeps a vendor FILE_GUID. v1 cannot build today: upstream edk2 is
git-ignored and absent, every gate script defaults to a path that no longer exists, and
the Makefile build targets are stale.

**Layout** — filed by phase, then by what the code is compatible with:

```
uefi/
  Sentinel.dec                 one package root for all of uefi/** ([Includes] → include/*)
  abi.yaml  edk2.lock          ABI registry (below); pinned upstream and submodule commits
  include/{common,amd,ami,board}/{Guid,Protocol,Library}/
  pei/{common,amd}/            SentinelPeiTelemetryPei (generic core; AMD MSR/CMOS hooks)
  dxe/common/                  SmmBase2Shim, XhciHandoff, MemoryClear, ResetTrace, FormBrowserStub,
                               MmCensusDxe, PeiTelemetryDxe, derived/{SecurityStubDxe, VariableSmmRuntimeDxe}
  dxe/amd/                     AmdMarker, ApcbLocal, AcpiEnable (FCH part); amd/zen5/ for Zen 5-only code (SMU 0x72)
  dxe/ami/                     EarlyVariable, DevicePolicy, PostManager, UsbPrestartShadow, BootDump, bds/
  dxe/board/asus-x870e-e/      AcpiNeuter, XhcS0w
  smm/common/                  MmIplDxe, derived/{MmCore, PiSmmCpu}, PageAuditMm, MmCensusMm,
                               SwDispatchMm (core), VariableMm (core)
  smm/amd/                     SmmRelocationDxe, AmdFlashDoorMm, SmmLockMm, FCH SW-SMI backend, ROM-Armor NV backend
  smm/ami/                     TpmPpiMm, NVAR store
  lib/{common,amd,ami,board,qemu}/
  legacy/traditional-smm/      old-line modules, frozen
  platform/{asus-x870e-e, qemu-ovmf-ia32x64, qemu-bootdump, hosttest}/
                               DSC/FDF, assertions.yaml, derive/ (image-derived constants and
                               platform derive scripts), board PCD values, the manifest of every
                               shipped module (upstream ones included), DEPEX pins
  patches/edk2/                0001 PKCS#7 messageDigest fix (upstreamable), 0002 (legacy)
  tests/{host, mocks, probes/{common,amd,ami,qemu}, gates/<gate>/, guards}/
```

**Rules.**

- **Upstream EDK2 stays external.** `run.py edk2 bootstrap` fetches commit `52bbcffe54`
  — `edk2-stable202602` plus 178 commits, not the tag itself, which is a different tree —
  with submodules at the commits recorded in `uefi/edk2.lock`, into a git-ignored folder,
  and applies `uefi/patches/edk2/`. v1's tree was that commit but dirty (both brotli
  submodules modified, `Conf/` changed, five QEMU test packages fetched later); P7a
  reproduces or explains that state. Moving to a newer stable tag is a separate step,
  taken only after P7's reproduction passes. The build puts the repo and that checkout on
  `PACKAGES_PATH`, which is what lets directories be rearranged freely.
- **FILE_GUIDs and BASE_NAMEs never change** — recipes, guards and evidence decoders key on them.
- **Derived code keeps script + upstream pin + committed output together**, with a check
  that regenerates and diffs; scripts take source/destination as arguments instead of
  assuming the working directory.
- **ABI pairs move together.** Each firmware header that defines a published variable or
  record travels with its Python decoder (now `tools/evidence/sentinel/…`) and its guard
  test: census v10, MM audit, HOB snapshot, PEI telemetry, MM event log, crumbs, CF9 trap,
  Sx trail, hand-off records — and the many records v1 published without a guard (it
  guarded four layouts).
- **`uefi/abi.yaml` is the ABI source of truth**: one entry per published variable,
  record, HOB, post code and runtime switch — name, GUID, header with version and size,
  writer and reader FILE_GUIDs, decoder; for a switch also its values and meanings,
  default, unsafe values and fail-safe marker (v1's convention: absent = on, `00` = off,
  a marker byte = an earlier run never finished). Guards check every versioned layout
  against its decoder and every switch constant against the C; the ABI-pair list, the
  `abi_pairs` gate and the switch section of `CHECKLIST.md` are generated from this file.
  Post codes are allocated against the vendor's checkpoint table.
- **Image-derived constants and bindings.** Some facts are compiled into Sentinel code
  from a target image: vendor Setup offsets, PCD tokens, NVRAM bank addresses, an
  option-ROM Authenticode hash, PSP root-key pins, the addresses and code fingerprints an
  interposer relies on. `uefi/platform/<board>/derive/` generates them from a named
  workspace's plugin outputs and stamps them with that image's SHA; each module declares
  the facts it binds to; the `bindings` gate re-derives them from the recipe's base and
  fails on any difference, so a vendor update cannot silently invalidate them.
- **Two steps.** *P7a* copies each module to the folder of its dominant compatibility and
  proves it still builds. *P7b* splits the nine modules that mix generic and vendor code
  (SwDispatchMm, VariableMm, AcpiEnableDxe, BootDumpDxe, MmIplDxe, SecurityStubDxe, the PEI
  telemetry pair, PlatformBootManagerLib, PageAuditLib) into a generic core plus vendor
  backend libraries — this is what makes them reusable on another chipset.
- **Artefact records.** Built modules go to a content-addressed store
  (`firmware/_artefacts/`) and recipes consume them by SHA-256. A record carries the
  FILE_GUID and BASE_NAME; the body SHA, taken after debug records are scrubbed; the
  sources' content hash and dirty flag (not a git SHA: v1's history rewrite invalidated
  the SHAs stored in its records); upstream and submodule commits, toolchain versions,
  the stack-cookie value, build root and rev stamp; assertion and gate results, markers,
  ABI header digests and bindings; and a verdict. Bodies that v1 built (`c5_work/`) may
  enter with verdict `imported` and v1's provenance, for tests and plans only;
  `FLASHABILITY.md` reports any imported artefact.
- **Reproducible module builds.** Each variant builds under a fixed build root,
  independent of the checkout path; debug records are zeroed before hashing (v1's IA32
  PEIM embedded its build path and reproduced only through a same-length path alias); the
  stack-cookie value is pinned where the toolchain allows and always recorded (a fresh
  v1 build directory drew a new one, changing eight bytes in every module);
  `sentinel/diff.py` gains a module diff that masks cookie and stamp; a guard builds from
  two checkout paths and compares the artefact SHAs.
- **Board-build assertions are data.** `uefi/platform/<board>/assertions.yaml` holds
  typed checks evaluated over the build tree, with results stored in each artefact
  record: PCD values, DEPEX bytes, library allow / deny (no QEMU-only library in a board
  build), required and forbidden byte or instruction patterns (no MSR write in the
  PEIM), size budgets (MM images against TSEG), execute-in-place layout, test and fault
  switches off, stamp checks. They are the checks QEMU cannot make, and they replace v1's
  710-line per-rev shell script.
- Generic guards for any module inserted into foreign firmware: no dynamic PCD reads, no
  side effects inside `DEBUG()`, exact DEPEX pins, no test probes in output, derived
  files reproduce, no GUID-typed handler for MM lifecycle events, and an ACPI check for
  any module that embeds or patches AML (§4.11).
- **Module gates** are `gate` plugins in `uefi/tests/gates/<gate>/`. v1 runs thirteen —
  host tests, mmio-negative, c2, c4, mor, frames, crumbs, bootdump, setup, imgverify,
  keyprov, pei, ebs — in a fixed, hand-kept order, only because every gate rebuilds one
  shared build directory with different defines. In v2 each gate declares its platform
  variant, resources (root, swtpm, loop devices, TCG rather than KVM for the AMD MSRs),
  fixtures (missing → counted `not_run`) and negative control, and each variant builds
  into its own output directory keyed by variant hash, so the ordering rules disappear.
  Run records go to the artefact store with the §4.11 statuses. The host gate includes
  the crypto known-answer and forgery tests, which guard patch 0001 across EDK2 re-pins.
  The OVMF gates need a working QEMU, which this host lacks today: until it is repaired
  each reports `not_run`.
- The platform module manifest lists **every shipped module**, upstream ones included,
  with any FILE_GUID or file-type override. v1's CryptoDxe (`FEA01457`, carrying the
  PKCS#7 fix) was built from a configuration that was never tracked; P7a adds it, pins
  its crypto library and aims at v1's body SHA; until that succeeds the body is an
  `imported` artefact.

Unverified: whether BaseTools accepts INFs that sit outside the directory of the `.dec`
they use (the single `Sentinel.dec` at `uefi/` avoids the question); submodule versions
and what v1's dirty submodules changed; whether the toolchain lets the stack cookie be
pinned; whether today's compiler reproduces v1's module bodies beyond cookie and stamp
(v1 recorded no compiler version); whether some old-line DXE binaries are still carried
in the latest board image.

### 4.13 Naming

Pluggable resolver; all hits kept with provenance and shown in the map. Modules: UI
section → workspace overrides → global overrides → vendor GUID databases → edk2 database →
PDB name. Seeds: v1's edk2 databases (1,367 modules; 780 GUID names, of which 313 are
protocols, 89 PPIs and 378 other GUIDs), re-derived from the pinned checkout once
`edk2 bootstrap` exists; UEFITool `guids.csv` (12,182, BSD-2-Clause, attributed); v1's
hand-identified names. The v1 importer drops any name whose only source is the
third-party SBOM scan or a census built on it (§7), and a test checks the tracked name
files. `guids harvest` grows a reviewed `learned` database from every processed capsule.

v1's per-module judgement — the roles, risk notes, do-not-patch flags and dated
corrections in its SMM and PEI maps, with later study corrections overriding stale map
text — is imported into `data/annotations/global.yaml` keyed by module SHA, falling back
to the FFS GUID with a "body differs" caveat. Because v1's SMM corpus bodies equal
A5557's (§3.3), the SHA keys attach on the first ingest.

## 5. Lessons carried forward — each becomes code, not prose

| v1 lesson | Where v2 enforces it |
|---|---|
| File names mean nothing; identify by hash | SHA-256 identity, index |
| An unrun check must not read as clean | `Status`, `RAN_NO_COVERAGE`, gate `not_run` → `INCOMPLETE` |
| Automated findings are leads | Finding schema: tier, confidence, `status = lead` |
| Measured evidence overrides static, both ways | `observed` layer, evidence importers |
| GUID databases and SBOM names are unreliable | Name provenance; never used as evidence |
| Never assume both per-family twins are live | Platform profile; unknown → *could not run* |
| Identify gBS/gSmst by provenance, not slot offset | `sentinel/disasm/provenance.py` |
| Proximity scans confuse install and locate | Core emits DEPEX/reference edges only |
| Never hardcode RVAs or offsets | Addresses as (module SHA, RVA); selectors over the model |
| Check consumers before removing a module — callers by MM Communicate GUID and by FILE_GUID, and variable owners, included | Removal gates: parity, Communicate consumers, FILE_GUID readers, variable references, liveness (§4.11) |
| Carried-forward changes get lost, and withdrawn ones must not return | Operation ids with status; markers and image scans; WAIVED / SUPERSEDED always reported |
| Pad checksum / State byte bricks | `ffs_integrity` gate; one implementation |
| LZMA parameter mistakes corrupt images | Single compressor rule |
| PSP regions are untouchable | Refusal without recipe exception; signature gate |
| Corrections never edit generated output | Overlays |
| Generated code must not be hand-edited | Derive scripts kept with the modules (§4.12) |
| A gate that cannot fail proves nothing | A negative control per gate, else `inconclusive`; the parent image must fail exactly the new assertions |
| A reproduced hash proves fidelity, not intent (the live member of a per-family pair was removed, and the pinned hash locked the mistake in) | Liveness gate; twin families decoded from the family gates |
| QEMU pass ≠ board pass | Board-build assertions as data; dispatch model run on the real image |
| Only one body of an ABI pair was rebuilt | `abi_pairs` gate over the header digests of `uefi/abi.yaml` |
| Sentinel code silently depends on vendor bytes | Image-derived constants and the `bindings` gate |
| ACPI that one OS accepts can bugcheck another | `acpi_legality` gate |
| A service-table slot means different things by module type | `callscan` slot tables per file type, checked against the pinned headers |
| Absence of a signal is not evidence; an all-ones read is "no decode" | Ran marks, `read_status`, expected records derived from the bound image |
| A static "runs" is only a prediction | Assumed providers listed; a model-vs-observed disagreement is a finding |
| "Nothing found" has a horizon | Static horizon metrics in coverage |
| A scan hit or a CVE attribution can be inferred without being verified | Match level confirmed / inferred / class; overlay status with a reason |
| Module bytes varied with the build path and a random cookie; git SHAs did not survive a history rewrite | Scrubbed, path-independent builds; content hashes in every record |
| A checker rewrote the registries it read | Read commands write only inside the workspace |
| Built capsules could be deleted by a clean target | No command deletes build outputs; clean-up touches the cache only |
| Key bytes and undisclosed findings must not leak into shareable output | Sensitivity flag; fingerprint-only crypto output; a test |
| Firmware code was run natively for crypto tests | Known-answer tests re-hosted on the emulator |

`docs/LESSONS.md` carries the full narrative (rev history arc, each brick and its rule).

**Carried only as documents** (the operator loop, D1; `docs/operator-loop.md`): Flashback
is the only write route and wipes NVRAM and keys; capture the key store before and after
a flash; take a chip dump before flashing an image that writes the NVRAM bank; one
behaviour change per flash; keep a known-good image on the flash medium and treat that
medium as a pre-OS key; the rollback hazards of particular images; only the owner
declares an image stable; status text goes stale; correlation is not cause; an agent's
summary is never evidence.

## 6. Roadmap

Each phase ends with a demonstrable exit criterion.

| Phase | Scope | Exit criterion |
|---|---|---|
| **P0 Foundation** | `pyproject` + uv env, `run.py`, `doctor`, workspace, plugin API/runner/cache, findings + coverage schema, fixtures registry, `docs/LESSONS.md`, project `CLAUDE.md` with safety rules | `run.py doctor` reports accurately; empty-dropzone run; unicorn confirmed importable (or interpreter pinned) |
| **P1 Ingest + model** | Sniffers, unified extractor, decompressors, AMD PSP layout, AMI NVAR, identify, profiles, names | A5557.cap, 02102026.bin, 2503.CAP ingest with golden outputs; CarbonX1.bin ingests as Intel with missing plugins shown in coverage; a `.bios`-renamed copy and a junk file behave correctly; A5557 provenance question settled |
| **P2 Evidence** | Bundle format, attach-by-measurement, importers, `observed` layer, capture kit, v1 captures converted, known-but-absent image records | The rev84 capture binds to its flash-window image by measured digest; rev97–124 bundles import as known-but-absent; a measured fact visibly overrides a static one; no identifier reaches a tracked file |
| **P3 Map** | All views in §4.8 | Offline map for A5557 incl. dependency graph and diff |
| **P4 Static plugins** | §4.9 | ≥130 SMI registrations on A5557 plus the FchSmm gap closed; dispatch model flags the synthetic cycle |
| **P5 Emulation** | P5a engine + SMM layer + registration census → P5b hunters 1–2 → P5c TOCTOU + fuzzing → P5d DXE → P5e PEI | Regressions 1–4 in §4.10; every module carries an emulation label |
| **P6 Build** | P6a ops + structural gates → P6b AMD/AMI ops + semantic gates → P6c re-ingest + generated reports | Zero-operation rebuild of A5557 is byte-identical; rev123 and rev124-nopeim decompressed PEI hashes (`21340215…`, `98882582…`) reproduced; rev34's bad pad is rejected |
| **P7 `uefi/` modules** | P7a EDK2 bootstrap, copy and re-file Sentinel sources, artefact store → P7b split mixed modules into generic core + vendor backends | Board and PEIM builds succeed from the new layout with FILE_GUIDs unchanged; host unit tests pass; derived files reproduce |
| **P8 Intel** | IFD/ME/FIT layout, Intel SMM silicon plugin, Boot Guard ranges protected | CarbonX1.bin fully mapped; first Intel SMM census |
| **P9 Later** | Watch/daemon polish, PSP-stage emulation (fresh, Unicorn ARM), crypto bill of materials, sourced advisory matcher, faster fuzz backend | — |

**Parallelism.** P0 and P1 are sequential (interfaces freeze at the end of P1). After
that, up to five agents can work with disjoint file ownership: map (`sentinel/map/`),
static SMM (`tools/smm/**`, `sentinel/disasm/`), emulation (`sentinel/emu/`), build
(`sentinel/build/`, `recipes/`), evidence + `uefi/` (`sentinel/evidence/`,
`tools/evidence/`, `collect/`, `uefi/`). One integrator owns the model and plugin API.

## 7. Risks and unverified assumptions

- **Emulation fidelity.** A wrong model (AMI core services, an FCH context layout) causes
  silent false negatives; fidelity footnotes and the regressions are the guard. AGESA-heavy
  modules will often stop at silicon access and be reported as partial.
- **PEI emulation** (IA32, TE, PSP-loaded) is the highest-risk item and is last.
- **Byte-exact build validation is limited**: no v1 output capsule is on disk; LZMA-encoder
  dependent hashes may not reproduce. Decompressed-content hashes and differential tests
  against v1's own functions are the reliable checks.
- **Unverified**: that the Unicorn wheel imports on CPython 3.14; whether it can hook
  RDMSR/WRMSR directly (fallback: per-site hooks from a capstone pre-scan); what ASUS USB
  BIOS Flashback actually verifies; whether A5557.cap is the pristine vendor image; the AMD
  save-state offsets; the meaning of the extra `AA55AA55` at 0x1000 in `02102026.bin`.
- **Naming coverage** on a never-seen capsule will be thin until vendor GUID databases grow.
- **Licensing** of third-party name sources (Binarly-derived names) — keep out of git until
  settled; UEFITool's list is BSD-2-Clause and can be included with attribution.

## 8. Verification

Per phase (see the exit criteria in §6): `uv run pytest` with synthetic fixtures
(every section type, bad FFS size, TE, LZMA-F86, Tiano, rejected junk); golden outputs for
the real images located via `tests/fixtures.yaml` (missing fixture → explicit skip, counted);
end-to-end `python run.py` on a dropzone containing A5557 renamed with an unusual
extension; opening the generated `map/index.html` from disk; the emulation regressions;
zero-operation rebuild identity and negative replays for builds.

## 9. Appendix — v1 → v2 source map (beyond §4.9)

| v1 | v2 | Action |
|---|---|---|
| `interactive_map/tools/fv_inventory.py`, `encryption/tools/fwcorpus.py` | `sentinel/extract.py`, `sentinel/fmt/*` | merge, fix |
| `pei_research/tools/extract_pei.py` | `sentinel/fmt/pe.py`, module-dir layout; locator → `tools/layout/amd/` | lift |
| `smm_research/tools/pe_util.py` | `sentinel/fmt/pe.py` | extend to PE32/TE |
| `tools/depex/build_{guid,protocol}_db.py` + JSON | `sentinel/names/importers/edk2.py`, `data/guids/edk2/` | lift |
| `tools/depex/analyze.py` | `sentinel/fmt/depex.py` (`parse_depex` only) | lift, fix |
| `firmware/builds/dispatch_model.py` | `sentinel/model/dispatch.py` | rewrite (PEI/DXE/MM, SOR/BEFORE/AFTER, cycles) |
| `firmware/builds/producer_parity.py` | gate | lift |
| `firmware/builds/lineage_invariants.py`, `pcr0_registry.py`, `RETIRED_CAPSULES.md` | `data/boards/asus-x870e-e/*.yaml` + core mechanisms | parameterise |
| `firmware/builds/build_rev*.py`, `rev1xx_steps.py` | `recipes/asus-x870e-e/legacy/*.yaml` | distil into recipes; drop code |
| `firmware/builds/_rev*_work/`, `c5_work/` | `tests/fixtures/genffs/` | keep as golden vectors |
| `reports/…wpbufaddr…/evidence/emulation/*` | `tests/regression/` + `sentinel/emu/` ideas | re-host |
| `encryption/tools/harness/*` | `sentinel/emu/stubs.py` semantics | port semantics; retire native executor |
| `tools/secureboot/{dbx_tool,verify_auth2,…}.py` | recipe input validators | lift |
| `interactive_map/assets/*`, `build_map.py`, `names.py` | `sentinel/map/*`, `sentinel/names/*` | lift shell; rewrite |
| `.claude/commands/flash-preflight.md`, `.claude/agents/firmware-refuter.md`, `tests/boot_checklist.md` | generated per-build documents | parameterise |
| `CLAUDE.md`, `docs/*`, `*/progress.md` | `docs/LESSONS.md` | distil |
