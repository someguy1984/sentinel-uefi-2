# Sentinel UEFI v2 — project plan

> **Status:** plan. Written 2026-11-24, revised the same day after a coverage review
> against v1, and re-centred on 2026-11-25 by the project owner: v2 is a **UEFI
> engineering framework for any firmware image**, not a tool for one board. Nothing is
> implemented yet; work follows the roadmap in §6.
>
> Built from eleven research and design agent reports on `../sentinel-uefi` (v1),
> `../uefi` and `../uefi_framework`, plus decisions made by the project owner. The
> coverage review (ten area readers over v1, four adversarial verifiers, three
> independent planners) corrected about thirty statements about v1 and added build, gate
> and analysis mechanisms throughout §4. The 2026-11-25 revision moved everything that
> concerns one board into `boards/asus-x870e-e/` (§4.16), restated every exit criterion
> over a fixture matrix (§3.3), brought Intel into the first phases, and added the
> security baseline and upgrade library, the explain outputs and the durable-state rules
> (§4.14–§4.17).
>
> This file is `PLAN_v2.md`. `PLAN.md` is the 2026-11-24 plan, left unchanged for
> comparison.

## 1. Context

`../sentinel-uefi` (v1) is a 327-commit (Mar–Nov 2026) project that replaced the ASUS/AMI
DXE and SMM service layer of one board (ASUS ROG Strix X870E-E, Ryzen 9950X, AMI Aptio,
AMD AGESA) with edk2 code and custom "Sentinel" modules across ~125 firmware revisions.
It holds ~477 Python scripts and a large body of hard-won lessons, but it grew
investigation by investigation: tooling is scattered over a dozen trees, 61 of the 75
Python files in `firmware/builds` (57 of them builders) hardcode a path that no longer
exists, there is no dependency manifest, and almost every script assumes this one board
and a particular revision.

v2 turns that experience into a framework. A firmware file of any name — AMD or Intel,
any firmware vendor, any board — is dropped into `firmware_dropzone/`; `python run.py`
moves it into its own workspace `firmware/{capsulename_datetime}/` and does four things:

1. **Understand.** Takes the image apart, identifies it and explains how it works: an
   interactive module map, a dossier per module, the boot flow, and which modules are
   stock EDK2, vendor-modified or proprietary (§4.8, §4.15).
2. **Assess.** Scores the image against a security baseline whose controls each cite a
   published standard, and says what could not be assessed (§4.14).
3. **Find issues.** Static analysis and per-module emulation of SMM, DXE and PEI,
   reported as leads with evidence (§4.9, §4.10).
4. **Change.** Builds new capsules from declarative recipes — among them a library of
   reusable upgrades that bring a firmware up to the baseline — and firmware modules
   built from source (§4.11, §4.12, §4.14).

Tooling is organised as phase / silicon / vendor plugins. v1 is the source of lessons, of
lifted tools and of test oracles; its board is one board folder among those the framework
can hold (§4.16), and no core phase depends on it. v2 is self-contained: everything worth
keeping from v1 and its precursors is copied in, in a layout designed for reuse, and v2
never depends on the old repositories at run time. v2 ends at a gated capsule, its
generated documents and imported board evidence: flashing and a board's release
bookkeeping stay with the human (§2.1).

## 2. Requirements and decisions

Stated by the user:

- Entry point `python run.py`; input `firmware_dropzone/`; workspace
  `firmware/{capsulename_datetime}/` holding all workings.
- **Any file extension** is accepted (`.cap`, `.bin`, `.bios`, `.rom`, none). Type is
  decided by content, never by extension.
- Pipeline order: analyse → module-map page → emulate segments hunting defects → build.
- Plugins: `tools/smm/(amd|intel)/*.py`, `tools/pei/(amd|intel)/`, OEM extensions such as
  `tools/smm/amd/asus/`. (First stated as "AMD first"; superseded on 2026-11-25.)
- Preserve v1's history and teachings; streamline.
- (2026-11-25) The plan is not about the owner's own capsules. It must take any capsule,
  Intel or AMD alike, take it apart, show how it works and what it does, map it in
  detail and analyse its modules: a UEFI framework with the tooling to update, upgrade
  and harden firmware to current security standards and to identify issues.
- (2026-11-25) Board-specific workings live in a board folder — scaffolded EDK2 workings
  and the rest — with a map of what is inside, for context.

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
| D2, D3 | Concern the ASUS board only (its v1 capsules; its build line): now in `boards/asus-x870e-e/PLAN.md` |

Decided by the user on 2026-11-25:

| Decision | Choice |
|---|---|
| D4 Purpose | **A framework, not a board project**: take any capsule, understand it, assess it, find issues, change it (§1). No core phase is defined on the owner's own images |
| D5 Silicon | **Intel and AMD alike**, from the first phase (§6) |
| D6 Board workings | **One folder per board**, `boards/<board>/`, holding that board's data, recipes, scaffolded EDK2 workings, documents and oracles, with a map of its contents (§4.16) |

Proposed in the 2026-11-25 revision and open to change: exits stated over a fixture
matrix (§3.3); a security baseline as data, with an upgrade library tied to it (§4.14);
the explain outputs (§4.15); a vault for state that cannot be regenerated (§4.17).

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
| 12 | Track upstream: EDK2 pin, revocation lists, vendor releases, advisories | **Partly**: a new vendor release is dropped in and `diff`ed; pinned `edk2 bootstrap`; sourced advisory matching (§4.9, P4) |

**Not covered, by decision D1.** v2 keeps no board ledger or image states (live / stable /
escape), no rollback ladder, no capsule-retention list, no verdict of a captured bundle
against a build's expectations, no flash-medium check, no board status command and no
investigation or decision store; no code reads or writes such state. v1's knowledge of
these stages is preserved in `boards/asus-x870e-e/docs/operator-loop.md`, written by
hand in the board track from v1's `CLAUDE.md`, `/flash-preflight`, `firmware-refuter.md`
and `tests/boot_checklist.md`: the
Flashback procedure and its side effects, what to capture before and after a flash, the
first-boot readings, rollback hazards, retention discipline, the refuter's nine failure
classes and the boot checklist. It holds procedure and hazards only, never live board
state, which stays with the board's build line (the board folder's plan).

