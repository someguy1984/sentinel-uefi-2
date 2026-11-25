# ASUS ROG Strix X870E-E — board plan

> **Status:** plan, 2026-11-25. Nothing here is implemented. This file holds what was
> moved out of the main [`PLAN_v2.md`](../../PLAN_v2.md) when the project was re-centred on a
> framework for any firmware image. Section numbers (§…) refer to the main plan. Phases
> B0–B5 are this folder's own; P0–P9 are the main roadmap's.

## 1. What this folder is

The board v1 (`../sentinel-uefi`) was built on: ASUS ROG Strix X870E-E Gaming WiFi, AMD
AM5 (a Ryzen 9950X in v1's work), AMI Aptio, AMD AGESA. It is the framework's first board
folder (§4.16) and its richest source of oracles: eight months of builds, captures and
studies. The framework never depends on it; it depends on the framework.

## 2. Decisions and open questions

Decided by the owner during the coverage review (2026-11-24):

| Decision | Choice |
|---|---|
| D2 v1 capsules | None of v1's built capsules (rev5–rev124) is or will be available as a fixture. Validation uses what v1 tracks in git (§7, §8) |
| D3 Board build line v1 → v2 | **Left open** (below). No cut-over gate is designed |

Open (owner):

- **Board build line (D3).** Keep building the board's revs in v1 until v2's P7 reproduces
  v1's module set; adopt a recovered image (a chip dump or the flash-medium copy) as a
  recipe base (`adopt`, §4.11); or re-derive the line from A5557, which needs inputs v1
  never tracked. No phase depends on the answer; C-line recipes stay plan-only until it
  is given (B5).
- **Prebuilt old-line binaries** (v1 `edk2-modules/`, 40 `.efi`): fixtures by SHA so that
  old-line legacy recipes can be exercised, or those recipes declared unbuildable.
- **Raw third-party scan files** (the CycloneDX SBOM/CBOM files in v1 git): kept outside
  git with only v1's triage verdicts imported (default), or importable data.
- **Embargoed material.** Expected-lead entries and overlay text about undisclosed
  findings stay out of tracked files by default (§4.6). This plan itself already names
  the subject of v1's embargoed report (§4.10, regression 1): whether that wording may
  stay in a pushed file is the owner's call.

## 3. Images of this board on disk

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

Checked on 2026-11-25:

- `A5557.cap` is BIOS **1715**: its `$FID` carries `17 15` where the vendor's 1605, 2301
  and 2503 releases carry their own numbers. (`A5557.cap` is also the file name ASUS
  Flashback expects for every release of this board, so the name identifies nothing.)
- Three vendor releases sit in `/home/j/Videos`: 1605 (`719b613c…`), 2301 (`49c92b6f…`,
  zipped) and 2503 (`0ed48d24…`). None is 1715, so the provenance question above cannot
  be settled from files on disk: it needs the vendor's 1715 download, hashed once.
- `lol.cap` (`dd01cc3b…`), beside the 1605 release, carries the 1605 `$FID` but is not
  the vendor file. Origin unknown.

## 4. Evidence captured on this board

v1 keeps board evidence in 26 `efivars/rev_*` directories and, beyond them, in
`smm_live_audits/` (the only measurements for rev82–85), `hob_analysis/`, `census/`,
`efivars_dump/`, `pei_research/rev125/evidence/` and the `board/` and flashed-review
directories under `smm_research/studies/`; the two `efivars/store_*` directories are
key-store snapshots, not captures. Layouts differ and there is **no common manifest**
(newer captures carry `captured_utc.txt`, `c6_state.json` and, from rev101,
`capsule_sha256.txt`). Identity is a rev number in a folder name; a measured FV_MAIN
digest exists from rev82, saved binary event logs only from rev97. Many captures hold
text already decoded by v1's readers rather than raw records. v2 normalises this.

**Known-but-absent images.** Every v1 capture of a Sentinel rev was taken on a capsule
that is not on disk (D2). v2 keeps image records keyed by capsule SHA and measured digest
in `data/boards/<board>/images.yaml`, seeded from v1's 66-entry `pcr0_registry.json`,
from `pei_registry.json` (container and stream digests, inflated SHA, build id) and from
the per-rev `modules.sha256` of v1's board builds, so that module-digest events bind too.
Those bundles import now and bind when an image is ingested. Attachable today: the rev84
measurement (`smm_live_audits/runs/2026-10-27`: a YAML event log and `audit.json`;
`efivars/rev_84` itself holds no event-log file) against the rev84 flash-window image,
and the rev70 dump — by claim only, since no rev70 capture carries a measurement.

One importer exists only for this board's history and lives in
`boards/asus-x870e-e/tools/evidence/legacy/`:

| Evidence | Importer | Join key | Needs payload? |
|---|---|---|---|
| Older record layouts (shield, census and audit versions of the vendor line) | `legacy/*` (this folder) | as the current form; needed to convert the rev73–85 captures | yes |

## 5. Regressions this board contributes

Specifications in `boards/asus-x870e-e/tests/regression/` (§4.10 gives the format). They
run when this folder's fixtures are present and gate no core phase.

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

## 6. Recipes and lineage

v1's builders are distilled into `recipes/legacy/*.yaml` here (§9 of the main plan), with
`why` and `decided` taken from v1's prune lists (§4.11).

Of v1's 91 hand-written invariants about half are markers inside Sentinel-built bodies;
others are whole-image absence scans, structural guards or pins on vendor bytes, and only
a minority are post-conditions of an image operation. B4 ships a table mapping every v1
key to derived / marker / scan / gate / dropped (with the reason).

Build oracles available without any v1 capsule: the decompressed PEI hashes of rev123
(`21340215…`) and of rev124-nopeim (`98882582…`), both reachable from this board's
baseline image by removal operations alone; and rev34's bad pad, which must be rejected
(`firmware/builds/_rev34_work/` in v1). The rev123 hash is recorded only in v1's prose
(its `CLAUDE.md`, the rev123 study and `docs/rev-history.md`), not in `pei_registry.json`:
B4 freezes it into `tests/` with its source.

## 7. EDK2 workings

`edk2/` here holds what is specific to this board and builds against the module library
in `uefi/` (§4.12): the platform (DSC/FDF, board PCD values, the manifest of all 43
shipped modules, DEPEX pins, `assertions.yaml`), `derive/` for the constants compiled in
from this board's image, and the two board modules, AcpiNeuter and XhcS0w.

v1's CryptoDxe (`FEA01457`, carrying the PKCS#7 fix) was built from a configuration that
was never tracked; B5 adds it, pins its crypto library and aims at v1's body SHA; until
that succeeds the body is an `imported` artefact.

## 8. Board track

| Phase | Needs | Delivers | Exit criterion |
|---|---|---|---|
| **B0 Folder** | P0 | This folder's scaffold: `README.md`, `board.yaml`, `profile.yaml`, `known_bad.yaml`, `recovery.yaml`, and `docs/operator-loop.md` written by hand from v1's `CLAUDE.md`, `/flash-preflight`, `firmware-refuter.md` and `tests/boot_checklist.md` | The folder passes the hygiene tests; the operator loop holds procedure and hazards only, never live board state |
| **B1 Images** | P1 | `tests/fixtures.yaml`; `images.yaml` seeded from v1's registries; golden outputs | The baseline capsule, the rev70 dump, the rev84 flash window and the 2503 release ingest with golden outputs; the provenance question of §3 is settled, or recorded as undetermined with the reason |
| **B2 Evidence** | P2 | v1's captures converted to bundles; the legacy decoders | The rev84 capture binds to its flash-window image by measured digest; the rev97–124 bundles import as known-but-absent |
| **B3 Oracles** | P4, P5 | v1's module judgements as global annotations; the five regressions of §5 | At least 130 SMI registrations on the baseline image, and for each of the four modules whose registrations v1 recovered by hand (§3.5) a record of whether it closed statically or through the emulated census (the conditional one can only close through the census); regressions 4 and 5 pass on the static model, 1 and 2 under emulation; regression 3, still an open question on hardware, reports reachable / not reachable / insufficient coverage |
| **B4 Lineage** | P6 | Legacy recipes; the invariant mapping table; `pcr0.yaml`, `assertions.yaml` | The two PEI hashes of §6 reproduce; rev34's bad pad is rejected; every v1 lineage key is mapped or dropped with a reason |
| **B5 Board build** | P7 | The board platform and board modules built from `edk2/` here | Board and PEIM builds succeed with FILE_GUIDs unchanged; body SHAs are compared with v1's per-rev `modules.sha256`, and every difference is explained (stack cookie, stamp, compiler) or reported |

## 9. Unverified, for this board

What ASUS USB BIOS Flashback actually verifies; whether `A5557.cap` is the pristine
vendor 1715 image; the meaning of the extra `AA55AA55` at 0x1000 in `02102026.bin`;
whether some old-line DXE binaries are still carried in the latest board image; whether
today's compiler reproduces v1's module bodies beyond cookie and stamp (v1 recorded no
compiler version).
