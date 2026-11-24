# Sentinel UEFI v2 — project plan

> **Status:** initial plan, 2026-11-24. Nothing is implemented yet; work follows the
> phased roadmap in §6.
>
> Built from eleven research and design agent reports on `../sentinel-uefi` (v1),
> `../uefi` and `../uefi_framework`, plus decisions made by the project owner during planning.

## 1. Context

`../sentinel-uefi` (v1) is a 327-commit (Mar–Nov 2026) project that replaced the ASUS/AMI
DXE and SMM service layer of one board (ASUS ROG Strix X870E-E, Ryzen 9950X, AMI Aptio,
AMD AGESA) with edk2 code and custom "Sentinel" modules across ~125 firmware revisions.
It holds ~477 Python scripts and a large body of hard-won lessons, but it grew
investigation by investigation: tooling is scattered over a dozen trees, 61 of the 75
build scripts hardcode a path that no longer exists, there is no dependency manifest,
and almost every script assumes this one board and a particular revision.

v2 turns that into an automated, capsule-agnostic service. A firmware file of any name
is dropped into `firmware_dropzone/`; `python run.py` moves it into its own workspace
`firmware/{capsulename_datetime}/`, identifies and extracts it, produces an interactive
UEFI module-map web page, emulates the major segments (SMM, DXE, PEI) hunting for
defects, and can build new capsules from declarative recipes. Tooling is organised as
phase / architecture / vendor plugins so Intel can be added later. v2 is self-contained:
everything worth keeping from v1 and its precursors is copied in, in a layout designed
for reuse, and v2 never depends on the old repositories at run time.

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

Conventional defaults chosen during planning (open to change):

- Workspace folder is exactly `{capsule file stem}_{YYYYMMDD-HHMMSS}`; a same-second
  collision gets `-2`. **Identity is the SHA-256** (v1: every Flashback image is named
  `A5557.cap`), stored in `manifest.json` and `firmware/index.json`.
- Re-dropping an identical file creates a new dated workspace linked to earlier ones with
  the same SHA; a content-addressed result cache keeps the re-run cheap.
- Unrecognised files move to `firmware/_rejected/{name}_{datetime}/` with `reason.json`.
- `run.py` processes the dropzone and exits; `run.py watch` is the optional service loop.
- v2 **never flashes** and contains no flash-write code path.

## 3. Research headline findings

### 3.1 What v1 has and lacks

| Area | State in v1 | Consequence for v2 |
|---|---|---|
| Ingestion | None. 5–6 copy-pasted in-memory walkers; capsule header stripped by assumption | One extractor, written once, that writes a tree |
| Module map | Static page, 70 KB vanilla JS, opens over `file://`; no dependency graph | Lift the shell; rewrite views; add graph, SMI, NVRAM, findings |
| SMM static analysis | **Strongest asset**: `trace.py` → `smi_registry.py` (130 registrations) → protocol graph → handler audit | Lift; rewrite the taint model |
| SMM emulation | One proof of concept (3 Unicorn files) at hardcoded RVAs | New general harness; PoCs become regressions |
| PEI / DXE emulation | **None.** A C harness runs modules natively, x64 only, unsandboxed | New; port its stub semantics only |
| PEI dispatch simulator, DEPEX cycle detector | **Do not exist** (cited scripts were never committed) | New, shared by analysis and build gates |
| Capsule build | Home-grown, hardware-proven byte splicing; 75 near-duplicate scripts | 12 operation types + declarative recipes |
| Build gates | Grew over 125 revs; several are stubs that always pass | Gate framework where "not run" ≠ "pass" |

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

None of v1's rev5–rev125 capsules is on disk. A5557's FV_MAIN digest (`a0326063…`) is the
registry's vendor-baseline entry (which also lists `sentinel-phase1-rev4` as identical in
the measured range).

### 3.4 Facts checked against the bytes

- A5557 capsule header: GUID `4A3CA68B-…` (UEFITool: `APTIO_SIGNED_CAPSULE_GUID`),
  `HeaderSize=0x1000`, `RomImageOffset=0x1000` at +0x1C. The payload offset must be read
  from `RomImageOffset`, not assumed.