### 2.2 Open decisions (owner)

- **Baseline scope** (§4.14): which published standards the first control set covers.
- **Fixture matrix members** (§3.3): which public vendor images fill the classes that
  have nothing on disk (Intel + AMI, Insyde, Phoenix). Hashes and sources are tracked;
  binaries never are.
- **Vault location** (§4.17): where state that cannot be regenerated is copied — inputs,
  evidence, overlays, built capsules, module artefacts.
- **Vendor-line Sentinel SMM modules** (SMI shield, MmAudit W^X stages, vendor-line
  census): frozen in `uefi/legacy/` as planned, or kept alive as a reusable "harden the
  vendor's SMM from outside" upgrade for foreign AMI capsules (§4.14). Their record
  decoders are needed either way (§4.7).
- **Hand-written map commentary** (v1 `interactive_map/data/commentary.json`: phase
  prose, module notes, a glossary): import the module notes and glossary, or drop it.
- **Embargoed material.** Expected-lead entries and overlay text about undisclosed
  findings stay out of tracked files by default (§4.6). One of the ASUS folder's
  regressions names the subject of v1's embargoed report: whether that wording may stay
  in a pushed file is the owner's call (`boards/asus-x870e-e/PLAN.md`).
- **Emulator evidence in hand-written dossiers**: whether a dossier may cite SMRAM
  guard-page hits (v1's proofs of concept deliberately touched only benign pages).

Decisions about the ASUS board alone (its build line, its prebuilt old-line binaries, its
raw third-party scan files) are in `boards/asus-x870e-e/PLAN.md`.

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

### 3.3 Fixture matrix

Every exit criterion in §6 is stated over classes of image, never over one named file. A
class with no member on disk is reported `not_run`; it does not pass.

| Class | What it proves | On disk today |
|---|---|---|
| AMD, AMI Aptio, signed capsule | Capsule unwrap, PSP layout, NVAR, the AMI plugins | Vendor releases of one ASUS AM5 board (`boards/asus-x870e-e/PLAN.md`) |
| AMD, AMI, another OEM and socket, raw image | Nothing is keyed to one OEM, one socket or one AGESA line | One MSI AM4 release (`b4601a82…`, 32 MiB, no capsule header), zipped in `/home/j/Videos` |
| AMD machine dump with live NVRAM | `live_state` handling, privacy, partial images | Two dumps of the ASUS board |
| Intel, in-house firmware vendor, machine dump | Descriptor / ME / FIT layout; "unknown firmware vendor" | `../uefi/CarbonX1.bin` (`08852717…`, 32 MiB, Alder Lake-P, Lenovo; may hold device-unique data, §3.7) |
| Intel, AMI Aptio vendor image | The AMI plugins on Intel silicon; Boot Guard manifests | None |
| Insyde or Phoenix vendor image | A second firmware vendor's stores and conventions | None |
| Reference build (OVMF from the pinned EDK2) | **Ground truth**: names, DEPEX, dispatch order and handler registrations are known from source and from the build's own reports | Built by `edk2 bootstrap` (P1) |
| Successive releases of one board | `diff`, annotations that follow a module by SHA, the `bindings` gate | The ASUS releases above |
| Synthetic | Every section type, malformed structures, planted defects with known answers | Generated by `tests/synth/` |

Public vendor images are recorded in `tests/fixtures.yaml` by SHA-256 and source and are
fetched by the owner; binaries are never tracked. `run.py fixtures freeze` copies every
fixture by SHA into the vault (§4.17), so tests stop depending on where a file happens to
sit (v1's history has already been rewritten once).

### 3.4 Format facts checked against the bytes

Checked on one AMD / AMI image (`A5557.cap`, the ASUS board's baseline) unless stated.
Each is a design input, re-tested on every matrix class it applies to.

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
- `unicorn 2.1.4` installs and imports on CPython 3.14.4 (its `cp37-abi3` wheel; tested
  2026-11-25). Its x86 instruction hook accepts IN, OUT, SYSCALL and CPUID and **refuses
  RDMSR, WRMSR, RDTSC and RDTSCP**. Left untrapped, RDMSR and WRMSR execute silently and
  RDTSC returns host time (three fresh runs gave three values). Trapping at execution
  time works: a block hook that disassembles each block when it is first reached, plants
  per-address hooks, drops the translated block and resumes; or a per-instruction hook.
  Both caught every such instruction in the test.
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
sentinel/                    shared core library (generic PI/UEFI; no vendor or board knowledge)
  cli.py workspace.py doctor.py profile.py hashing.py guid.py diff.py log.py vault.py
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
  baseline/   control schema, evaluator, scorecard                       (§4.14)
  explain/    module dossier, boot flow                                  (§4.15)
  board/      board-folder schema, scaffold, contents map                (§4.16)
  map/        build.py, static/{index.html, assets/{core.js, style.css, views/*.js}}
tools/<phase>/{common,amd,intel,ami}/[<oem>/]     plugins                (§4.2)
uefi/<phase>/<compat>/…      firmware module library                     (§4.12)
boards/<board>/              everything that concerns one board          (§4.16)
collect/                     read-only capture kit run on a live board   (§4.7)
recipes/upgrades/*.yaml      reusable upgrades, not tied to one image    (§4.14)
data/        vendors.yaml, profiles/<id>.yaml, baseline/<set>.yaml,
             guids/{edk2,vendor,learned}/, names/overrides.yaml, annotations/global.yaml,
             catalogues/ (sourced lists, each with its source and date: advisories, Secure Boot
             objects and revocations, pattern rules, vendor releases)
firmware_dropzone/           input (contents git-ignored)
evidence_dropzone/           evidence bundles in (contents git-ignored)
evidence/<board>/<bundle>/   normalised bundles, each stored once (git-ignored, §4.7)
firmware/                    workspaces, _rejected/, _cache/ (the only clean-up target),
                             _artefacts/ (built modules by SHA, §4.12), _overlays/ (§4.17),
                             index.json (git-ignored)
ext/edk2/                    pinned upstream checkout made by `edk2 bootstrap` (git-ignored)
docs/        LESSONS.md, architecture.md, plugin-authoring.md, findings-doctrine.md, safety.md
tests/       unit/, synth/ (fixture generators, planted defects), golden/<fixture>/,
             regression/ (emulation specs), oracles/ (frozen outputs and check functions),
             fixtures.yaml, fixtures/bin/ (git-ignored)
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

A board folder adds `boards/<board>/tools/<phase>/` and `boards/<board>/contrib/`, loaded
only when the image's identity matches that board (§4.16).

Phases: `ingest layout identify nvram hii acpi oprom crypto image sec pei dxe smm psp
platform evidence baseline build gates`. `platform/` holds phase-neutral device models (MSR,
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
    kind: Literal["sniffer","decoder","identify","analyzer","hunter","importer","op","gate","emitter","control"]
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
  (version, date, source, capsule SHA), which says whether a file is an untouched vendor
  release. On Intel images the list also carries the Boot Guard indicators (FIT entries,
  the ACM, the key and boot-policy manifests and the ranges they cover), the ME version
  and the FSP version. v1 tracked exposure by microcode, AGESA and trustlet versions, by
  hand.
- **Platform profile** (`data/profiles/*.yaml`, or a board folder's `profile.yaml`)
  supplies what a capsule cannot: CPU family and SoC gates, default Setup/PCD values,
  boot mode, SMRAM assumptions. Most specific match wins; no match → `generic-<silicon>`
  with every field `unknown`, and plugins that need a gate report *could not run* (never
  assume both per-family twins are live). Each field records where it came from: a
  catalogue, an overlay, or a bound evidence bundle (the CPU a capture ran on, a measured
  `boot_integrity`). The resolved profile is frozen as `profile.lock.yaml`; its digest is
  in every cache key. One image often serves several CPU families, so a workspace may
  hold several profile variants: results are stored per profile digest, and evidence
  that changes a field creates a new variant instead of rewriting a locked one.

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
  modules/<GUID>_<sha8>/ image.efi|.te, manifest.json, strings.txt, dossier.json (§4.15)
  model/                 image regions volumes files sections modules census graph names identity observed
  analysis/<plugin-id>/<profile8>/  result.json + outputs, per profile variant (§4.3)
  emu/                   per-module run records, coverage, proposed plugin data
  evidence/<bundle-id>.json  binding to this image (match method, confidence); the bundle
                         itself is stored once in top-level evidence/     (§4.7)
  findings/              findings.jsonl, coverage.json, summary.md, boot_flow.md
  baseline/              scorecard.json, scorecard.md                     (§4.14)
  overlay/               → firmware/_overlays/<image sha>/: annotations, names, profile
                         (hand-edited; shared by every workspace of the same image, §4.17)
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
index → evidence bind → static analyzers → assess → findings → explain → **map** →
emulation → assess → findings → **map**. The map is rendered as soon as static results
exist and refreshed after emulation, evidence binding and `annotate`: that keeps the
order asked for in §2 (analyse → map → emulate → build) and puts a page in front of the
user before slow emulation finishes. Build runs on demand. The DAG comes from
`needs`/`produces`.

Cache key = plugin source hash + the hash of every `sentinel/` module the plugin imports,
transitively + the dependency lock + the model schema version + inputs + profile digest +
contribution digests. Without the core library in the key, a fix to the tracer would
leave every result built on it marked `CACHED`.

A plain `run.py` processes both dropzones. Evidence bundles are normalised into the
top-level store first; binding then runs across every workspace, and every later image
ingest re-runs it for bundles that are still unbound (§4.7).

```
run.py [process] [--file F] [--profile ID] [--only S…] [--until S] [--quick|--deep] [-j N]
run.py rerun <ws> [STAGE | --from STAGE] [--force]
run.py list | show <ws> [--gates] | plugins [--for <ws>] | map <ws> [--open]
run.py explain <ws> [--module G]       module dossier and boot flow (§4.15)
run.py assess <ws> [--set S]           baseline scorecard (§4.14)
run.py upgrades <ws>                   applicable / blocked / not assessable (§4.14)
run.py findings <ws> [--coverage | --across F]
run.py diff <wsA> <wsB>          run.py watch
run.py evidence import <bundle> | evidence bind <bundle> --claim <ws> | evidence list
run.py emulate <ws> [--module G] [--hunter H]
run.py build <recipe.yaml> [--variant V] [--plan]
run.py build --upgrade U --base <ws> [--plan]     an upgrade instantiated for one image (§4.14)
run.py check <ws> --recipe R     R's gates and assertions on any ingested image (§4.11)
run.py annotate <ws> <finding> --status S --reason … [--evidence P] | --note … | --promote
run.py board new <id> --from <ws> | board map <id> | board list           (§4.16)
run.py fixtures freeze | vault sync | vault verify                        (§4.17)
run.py guids import … | guids harvest
run.py edk2 bootstrap            fetch pinned EDK2 + apply uefi/patches   (§4.12)
run.py uefi build [--platform P] [--module M] | uefi gates [--only G]     (§4.12)
run.py doctor [--json]
```

`--quick` stops before emulation; `--deep` adds the fuzzing hunters. `doctor` reports
every capability and which stages it disables (today: unicorn not installed, BaseTools
missing, qemu broken, no vault). Exit codes: 0 all ran, 1 partial, 2 usage, 3 everything
rejected, 4 core environment broken. Findings never change the exit code unless asked.

Every read command takes `--json` (schema-versioned). Read commands write nothing outside
`firmware/<ws>/`; `data/` and `boards/` change only through explicit import, harvest or
scaffold commands, which print a diff. (Two of v1's "checkers" rewrote the registries
they were run to check.)

**Ground truth, one command each.** An agent runs the command and quotes its output; it
never infers the answer.

| Question | Command |
|---|---|
| What is this file, and which workspaces hold the same image? | `show <ws> --json` |
| What does this module do? | `explain <ws> --module G --json` |
| How does this image score, and what could not be assessed? | `assess <ws> --json` |
| Which upgrades apply to it, and what blocks the others? | `upgrades <ws> --json` |
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

Evidence is what a running board says about itself: event logs, PCRs, variables, tables,
registers. It is the only source for much of the baseline (§4.14: lock bits and
measurements are runtime state) and the only check on static and emulated conclusions.
v1 gathered it in more than thirty directories with differing layouts, no common manifest
and a rev number in a folder name as identity (`boards/asus-x870e-e/PLAN.md`). v2
normalises it.

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

**Known-but-absent images.** A bundle may describe an image that was never ingested.
Image records keyed by capsule SHA and measured digests live in a board folder's
`images.yaml` and may carry per-module digests, so that module-digest events bind too.
Such bundles import at once and bind when the image arrives.

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
15. Security baseline — scorecard by control family; per control its status, evidence
    tier and source standard, the modules involved and the upgrades that address it (§4.14)
16. Module dossier — one page per module, reached from every other view (§4.15)
17. How it boots — the boot-flow narrative, each statement linked to its fact (§4.15)

Each view declares the data it needs. Where the producing plugin has not run, the view
names it and says why, instead of showing an empty table.

### 4.9 Static analysis plugins

Lifted from v1 where v1 had the tool. v1 had no Intel code, so the Intel rows are new.

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
| `tools/layout/intel/{ifd,fit,bootguard}.py` | — | new: descriptor regions and access masks; FIT entries (microcode, ACM, key and boot-policy manifests); the ranges Boot Guard covers, which the build then protects (§4.11); ME region bounds and version only |
| `tools/pei/intel/fsp.py` | — | new: FSP-T / M / S headers, versions and configuration regions |
| `tools/smm/intel/` | — | new: the Intel save-state map, chipset SMI dispatch children as contributed data, SMRAM and flash lock predicates |
| `tools/platform/intel/` | — | new: device models for emulation — CPUID per profile, lock and feature-control MSRs, the chipset's flash-protection registers |
| `tools/nvram/common/vss.py` | — | new: the EDK2 variable store, plain and authenticated, and its fault-tolerant-write block, for images that do not use AMI NVAR |
| `tools/image/common/upstream_match.py` | the method of `code_similarity.py` | new: attributes each module to its upstream EDK2 origin (§4.15) |
| `tools/baseline/common/assess.py` | v1's hand-kept hardening and exposure ledgers | new: evaluates the control catalogue over model, emulation and evidence facts (§4.14) |

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
  lead that depended on it is footnoted. Port I/O and CPUID use Unicorn's instruction
  hooks. RDMSR, WRMSR, RDTSC and RDTSCP cannot be hooked that way (§3.4) and are trapped
  at execution time: each block is disassembled when first reached and its sites are
  hooked. A static pre-scan is not relied on, because a site it misses runs silently.
- Determinism — RDTSC = instruction count (through the trap above; untrapped it returns
  host time), seeded RDRAND, fixed time, bump allocator; identical inputs give
  byte-identical run summaries.
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
  state), `tools/platform/intel/` and `tools/smm/intel/` (lock and feature-control MSRs,
  flash-protection registers, Intel save state, chipset SMI dispatch — §4.9),
  `tools/smm/ami/` (AMI core services `DA473D7F` — required, or vendor handlers fail
  closed and hide defects), and OEM folders such as `tools/smm/amd/asus/`.

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

Three sources, in order of how much they prove about the framework:

1. **Planted defects** (`tests/synth/vuln/`): small modules written for the purpose and
   built against the pinned EDK2 — an SMM callout, a CommBuffer pointer write with and
   without a range check, a double fetch, a variable-supplied pointer — each with its
   negative control and for both save-state layouts. Every hunter must find every
   planted defect and stay silent on every control.
2. **The reference build** (§3.3): handlers and dispatch order are known from source, so
   the emulated registration census and the emulated dispatch are checked against
   ground truth.
3. **Board regressions**: results first obtained on real firmware, contributed by a
   board folder (`boards/<board>/tests/regression/`). The ASUS folder brings five from
   v1: a pointer write reached through an SW SMI, a store-rule refusal matrix, a cached
   boot-services pointer, two dispatch what-ifs, and PEI dispatch against a measured
   trail. They run when that folder's fixtures are present and gate no core phase.

### 4.11 Capsule builds, gates and safety

**Recipes** (`boards/<board>/recipes/<id>.yaml`, or an upgrade from `recipes/upgrades/`
instantiated for one image, §4.14; pydantic-validated): base pinned by SHA-256;
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
scans, from the recipe and from `boards/<board>/assertions.yaml`, over every decoded
section; and the structural gates below. (How v1's 91 hand-written invariants map onto
these four sources is board-track work: `boards/asus-x870e-e/PLAN.md`.)

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
path) lives in `boards/<board>/` (§4.16). An image that matches no board folder builds
with no board data: nothing is known-bad, no recovery artefact is declared, and
`FLASHABILITY.md` says `NOT FLASHABLE` until a folder exists (`board new`).

**Output** lands in `firmware/<base-ws>/build/<recipe>_<datetime>/`: the capsule; the
resolved recipe (flattened, with operation statuses) and a `build_id` derived from it and
stamped into any module that carries a build-id marker; input and artefact SHAs; tool and
encoder versions; gate results; a lineage report (PASS / FAIL / WAIVED / SUPERSEDED per
operation, marker and scan); intermediates and variants; the digests of every range the
board profile says is measured; and a `DRAFT — NOT FLASHED` banner. The image's identity
record (capsule SHA, measured-range and container digests, build id, recipe id) is
appended to the generated `boards/<board>/images.built.json`, which carries identity
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

**Intel Boot Guard precondition.** Any write inside a range covered by the boot-policy
manifest, or inside the FIT, the ACM, microcode, the descriptor or the ME region, is
refused without a recipe exception, and an exception also requires evidence that Boot
Guard is not enforced on the target. Unknown state means *could not run*.

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

### 4.12 `uefi/` — firmware module library

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
  include/{common,amd,intel,ami}/{Guid,Protocol,Library}/
  pei/{common,amd}/            SentinelPeiTelemetryPei (generic core; AMD MSR/CMOS hooks)
  dxe/common/                  SmmBase2Shim, XhciHandoff, MemoryClear, ResetTrace, FormBrowserStub,
                               MmCensusDxe, PeiTelemetryDxe, derived/{SecurityStubDxe, VariableSmmRuntimeDxe}
  dxe/amd/                     AmdMarker, ApcbLocal, AcpiEnable (FCH part); amd/zen5/ for Zen 5-only code (SMU 0x72)
  dxe/ami/                     EarlyVariable, DevicePolicy, PostManager, UsbPrestartShadow, BootDump, bds/
  smm/common/                  MmIplDxe, derived/{MmCore, PiSmmCpu}, PageAuditMm, MmCensusMm,
                               SwDispatchMm (core), VariableMm (core)
  smm/amd/                     SmmRelocationDxe, AmdFlashDoorMm, SmmLockMm, FCH SW-SMI backend, ROM-Armor NV backend
  smm/ami/                     TpmPpiMm, NVAR store
  lib/{common,amd,intel,ami,qemu}/
  legacy/traditional-smm/      old-line modules, frozen
  platform/{qemu-ovmf-ia32x64, qemu-bootdump, hosttest}/
                               reference platforms: DSC/FDF, assertions.yaml, the manifest of
                               every shipped module (upstream ones included), DEPEX pins
  patches/edk2/                0001 PKCS#7 messageDigest fix (upstreamable), 0002 (legacy)
  tests/{host, mocks, probes/{common,amd,ami,qemu}, gates/<gate>/, guards}/
```

Board modules and a board's platform (DSC/FDF, image-derived constants, board PCD values,
board assertions) are not here: they live in `boards/<board>/edk2/` and build against
this library through `PACKAGES_PATH` (§4.16). The `intel/` folders start empty — v1 wrote
nothing for Intel silicon — and P7b's split into generic core plus vendor backend is
what makes room for them.

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
  interposer relies on. `boards/<board>/edk2/derive/` generates them from a named
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
- **Board-build assertions are data.** `boards/<board>/edk2/assertions.yaml` holds
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
  with any FILE_GUID or file-type override. (v1's untracked CryptoDxe configuration is a
  board-track item.)

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
the vendor image's, the SHA keys attach the first time that image is ingested.

### 4.14 Security baseline and upgrade library

"Up to current security standards" needs a definition that is data, not opinion.

**Controls.** `data/baseline/<set>.yaml` holds controls. Each has an `id`; a `source`
(the published document, its section and date — a control without a source is not
admitted); `applies` (a `Match`, plus the facts it needs); one or more assessments, each
naming the fact it reads and that fact's layer — static (in the image), emulated (a §4.10
hardening predicate) or measured (an evidence item, §4.7); a verdict rule; and
`remediation`: upgrade ids, or the statement that only the vendor or the silicon can
change it.

**Status** is `pass`, `fail`, `unknown` or `not_applicable`, with the evidence tier that
produced it. Much of any baseline is runtime state — lock bits, measurements — that an
image alone cannot show: such a control is `unknown` until emulation or evidence supplies
the fact, and "the code that sets the lock is present" is a static pass, shown as weaker
than a measured one. `unknown` is never rounded to `pass`, and the scorecard always
states how many controls could not be assessed and why.

First control families. Each entry cites its exact source when the catalogue is written
(P4); the baseline adds no analysis of its own — the fact producers are the plugins of
§4.9 and the hardening predicates of §4.10.

| Family | Examples | Sources |
|---|---|---|
| Update and recovery | Signature coverage of the update, rollback protection, a recovery path | NIST SP 800-147, SP 800-193 |
| Root of trust | Boot Guard profile and covered ranges; Platform Secure Boot; which ranges are measured | Intel and AMD documentation; TCG PC Client specifications |
| Secure Boot | The platform key is not a test key; the revocation list against the current published one; certificate validity; default mode | UEFI specification; the UEFI revocation list |
| Flash protection | Write-protect and lock registers, protected ranges, descriptor access | Silicon documentation; Microsoft HSTI |
| SMM | SMRAM locked, nothing executed outside it, page-table protections, CommBuffer validation, WSMT flags matched by behaviour | Microsoft WSMT and Secured-core requirements; EDK2 guidance |
| DMA and memory | IOMMU tables and pre-boot DMA protection; the memory-overwrite request and its lock | Microsoft and TCG specifications |
| Image hardening | NX compatibility, no writable-and-executable sections, stack protection, the Memory Attributes Table | UEFI specification; Microsoft requirements |
| Variables | Lock or policy on boot-critical variables; authenticated writes | UEFI specification; EDK2 variable policy |
| Component currency | Microcode, AGESA or FSP, PSP or ME, embedded crypto libraries, known advisories | `data/catalogues/advisories.yaml` (§4.9) |
| Attack surface | Network stack, CSM, legacy option ROMs, shells, debug modules, persistence agents | Reported as an inventory with consumers, not as pass / fail |

**Upgrade library.** `recipes/upgrades/<id>.yaml` is a recipe template that is not tied
to one image. It declares `applies` and `preconditions` (facts and controls that must
hold: Boot Guard or Platform Secure Boot not covering the target, a consumer scan that
comes back empty); selectors written by role — the module that installs a given
protocol, the module attributed to a given EDK2 origin, a key store found by content —
never by one image's GUIDs or offsets; operations; `fixes`, the controls it is expected
to move; and `proven`, how far it has been tested: gates only, emulated, or booted on
named boards.

`run.py upgrades <ws>` lists each upgrade as applicable, blocked (naming the
precondition) or not assessable. `run.py build --upgrade U --base <ws>` instantiates it
as an ordinary recipe pinned to that image's SHA and builds it through §4.11: every gate
and refusal applies. The child is re-ingested and re-assessed, and a `baseline_delta`
gate requires that the `fixes` controls moved and that no other control got worse.

The first upgrades are what v1 proved on hardware, restated without its board: refresh
the revocation list and replace default key stores; replace a vendor module with its
EDK2 equivalent from `uefi/`; remove network, option-ROM and other unused surface behind
the removal gates; harden default settings in the defaults store. An upgrade booted on
one board says exactly that: `proven` is shown wherever the upgrade is offered.

### 4.15 Explain: dossiers, upstream attribution, boot flow

The map shows structure. These three outputs say what the firmware does. They are
emitters over facts that other plugins produced; every statement links to its fact and
tier, and each output lists what could not be determined.

- **Module dossier** (`modules/<GUID>_<sha8>/dossier.json`, view 16): names with their
  sources; upstream origin; phase, type and DEPEX; what it installs and what it consumes
  (protocols, PPIs, events, HOBs, configuration tables); its service-call histogram; the
  hardware it touches (ports, MSRs, MMIO, PCI), resolved against catalogues; variables
  read and written; SMI handlers and Communicate GUIDs; the Setup questions it owns;
  crypto use; hardening flags; clone family; the findings and baseline controls that
  involve it; its emulation label.
- **Upstream attribution** (`tools/image/common/upstream_match.py`): each module is
  compared, function by function, with a reference corpus built from the pinned EDK2 and
  from earlier stable tags, using the normalised function hashes of `clones.py`. The
  result — origin path, closest tag, fraction matched, the functions that differ — sorts
  every module into stock EDK2, vendor-modified EDK2 or proprietary. That tells an
  engineer where to read first, lets advisories be matched by inferred version (match
  level `inferred`, §4.9) and gives upgrades their "replace with the EDK2 equivalent"
  selectors. Vendors build with other compilers than the corpus, so function hashes may
  match poorly (§7); strings, GUID sets and constants are the fallback signals, reported
  at lower confidence.
- **Boot flow** (`findings/boot_flow.md`, view 17): from reset to runtime in the image's
  own terms — which stage verifies which, who initialises memory, the dispatch order the
  model predicts for PEI, DXE and MM, where the locks are set, what stays callable at
  runtime. Predictions are marked as such; where a bound bundle measured the boot, the
  measured sequence is shown beside the predicted one.

### 4.16 Board folders

Everything that is true of one board, and only of that board, lives in `boards/<board>/`
(`<oem>-<model>`, lower case: `asus-x870e-e`).

```
boards/<board>/
  README.md        what the board is, what the folder is for, how it is used (hand-written)
  MAP.md           generated map of the folder: every entry, what it is, its state, and counts
                   of images, recipes, modules, bundles and regressions
  board.yaml       identity and matching rule ($FID tag, SMBIOS), OEM, silicon, firmware vendor,
                   write route, which event-log entries identify the image
  profile.yaml     platform profile (§4.3)
  images.yaml  images.built.json  known_bad.yaml  pcr0.yaml  recovery.yaml  assertions.yaml
  contrib/         *.contrib.yaml, merged after OEM data (§4.2)
  tools/<phase>/   plugins that make sense only on this board
  recipes/         this board's recipes; legacy/ for history replayed as data
  edk2/            scaffolded EDK2 workings: platform DSC/FDF, board modules
                   ({pei,dxe,smm,lib}/), derive/ (image-derived constants), board PCD values,
                   assertions.yaml, the manifest of every shipped module, DEPEX pins
  evidence.yaml    index of this board's bundles (the bundles stay git-ignored, §4.7)
  docs/            operator loop, history, decisions and lessons that concern only this board
  tests/           regressions and oracles the board contributes; fixtures.yaml (hashes and
                   locations, never binaries)
```

- **Scaffold.** `run.py board new <id> --from <ws>` creates the folder from an ingested
  image: identity facts fill `board.yaml`, profile fields start `unknown`, and `edk2/`
  gets a platform skeleton that builds against `uefi/`. Nothing is invented. `board map`
  regenerates `MAP.md`; a test fails when `MAP.md` is stale or a tracked file is missing
  from it.
- **One-way dependency.** `sentinel/`, `tools/` and `uefi/` never import from a board
  folder or name one; a lint test enforces it. A board folder is loaded only when the
  image's identity matches its `board.yaml`. No exit criterion in §6 depends on one.
- **Promotion.** Knowledge starts where it was learned and moves up when a second case
  needs it, instead of being copied: board → OEM (`tools/<phase>/<silicon>/<oem>/`) →
  silicon or firmware vendor → common.
- The folder is tracked, so the privacy rules of §4.7 apply to everything in it.

The first folder is `boards/asus-x870e-e/`, v1's board. Its own `PLAN.md` holds what was
moved out of this plan: the images on disk, v1's evidence, the five regressions, the rev
lineage and the board's EDK2 platform, as a board track that runs beside §6.

### 4.17 Durable state

`firmware/`, `evidence/` and the dropzones are git-ignored, rightly: they hold firmware
and machine identifiers. But part of what they hold cannot be regenerated, and v1 shows
the cost: none of its built capsules survives, although it kept a never-delete list.

| Store | Regenerable? | Kept by |
|---|---|---|
| Workspace derivatives (extract, model, analysis, map) | Yes, from the input | Nothing: `rerun` |
| Input images | Vendor releases yes, from their source; dumps no | Vault |
| Evidence bundles, `raw/` included | No | Vault |
| Overlays (hand-written judgement) | No | Vault; public entries promoted to the tracked `data/annotations/global.yaml` (`annotate --promote`) |
| Built capsules and their build records | Only while base, artefacts, recipe and encoder versions all survive | Vault |
| Module artefacts (`firmware/_artefacts/`) | Only by a reproducible rebuild | Vault |
| Fixtures | No | Vault (`fixtures freeze`) |

- Overlays are stored by image SHA (`firmware/_overlays/<sha>/`), so every workspace of
  the same image shares them and a re-drop loses nothing.
- The **vault** is a directory outside the repository, named in an untracked local
  config. `run.py vault sync` copies new items into it by SHA: it only adds, and never
  deletes or overwrites. `vault verify` re-hashes it. `doctor` reports unsynced items
  that cannot be regenerated, and a missing vault as a disabled capability.
- **Schema changes.** A workspace whose `schema_version` is older than the code is
  rebuilt from its input by `rerun`. Overlays, evidence bindings and build records sit
  outside the rebuilt part and are migrated, never dropped.

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
| A tool grown on one board assumes that board everywhere | Exits stated over a fixture matrix; board knowledge confined to `boards/<board>/`; the core never imports from it |
| Hardening was tracked in prose ledgers that went stale | Baseline controls as data, each with a source, a status and an evidence tier (§4.14) |
| None of v1's built capsules survives, despite a never-delete list | A vault for state that cannot be regenerated; `doctor` reports what is unsynced (§4.17) |

`docs/LESSONS.md` carries the full narrative (rev history arc, each brick and its rule).

**Carried only as documents** (the operator loop, D1;
`boards/asus-x870e-e/docs/operator-loop.md`): Flashback is the only write route and wipes NVRAM and keys; capture the key store before and after
a flash; take a chip dump before flashing an image that writes the NVRAM bank; one
behaviour change per flash; keep a known-good image on the flash medium and treat that
medium as a pre-OS key; the rollback hazards of particular images; only the owner
declares an image stable; status text goes stale; correlation is not cause; an agent's
summary is never evidence.

## 6. Roadmap

Each phase states what it needs, what it delivers and an exit criterion over the fixture
matrix (§3.3). A matrix class with no member is reported `not_run`; it does not pass.

| Phase | Needs | Delivers | Exit criterion |
|---|---|---|---|
| **P0 Foundation** | — | `pyproject` + uv env, `run.py`, `doctor`, workspace, plugin API / runner / cache, findings + coverage schema, fixtures registry and `fixtures freeze`, vault, `.gitignore`, hygiene tests (§8), `docs/LESSONS.md`, project `CLAUDE.md` with safety rules; the host repairs `doctor` lists (QEMU's missing library, swtpm) | `doctor` reports accurately; empty-dropzone run; the Unicorn trap test of §3.4 passes; the hygiene tests fail on a planted identifier and a planted private key |
| **P1 Dissect** | P0 | Sniffers, extractor, decompressors; layout for **AMD (EFS, PSP) and Intel (descriptor, FIT, Boot Guard manifests, ME bounds, FSP)**; NVRAM stores (AMI NVAR, the EDK2 variable store); identify, profiles, names; DEPEX and the dispatch model; `edk2 bootstrap` and the reference build; the map shell (views 1–3, 13); one thin plugin of every kind | Every matrix class ingests, and claimed plus unclaimed bytes equal the image size; silicon, firmware vendor and OEM are identified with evidence or reported `unknown`, never guessed; an image from a vendor with no plugins yields a map and a coverage list, not an error; on the reference build, names and DEPEX equal the build's own reports; a renamed-extension copy and a junk file behave correctly. **Interfaces freeze** only after a thin plugin of every kind has run |
| **P2 Evidence** | P1 | Bundle format, bind-by-measurement, importers, the `observed` layer, the capture kit | A kit capture from the development host imports with a `read_status` on every item; a synthetic bundle binds to its image by measured digest and to no other; an unbound bundle binds when its image is ingested later; a virtualised capture never binds; no identifier reaches a tracked file |
| **P3 Map + explain** | P1 | The remaining views, each declaring its data; `diff`; module dossier and boot flow | The offline map opens from disk for every matrix class; every view shows data or names the plugin that has not run; `diff` of two releases of one board lists changed, added and removed modules |
| **P4 Analyse + assess** | P1; P2 for measured facts | The §4.9 plugins on both silicon lines; upstream attribution; exposure matching; the baseline catalogue and scorecard | On the reference build the SMI / MMI registry equals the registrations in source and attribution names every module's origin; on AMD and Intel vendor images the registry runs and counts its unresolved sites; the dispatch model flags the synthetic cycle; every class gets a scorecard in which no control passes without evidence; a measured fact overrides a static one in view 11 |
| **P5 Hunt** | P1, P4 (registry) | P5a engine + SMM layer for both save-state layouts + registration census → P5b hunters 1–2 → P5c TOCTOU + fuzzing → P5d DXE, with the known-answer tests re-hosted → P5e PEI | Every planted defect is found and every control case stays silent (§4.10); the emulated census equals source on the reference build; every module of every matrix image carries an emulation label; two runs give identical summaries |
| **P6 Change** | P1, P4 (consumers, variable references, dispatch); P2 for the liveness gate | P6a operations + structural gates + artefact records → P6b vendor operations (AMI, AMD), Intel refusals, semantic gates → P6c re-ingest, generated documents, the upgrade library | A zero-operation rebuild is byte-identical for every matrix image; every gate has been seen to fail on its negative control; one upgrade is applied to an AMD image and offered to an Intel image, each ending in a gated capsule or a refusal that names its precondition; the re-ingested child shows the targeted control moved and none worse |
| **P7 Module library** | P1 (`edk2 bootstrap`), P6a (artefact records) | P7a re-file the generic Sentinel sources, per-variant builds, gates as plugins → P7b split mixed modules into generic core + vendor backends | The reference platforms build from the new layout with FILE_GUIDs unchanged; host unit tests pass; derived files reproduce; two checkouts give the same artefact SHAs; every OVMF gate runs, or is counted `not_run` with the missing host capability named |
| **P8 Breadth** | P4 | Further firmware vendors (Insyde, Phoenix), more Intel and AMD generations, further OEM data | Each new class joins the matrix and meets the P1–P4 exits |
| **P9 Later** | — | Watch / daemon polish, PSP-stage emulation (fresh, Unicorn ARM), ME internals, a faster fuzz backend | — |

**Milestones** a user can see: *Dissect* (P0–P1: any image → workspace and map), *Explain
and assess* (P3–P4: dossiers, boot flow, scorecard), *Hunt* (P5), *Upgrade* (P6–P7).

**Board track.** Work on a board folder runs beside these phases and never gates them.
`boards/asus-x870e-e/PLAN.md` holds the ASUS track: v1's captures converted, its results
replayed as regressions, its revs distilled into recipes, and its board modules and
platform built from the new layout.

**Parallelism.** P0 and P1 are sequential. After the interface freeze up to six agents
can work with disjoint file ownership: map and explain (`sentinel/map/`,
`sentinel/explain/`), static analysis and baseline (the analyzers under `tools/`,
`sentinel/disasm/`, `sentinel/baseline/`), emulation (`sentinel/emu/`), build
(`sentinel/build/`, `recipes/`), evidence (`sentinel/evidence/`, `tools/evidence/`,
`collect/`) and the module library (`uefi/`). One integrator owns the model and the
plugin API.

## 7. Risks and unverified assumptions

- **Breadth against depth.** v1's depth came from one board over eight months. A framework
  spread over many images can end shallow on all of them. The guards are the reference
  build, which gives ground truth in every phase, and honest coverage: an image the tools
  understand poorly must say so.
- **Fixture supply.** Three matrix classes have no member on disk (§3.3). Until they do,
  what the plan says about Intel + AMI and about a second firmware vendor is untested.
- **Intel knowledge starts at zero.** v1 has no Intel code and no evidence from an Intel
  board. The Intel plugins are new work, checked statically and on the reference build
  until someone captures evidence from an Intel machine.
- **Emulation fidelity.** A wrong model (AMI core services, an FCH context layout, a
  save-state map) causes silent false negatives; fidelity footnotes, planted defects and
  the reference build are the guard. Silicon-heavy modules will often stop at hardware
  access and be reported as partial.
- **PEI emulation** (IA32, TE, loaded by the PSP or under Boot Guard) is the highest-risk
  item and is last.
- **Upstream attribution across compilers.** Unverified that normalised function hashes
  survive the difference between a vendor's toolchain and the reference corpus's (§4.15).
- **Baseline assessability.** Many controls are runtime state. On an image with no
  evidence the scorecard may be mostly `unknown`: the honest answer, but it has to stay
  useful, so static indications are reported separately from measured results.
- **Upgrades are proven on one board.** Everything distilled from v1 booted on one board
  only, and Boot Guard or Platform Secure Boot enforcement will block most payload
  changes on many retail machines; `proven` and the preconditions say so per image.
- **Byte-exact build validation is limited**: LZMA-encoder dependent hashes may not
  reproduce. Zero-operation identity, decompressed-content hashes and differential tests
  against v1's own functions are the reliable checks.
- **Unverified**: what a vendor's update path actually verifies, per board; the AMD
  save-state offsets and the Intel save-state map against real firmware; whether
  BaseTools accepts the single-`.dec` layout (§4.12); QEMU on this host, broken today.
  Settled on 2026-11-25: Unicorn imports on CPython 3.14, and RDMSR / WRMSR / RDTSC need
  execution-time traps (§3.4).
- **Naming coverage** on a never-seen capsule will be thin until vendor GUID databases grow.
- **Licensing** of third-party name sources (Binarly-derived names) — keep out of git until
  settled; UEFITool's list is BSD-2-Clause and can be included with attribution.

## 8. Verification

Per phase (the exit criteria of §6), over the fixture matrix:

- `uv run pytest` with synthetic fixtures: every section type, a bad FFS size, TE,
  LZMA-F86, Tiano, rejected junk, planted defects and their controls.
- The reference build as ground truth for names, DEPEX, dispatch, handler registration
  and attribution.
- Golden outputs for real images located through `tests/fixtures.yaml`. A missing fixture
  is an explicit skip, counted, and its matrix class is `not_run`.
- End to end: `python run.py` on a dropzone holding one image of each class, one of them
  renamed with an unusual extension; the generated `map/index.html` opened from disk.
- **Hygiene tests, from P0**: no identifier pattern, private key or firmware binary in a
  tracked file; no flash-write code path; nothing outside `boards/` imports from or names
  a board folder; the `MAP.md` of every board folder is current.
- Builds: zero-operation rebuild identity, each gate's negative control, negative replays.
- Board folders bring their own regressions (§4.10). They run when the folder's fixtures
  are present and never gate a core phase.

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
| `firmware/builds/lineage_invariants.py`, `pcr0_registry.py`, `RETIRED_CAPSULES.md` | `boards/asus-x870e-e/*.yaml` + core mechanisms | parameterise |
| `firmware/builds/build_rev*.py`, `rev1xx_steps.py` | `boards/asus-x870e-e/recipes/legacy/*.yaml` | distil into recipes; drop code |
| `firmware/builds/_rev*_work/`, `c5_work/` | `tests/fixtures/genffs/` | keep as golden vectors |
| `reports/…wpbufaddr…/evidence/emulation/*` | `tests/regression/` + `sentinel/emu/` ideas | re-host |
| `encryption/tools/harness/*` | `sentinel/emu/stubs.py` semantics | port semantics; retire native executor |
| `tools/secureboot/{dbx_tool,verify_auth2,…}.py` | recipe input validators | lift |
| `interactive_map/assets/*`, `build_map.py`, `names.py` | `sentinel/map/*`, `sentinel/names/*` | lift shell; rewrite |
| `.claude/commands/flash-preflight.md`, `.claude/agents/firmware-refuter.md`, `tests/boot_checklist.md` | generated per-build documents | parameterise |
| `CLAUDE.md`, `docs/*`, `*/progress.md` | `docs/LESSONS.md` | distil |