- All 8 LZMA sections in A5557: dictionary 16 MiB, props 0x5D, true size stored; no
  LZMA-F86 sections. FV_MAIN is outer FV `5473C07A…` at SPI 0x1000000, size 0xB50000.
- Both PEI copies decompress to the image v1's PEI builds started from
  (`0bfe931b…`, 0x500020 B). Re-running zlib level 9 does **not** reproduce the vendor stream.
- `$FID` (tag `A5557`, date, `ALASKA A M I`) is readable at 0x1C012EC; `AGESA!` and the
  board name are inside compressed sections ⇒ identification must run after extraction.
- `unicorn 2.1.4` ships a `cp37-abi3` wheel (should install on 3.14 — to be confirmed in
  P0); Qiling's last release is 2023-08 with ~31 dependencies and Intel-only SMM.

### 3.5 Known v1 bugs to fix while lifting

- `pei_research/tools/ppi_graph.py:58` drops every terminating PPI descriptor (flags `0x80000010`).
- `tools/depex/analyze.py` uses section type 0x18 for MM DEPEX (it is 0x1C), ignores
  AND/OR/NOT, and mis-computes GUID_DEFINED offsets for 8-byte headers.
- `scan_smm_handlers.collect_files` reads large-file size at +20 instead of +24.
- `analysis/_tools/fw_walk.py` sends EFI-compressed sections to LZMA; nothing decodes Tiano.
- `smi_registry.py:220-222` skips AMD FCH dispatch protocols, so FchSmm `06BE9C49`,
  `395B0181`, `34F85EE2` have no registry entries.
- `handler_audit.py` treats SW-SMI `r8` as a CommBuffer (it is `EFI_SMM_SW_CONTEXT`); taint
  dies at joins and calls; `GetVariable` output is not a taint source.
- `fv_inventory.py` aborts a whole FV walk on one bad FFS size.
- `tests/depex_verify.py` and `tools/spi/verify_psp.py` are stubs that report success.

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
  `pe_entry_anomaly` found nothing), yet `sentinel-uefi/docs/architecture.md:34-37` still
  repeats them. `docs/LESSONS.md` records them as retracted.

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

v2 is designed so none of this class of data enters its git history (§4.7 privacy).

## 4. Architecture

### 4.1 Repository layout

```
run.py                       thin entry: sentinel.cli.main()
pyproject.toml               uv-managed; deps: pyyaml, pydantic, capstone, brotli, unicorn
sentinel/                    shared core library (generic PI/UEFI; no vendor knowledge)
  cli.py workspace.py doctor.py profile.py hashing.py guid.py diff.py log.py
  fmt/        capsule fv ffs section pe(PE32/PE32+/TE) depex esl hob magic
              compress/{lzma_bcj, tiano, brotli_, registry}
  ingest/     dropzone, sniff, index
  extract.py  the single walker → model + tree
  model/      schema, store, graph, dispatch (PEI+DXE+MM evaluator, cycles), observed
  names/      resolver, providers, harvest, importers/{edk2, uefitool, v1}
  plugins/    api, loader, contrib
  pipeline/   dag, runner, worker (subprocess isolation), cache
  findings/   schema, overlay, coverage, report
  evidence/   bundle schema, attach, override rules                      (§4.7)
  disasm/     trace (from v1 trace.py), cfg, provenance (gBS/gSmst by origin)
  emu/        engine, loader, memory, stubs, layers/{pei,dxe,smm}, devices, hunters, coverage
  build/      recipe, editor, ffs wrap, compress, ops, gates
  map/        build.py, static/{index.html, assets/{core.js, style.css, views/*.js}}
tools/<phase>/{common,amd,intel,ami}/[<oem>/]     plugins                (§4.2)
uefi/<phase>/<compat>/…      Sentinel firmware module sources            (§4.12)
collect/                     read-only capture kit run on a live board   (§4.7)
recipes/<board>/*.yaml       build recipes
data/        vendors.yaml, profiles/<id>.yaml, boards/<board>/{lineage,known_bad,pcr0,recovery}.yaml,
             guids/{edk2,vendor,learned}/, names/overrides.yaml, catalogues/, annotations/global.yaml
firmware_dropzone/           input (contents git-ignored)
evidence_dropzone/           evidence bundles in (contents git-ignored)
firmware/                    workspaces, _rejected/, _cache/, index.json (git-ignored)
docs/        LESSONS.md, architecture.md, plugin-authoring.md, findings-doctrine.md, safety.md
tests/       unit/, synth/ (fixture generators), golden/<fixture>/, fixtures.yaml, fixtures/bin/ (git-ignored)
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
  → OEM → board with provenance. `tools/smm/amd/fch_dispatch.contrib.yaml` closes the
  FchSmm registry gap with no new code.
- **Three outcomes never collapse.** *Not applicable* (match failed), *could not run*
  (missing dependency, unknown profile field, undecodable input, upstream failure), *ran*.
  `RAN` with an empty coverage list is recorded as `RAN_NO_COVERAGE` and reported as
  "unknown", never "clean" — the guard against v1's unrun-scan-reads-as-clean failure.
- **Isolation.** Each analyzer runs in a subprocess (`python -I -m sentinel.pipeline.worker`)
  with timeout and `RLIMIT_AS`; a crash is an `ERROR` for that plugin only.

### 4.3 Ingest and identification

- Dropzone scan: top-level regular files, any extension; files still being copied
  (size/mtime changed in the last 3 s) are skipped until stable.
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
  OEM/board (`$FID` tag, SMBIOS defaults), AGESA version.
- **Platform profile** (`data/profiles/*.yaml`) supplies what a capsule cannot: CPU family
  and SoC gates, default Setup/PCD values, boot mode, SMRAM assumptions. Most specific
  match wins; no match → `generic-<silicon>` with every field `unknown`, and plugins that
  need a gate report *could not run* (never assume both per-family twins are live). The
  resolved profile is frozen as `profile.lock.yaml`; its digest is in every cache key.

### 4.4 Extractor, workspace and model

One walker (`sentinel/extract.py`) merging v1's `interactive_map/tools/fv_inventory.py`
recursion with `encryption/tools/fwcorpus.py`'s on-disk dump, fixing: resync after a bad
FFS size; GUID_DEFINED data offset from the header; LZMA-F86 (pure-Python BCJ); Tiano/EFI
decompression (pure-Python port); Brotli; CRC32 sections; TE images; FFS3 large files.
A second pass walks extra roots that layout plugins find outside any FV (the AMD
BIOS-directory 0x62 PEI container).

```
firmware/{name}_{datetime}/
  manifest.json          sha256, size, original_name, container_chain, parent, same_image_as
  input/<original file>  + SHA256SUMS            (read-only)
  image/payload.bin      unwrapped image; capsule_header.bin
  profile.lock.yaml
  blobs/<sha[:2]>/<sha>  content-addressed bodies
  extract/<region>/<fv>/<NNN>_<GUID>_<Name>/…    browsable tree (hardlinks into blobs)
  modules/<GUID>_<sha8>/ image.efi|.te, manifest.json, strings.txt
  model/                 image regions volumes files sections modules graph names identity observed
  analysis/<plugin-id>/  result.json + outputs
  emu/                   per-module run records, coverage, proposed plugin data
  evidence/<capture-id>/ attached evidence bundles                        (§4.7)
  findings/              findings.jsonl, coverage.json, summary.md
  overlay/               annotations.yaml, names.yaml, profile.yaml       (hand-edited)
  map/                   index.html, assets/, data/*.js
  build/<recipe>_<datetime>/                                              (§4.11)
  logs/  state/
```

Model files carry `schema_version`; ids are paths (`fv@0x1000000/f12/s2/fv0`).
`regions.json` lists unclaimed byte ranges explicitly. `graph.json` edges carry
`role` (depex_requires, installs, locates, notifies, references), `method` (depex,
literal, trace, emulation, measured), `tier` and `confidence`; the core emits only
DEPEX and reference edges because byte-proximity scans confuse install with locate.
Static, emulated and measured facts live in separate layers and never overwrite each other.

### 4.5 Pipeline and CLI

Stage order: ingest → extract → layout → extract (pass 2) → identify → profile → names →
index → evidence attach → static analyzers → emulation → findings → map. Build runs on
demand. The DAG comes from `needs`/`produces`; cache key = plugin source hash + inputs +
profile digest + contribution digests.

```
run.py [process] [--file F] [--profile ID] [--only S…] [--until S] [--quick|--deep] [-j N]
run.py rerun <ws> [STAGE | --from STAGE] [--force]
run.py list | show <ws> | findings <ws> | plugins [--for <ws>] | map <ws> [--open]
run.py diff <wsA> <wsB>          run.py watch
run.py evidence add <ws> <bundle> | evidence list
run.py emulate <ws> [--module G] [--hunter H]
run.py build <recipe.yaml> [--variant V]
run.py annotate <ws> <finding> --confirm|--dismiss|--note … --reason …
run.py guids import … | guids harvest
run.py edk2 bootstrap            fetch pinned EDK2 + apply uefi/patches   (§4.12)
run.py doctor [--json]
```

`doctor` reports every capability and which stages it disables (today: unicorn missing,
BaseTools missing, qemu broken). Exit codes: 0 all ran, 1 partial, 2 usage, 3 everything
rejected, 4 core environment broken. Findings never change the exit code unless asked.

### 4.6 Findings, coverage and overlays

v1's doctrine becomes the schema: **every automated result is a lead, never a verdict.**

- Finding: `id` (stable fingerprint of rule + module SHA + normalised location), `class`,
  `subject`, `location` as (module SHA, RVA), `tier` (behavioral / descriptive /
  structural), `confidence` (certain / likely / unknown), `evidence[]`, `fidelity[]`
  (models, phantoms, unmodelled hardware on the path), `status = "lead"`.
- Corrections go in overlays (`overlay/annotations.yaml` per workspace,
  `data/annotations/global.yaml` keyed by module SHA so they follow identical modules into
  other capsules). Generated files are never hand-edited.
- `coverage.json` is a plugin × subject matrix; the report and the map list everything
  that was **not** checked: plugins not applicable / could not run / errored, undecodable
  sections, modules never emulated, profile fields left unknown.

### 4.7 Board evidence — first-class

v1 has 28 capture directories in three inconsistent layouts with **no machine-readable
manifest**; identity is a rev number in a folder name, a capsule SHA from rev101, and a
measured digest only from rev97. v2 normalises this.

**Bundle.** `evidence/<board>/<boot_utc>_<fvmain8>/` with `manifest.json`, `raw/`,
`decoded/`, `redacted/`. Manifest fields: `bundle_id`, `schema_version`, `captured_utc`,
`boot_id`, kernel/OS, lockdown and Secure Boot state, `board_profile`, capture tool +
version, and

- `identity`: `pcr0_live`, `pcr0_replay_match`, `blob_events[]` (description, base,
  length, digest), `candidate_images[]` (capsule SHA, match method), `confidence`
  (**measured / claimed / inferred**);
- `items[]`: type, path, SHA-256, source interface, decoder + version,
  `payload_required` (true when it only exists because a telemetry module was flashed),
  `sensitivity` (public / identifier / secret);
- `active_actions[]`: anything during capture that was not a pure read.

**Attaching to an image — generic, with board knowledge as data.** The TCG event log's
`EV_EFI_PLATFORM_FIRMWARE_BLOB(2)` events are self-describing (description, base, length).
v2 hashes each described range in every ingested image under two interpretations of
`base` (SPI offset; memory-mapped address) and records matches. The board profile says
which interpretation applies, which events identify the image versus runtime, and what is
**not** measured (on this board: the PEI volume and NVRAM). Because unmeasured regions
mean several images share one digest (v1's registry records such collisions), a bundle
binds to a **set of candidate images**, never silently to one.

**Known-but-absent images.** Most v1 captures (rev97–124) were taken on capsules that are
not on disk. v2 keeps image records keyed by capsule SHA and measured digest
(`data/boards/<board>/images.yaml`, seeded from v1's 66-entry `pcr0_registry.json` and
`pei_registry.json`) so those bundles import now and bind when the image is ingested.
Attachable today: rev84 (partial image) and the rev70 dump.

**Importer plugins** (`tools/evidence/…`), each producing facts in the model's `observed`
layer with a join key:

| Evidence | Importer | Join key | Needs payload? |
|---|---|---|---|
| TCG event log, PCRs | `common/tcg_log` (from `tools/security_state/tcg_log_events.py`) | digest ↔ image range | no |
| Secure Boot stores, mode bytes, MOR, variable listing | `common/uefi_vars` | variable name + GUID | no |
| Live ACPI tables | `common/acpi_live` | table signature + OEM table id; diffed against the image's own tables | no |
| Runtime map, Memory Attributes Table | `common/memory_map` | address range | no |
| Kernel log, reset reason, DMI | `common/host` (redacted on import) | — | no |
| MSR snapshot, PSP sysfs, FCH registers | `amd/security_state` (from `read_security_state.py`, `c6_validate.py`) | MSR number, MMIO address | no |
| Flash-window / SPI dump | re-ingested as an image, linked to the bundle | — | no |
| HOB list | `common/hob` | HOB GUID + address range | yes |
| PEI dispatch trail | `sentinel/pei_trail` | **FFS file GUID** | yes |
| MM audit / census ("did not load") | `sentinel/mm_audit` | full module GUID (v1 used only the first 8 hex digits — fragile) | yes |

**Override rule.** Static, emulated and measured facts are separate layers. Where they
disagree the measured fact wins *for that board and boot*, the disagreement is reported
as its own finding class (e.g. "statically 'runs', never dispatched"), and the caveats
travel with it (one boot only; APRIORI modules run before a trail listener exists, so
absence there proves nothing).

**Capture kit** (`collect/`): one script producing a bundle with the generic items using
stock Linux tools (TCG log, PCRs, efivarfs listing and stores, ACPI tables, runtime map,
kernel log, DMI, `tpm2_getcap`, MSRs; AMD extras). Strictly read-only by construction.
**Excluded from v2 entirely**: v1's deliberate SMI test scripts (they write `/dev/mem` and
fire SW SMIs), the NVRAM pruner's `--execute` mode, and key-enrolment scripts.

**Privacy.** `raw/` never enters git and never reaches a generated page. Import redacts
serials, UUIDs, MAC addresses, SSIDs/BSSIDs, hostnames, the TPM EK and disk GUIDs; only
`redacted/` and derived facts are publishable. `evidence/`, `firmware/` and both
dropzones are git-ignored; a test fails if a tracked file matches identifier patterns.

### 4.8 Interactive map

Self-contained, offline, opens over `file://` (classic scripts, lazy-loaded data shards;
no framework). Lifted from v1's `interactive_map/assets/app.js`: hash routing, theme,
drawer, search. Views:

1. Overview — identity with evidence, profile, container chain, counts, coverage, top leads
2. Flash layout — derived from `regions.json` (nothing hardcoded), nested FV tree, unclaimed space
3. Modules — table with phase, name source, PE hardening, findings
4. Dispatch — PEI and DXE order lanes, unsatisfied DEPEX, cycles, profile assumptions
5. Dependency graph — layered ego graph (1–2 hops) across PPI / DXE / MM; edge style = evidence tier
6. SMI surface — type, key, module, handler, reachability, leads
7. NVRAM variables · 8. Setup/HII questions
9. Findings with overlay status and coverage · 10. Emulation results
11. Evidence — measured vs static differences
12. Diff against another workspace (parent vs built child, or any two) · 13. Glossary

### 4.9 Static analysis plugins (lifted from v1)

| Plugin path | From v1 | Action |
|---|---|---|
| `sentinel/disasm/trace.py` | `smm_research/tools/trace.py` | lift |
| `tools/smm/common/registry.py` | `smi_registry.py` | lift; dispatch table becomes contributed data |
| `tools/smm/amd/fch_dispatch.contrib.yaml` | — | new (FCH GUIDs `881B4AB6`, `87E2A6CF`, `A365240E`, `736102F1`, `7051AB6D`, `91288FC4`) |
| `tools/smm/common/{protocol_graph,handler_audit}.py` | same names | lift graph; rewrite audit taint |
| `tools/smm/ami/` | AMI SMM core layout, RT shim `395C33FE`, NvramSmm rules | parameterise as data |
| `tools/smm/amd/asus/` | `750890A6` hook table, SW SMIs 0x96/0x98/0x99/0xDA | data + decoders |
| `tools/acpi/common/aml_smi.py` | `smm_live_audits/smm_audit.py --from-dsl`, `aml_smi_callers.py` | lift |
| `tools/dxe/common/{bs_census,bs_table_map,win_strings}.py` | `bootservices/tools/*` | lift |
| `tools/pei/common/ppi_graph.py`, `pcd_db.py` | `pei_research/tools/ppi_graph.py`, `iommu/tools/decode_pcd_db.py` | lift, fix bug |
| `tools/pei/amd/` | AGESA chain, CPUID family gates, 0x62 container | parameterise as data |
| `tools/layout/amd/psp_dirs.py` | `tools/spi/psp_inventory.py` | parameterise (walk from EFS, no fixed offsets) |
| `tools/nvram/ami/nvar.py` | `read_nvram.py`, `nvar_dump.py` | rewrite: locate by store GUID |
| `tools/hii/common/ifr.py` | `tools/hii/ifr_export.py` | lift |
| `tools/crypto/common/{census,key_material,esl}.py` | `encryption/tools/{crypto_census,canon,key_material_scan,esl}.py` | lift |
| `tools/crypto/ami/`, `tools/crypto/amd/asus/` | `ami_digsig_kat`, `pss_probe`; `fw_key_compare` | parameterise |
| `tools/image/common/{pe_hardening,entry_anomaly}.py` | `pe_entry_anomaly.py` | lift, add TE |
| `tools/oprom/common/`, `tools/oprom/amd/atom.py` | `fw_walk.find_oproms`, `atom_disasm.py` | lift |
| `tools/image/common/cve_patterns.py` | hand-applied byte/instruction patterns | new, catalogue-driven |

Dropped: Windows-driver analysis trees, per-rev annotations, Sentinel build lineage gates.

### 4.10 Emulation harness and hunters

**Engine.** Own harness on Unicorn 2.1.4 behind an `Engine` interface (IA32 + x64).
Qiling rejected (stale release, 31 dependencies, Intel-only SMM, no Communicate path);
ideas borrowed. Whole-image boot is impossible (signed PSP/AGESA stage), so emulation is
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
  lead that depended on it is footnoted.
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
- PEI (IA32) — PEI Services, PPI database with notify, HOB list, PCD PPI, FFS services,
  and a new PEI dispatch simulator.
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
| later | Hardening state, dispatcher quirks, dead code | Lock bits at end of init; duplicate SW values; static-vs-dynamic registration diff |

Embedded keys and known-CVE patterns stay static (§4.9).

**Honesty.** Each module is labelled `EMULATED_COMPLETE`, `EMULATED_PARTIAL(reasons)` or
`NOT_EMULATED(reason)`; a hunter below a coverage threshold reports "insufficient
coverage", never "clean". The static-vs-dynamic registration difference is itself a signal.

**Output policy.** Results are emulator evidence (source field, sink, rule site, trace).
v2 does not generate stand-alone exploit payloads for real hardware.

**Regressions (semantic, no RVAs).**

1. BIOS 2503 — rediscover the `WpBufAddr` variable-supplied pointer write in `750890A6`
   via SW SMI 0xA0. (Running it on A5557 answers an open question, not a regression.)
2. BIOS 2503 — reproduce the NvramSmm store-rule refusal matrix through the module's real
   interfaces; if its entry cannot be emulated the case reports `NOT_EMULATED`.
3. A5557 — the hardware-confirmed callout in `34DF9766`/`2EF2DEE2` under the Zen 5 profile;
   under a Phoenix profile the twin `EF9A853E` registers instead.
4. Dispatch what-ifs on A5557 — the rev86–90 DEPEX cycle and the rev86/87 lost
   MonotonicCounter architecture protocol, built as model overlays.

### 4.11 Capsule builds, gates and safety

**Recipes** (`recipes/<board>/<id>.yaml`, pydantic-validated): base pinned by SHA-256;
inputs pinned by SHA-256; **explicit volume selectors** that must resolve to exactly one
node (the "largest LZMA section" heuristic is banned); operations with guards; expected
file-set delta; waivers; `exceptions` for protected regions; variants.
Inheritance defaults to `flatten` (re-apply the whole ancestry to the pinned base); every
operation's post-condition becomes a lineage assertion, so carried-forward changes are
checked by construction and most of v1's 92 hand-written invariants become derived.

**Operations** (twelve types cover everything v1 ever did):

| Home | Operations |
|---|---|
| `sentinel/build/ops` (generic PI) | `insert_ffs`, `replace_body`, `replace_file`, `remove_ffs` (→ same-size pad), `patch_pe`, `replace_raw`, `edit_apriori`, `grow_lzma_section` |
| `tools/build/ami/` | `ami.edit_nvar`, `ami.replace_default_key` |
| `tools/build/amd/` | `amd.psp_edit`, `amd.apcb_token`, `amd.pei_container` (0x62, both copies, PE→TE rebase) |

- FFS wrapping is **pure Python** (BaseTools absent; GenFfs output needed patching anyway);
  GenSec/GenFfs/GenFw are optional test oracles. State byte and pad-header checksum follow
  the volume's erase polarity (generic PI, not AMI-specific).
- **Single compressor**: LZMA1 alone-format, lc3 lp0 pb2, dictionary from the original
  stream unless the recipe overrides with a reason, true size patched, decompressed length
  preserved, section never shrunk, overflow refused. Edits inside LZMA-F86, Tiano or
  Brotli are refused at first.
- Proven kernel to lift: `firmware/builds/ffs_replace.py:38`, `ffs_replace_batch.py:26,66`,
  `build_rev42.py:39`, `build_rev81.py:232,261`, `rev123_steps.py:128-389`,
  `rev124_steps.py:127-215`, `rev111_steps.py:97-193`, `rev118_steps.py:66-225`,
  `build_sentinel_mm.py:341-411`, `lineage_invariants.py:58,1602`.

**Gates** return `pass | fail | not_run | inconclusive | waived | error`; any `not_run` or
`inconclusive` makes the verdict `INCOMPLETE`. First set: same size; container header
passthrough; no byte outside declared spans; unchanged files byte-identical; file-set
delta; FFS integrity; rewritten sections decode; architecture protocols still installed;
dispatch model with cycle detection; producer parity; reproducible double build; derived
lineage; AMD PSP signatures, dual EFS and PEI copies identical. Board-specific data
(lineage, known-bad SHA ledger, PCR0 ranges, recovery path) lives in `data/boards/<board>/`.

**Output** lands in `firmware/<base-ws>/build/<recipe>_<datetime>/` with a full manifest,
then is **re-ingested automatically as a child workspace** with a parent link, so the map
shows parent-vs-child and hunters re-run only on changed modules.

**Safety (code-enforced).** No flash-write path exists (a lint test enforces it). Refused
without an explicit recipe exception: any write inside PSP/BIOS directories, EFS,
PSP-referenced blobs, microcode, NVRAM or the boot block; a size change; a base on the
known-bad ledger; an ambiguous selector; a removal without the parity gate; duplicate
GUIDs; unpinned inputs. Every build emits `FLASHABILITY.md` stating which gates ran, what
static checks can never prove, **that the capsule header signature is invalid after
modification** and installable only through an unauthenticated recovery path, and the
board's declared recovery path (none declared → `NOT FLASHABLE`). v1's flash-preflight and
refuter become generated `CHECKLIST.md` and `REFUTER_BRIEF.md` for a human.

### 4.12 `uefi/` — Sentinel firmware module sources

**What exists in v1**: 155 INFs — 31 shipping board modules (17 DXE, 4 runtime, 8
Standalone MM, 1 MM core, 1 PEIM), 55 libraries + 3 mocks, 18 test probes, 37 host unit
tests, 11 old-line (Traditional SMM) modules; ~84k lines of C in `SentinelMmPkg`, of which
~37k are **generated from upstream by 12 `derive_*.py` scripts**. v1 cannot build today:
upstream edk2 is git-ignored and absent, every gate script defaults to a path that no
longer exists, and the Makefile build targets are stale.

**Layout** — filed by phase, then by what the code is compatible with:

```
uefi/
  Sentinel.dec                 one package root for all of uefi/** ([Includes] → include/*)
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
                               DSC/FDF, platform derive scripts, board PCD values, module/FFS manifest, DEPEX pins
  patches/edk2/                0001 PKCS#7 messageDigest fix (upstreamable), 0002 (legacy)
  tests/{host, mocks, probes/{common,amd,ami,qemu}, gates, guards}/
```

**Rules.**

- **Upstream EDK2 stays external.** `run.py edk2 bootstrap` fetches the pinned tag
  (edk2-stable202602, commit `52bbcffe`) plus its submodules into a git-ignored folder and
  applies `uefi/patches/edk2/`. The build puts the repo and that checkout on
  `PACKAGES_PATH`, which is what lets directories be rearranged freely.
- **FILE_GUIDs and BASE_NAMEs never change** — recipes, guards and evidence decoders key on them.
- **Derived code keeps script + upstream pin + committed output together**, with a check
  that regenerates and diffs; scripts take source/destination as arguments instead of
  assuming the working directory.
- **ABI pairs move together.** Each firmware header that defines a published variable or
  record travels with its Python decoder (now `tools/evidence/sentinel/…`) and its guard
  test: census v10, MM audit, HOB snapshot, PEI telemetry, MM event log, crumbs, CF9 trap,
  Sx trail, hand-off records.
- **Two steps.** *P7a* copies each module to the folder of its dominant compatibility and
  proves it still builds. *P7b* splits the nine modules that mix generic and vendor code
  (SwDispatchMm, VariableMm, AcpiEnableDxe, BootDumpDxe, MmIplDxe, SecurityStubDxe, the PEI
  telemetry pair, PlatformBootManagerLib, PageAuditLib) into a generic core plus vendor
  backend libraries — this is what makes them reusable on another chipset.
- Built modules go to a content-addressed artefact store with provenance and are consumed
  by recipes by SHA-256.
- Generic guards worth keeping for any module inserted into foreign firmware: no dynamic
  PCD reads, no side effects inside `DEBUG()`, exact DEPEX pins, no test probes in output,
  derived files reproduce.
- Reusable OVMF gates (generic "Standalone MM under OVMF" harness, post-lock attacks,
  ExitBootServices, MMIO negative test) move to `uefi/tests/gates/`; they need a working
  QEMU, which this host lacks today.

Unverified: whether BaseTools accepts INFs that sit outside the directory of the `.dec`
they use (the single `Sentinel.dec` at `uefi/` avoids the question); submodule versions;
whether some old-line DXE binaries are still carried in the latest board image.

### 4.13 Naming

Pluggable resolver; all hits kept with provenance and shown in the map. Modules: UI
section → workspace overrides → global overrides → vendor GUID databases → edk2 database →
PDB name. Seeds: v1's edk2 databases (1,367 modules, 780 protocols), UEFITool `guids.csv`
(12,182, BSD-2-Clause, attributed), v1's hand-identified names. `guids harvest` grows a
reviewed `learned` database from every processed capsule.

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
| Check consumers before removing a module | `producer_parity` gate mandatory on `remove_ffs` |
| Carried-forward changes get lost | Flatten inheritance + derived lineage |
| Pad checksum / State byte bricks | `ffs_integrity` gate; one implementation |
| LZMA parameter mistakes corrupt images | Single compressor rule |
| PSP regions are untouchable | Refusal without recipe exception; signature gate |
| Corrections never edit generated output | Overlays |
| Generated code must not be hand-edited | Derive scripts kept with the modules (§4.12) |

`docs/LESSONS.md` carries the full narrative (rev history arc, each brick and its rule).

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
