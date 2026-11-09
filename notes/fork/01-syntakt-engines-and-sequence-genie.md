# fork/01 — The Syntakt engines and Sequence Genie in model27 (investigation)

**Status: investigation, 2026-11-08. Nothing is built.** The maintainer (Nicolai) asked how best to add two things
"as a package": the Syntakt engines, and Sequence Genie from AveyCole's Supermodel firmware. This note says what each
one is today, what stands in the way, and in which order to go. Addresses are VAs of OS 1.13.

Sources read: this repository at `b7ec5d3`; [AveyCole/Supermodel](https://github.com/AveyCole/Supermodel) at
`94d69019` (2026-11-08), cloned outside the repository, read only (notes 53, 54, 56, 103, 132; `tools/gen_seq_gen.py`;
`tools/machines/seq_gen/`; `tweaks/model-cycles_OS1.13/49-scale-gen*.json`; `docs/flasher/`).

## 0. In short

- **In this repository's words neither is a "package" as it stands.** A package (notes/59) is an added machine:
  a manifest plus a committed relocatable object. Sequence Genie is not a machine but a page on the sequencer, so it
  would be a **part** (`part.json`, like RESAMPLE and Transpose). The Syntakt engines are machines, but their code
  cannot be committed (hard rule 1): it is extracted from the user's `Syntakt_OS1.42.syx` at build time. The package
  format has no such kind yet.
- **Syntakt engines: the cheap way is closed.** The 252 `syntakt-*` tweaks are all still here and still build, but
  each one excludes 11 of the 14 cards offered today (§1.2). Putting `syntakt` back into `OFFERED` would give a
  flasher where ticking an engine unticks SAMPLE, every Track FX, both delays, the CPU page, RESAMPLE and Transpose.
- **Syntakt engines: the real way is a new package kind**, "machine from a recipe", in `compose.py`, `relocate.py`
  and `composer.js` (§1.3). It is the largest piece of builder work since B4, and it fills the id space (§1.4).
- **Sequence Genie: settle the licence first.** Supermodel has no licence of its own (§2.1). Its Genie sources carry
  no grant. Hard rule 7 and `CONTRIBUTING.md` forbid porting or closely paraphrasing them without AveyCole's written
  OK.
- **Sequence Genie: a port, not a copy.** Every Supermodel variant `requires` Model-TG (§2.2), which model27 does not
  offer. It reads Model-TG's Scale and Root, chains onto Model-TG's key hook, and sits in Supermodel's own sprite
  masks. In model27 it becomes a part with its own scale state and its own placement, chained next to RESAMPLE on the
  key-code getter (§2.3).

## 1. The Syntakt engines

### 1.1 What is here `[FAIT]`

- `tools/gen_syntakt_engines.py` (1 550 lines; notes/17, 20 to 30) generates one tweak per combination of the five
  engines SD VINTAGE, CP VINTAGE, SY TOY, SY BITS and SY SWARM, with and without Model-TG: 252 files
  `2x-syntakt-*.json`. The engines' code is extracted from the user's Syntakt file by a recipe (the `syntakt` part of
  `append`), relocated to `0x43000000`, and partly copied into SRAM at boot (notes/26, 28, 29), with the load
  governor (`tools/machines/gov/`, notes/25, 30, 36).
- The flasher card exists in `tools/gen_flasher_tweaks.py` (`"id": "syntakt"`, one box per engine, status `tested`),
  but `OFFERED` leaves it out. notes/59 scoped Level B as "no Model-TG, no Syntakt engines".
- The engines were tested on a Model:Cycles in Modded-Cycles (notes/20 to 22, 30/09/2026). They were never tested in
  a model27 build, since no model27 build has offered them.

### 1.2 What re-offering the tweaks would exclude `[FAIT]`

Measured on the committed tweaks, without firmware: spans of `syntakt-sd` and `syntakt-sd-cp-toy-bits-swarm`
(writes, plus the `append` at `0x401aa140`, 260 352 B and 269 056 B, destination `0x43000000`) against each offered
card's tweak:

| Card | Overlapping spans | Declared conflict |
|---|---|---|
| `sampler` | 115-116 | yes |
| `digidelay`, `pots`, `smpfilter`, `crush`, `ring`, `reso`, `eq`, `resample`, `transpose` | 4 each | yes |
| `cpu` | 5 | yes |
| `control-all-config`, `trig-cond-order`, `trig-preview` | 0 | no |

The four common spans are the payload and the registrar's places, which every composed build takes. The engines'
tweaks and model27's composed builds are two worlds that cannot be ticked together. Only the three plain mods
combine with them.

### 1.3 The way: a package kind "machine from a recipe"

What it needs, from what notes/59 to 63 built:

1. **The manifest.** As SAMPLE's (`tools/machines/sampler/manifest.json`), with the engine number, knobs, image and
   decay from `CATALOG` in `gen_syntakt_engines.py`. In place of `build.objects`, a `recipe`: the code ranges, the
   tables, the immediates that are addresses, the roots and the relocations, as `gen_syntakt_engines.py` computes
   them today. Only the recipe is committed, never bytes from the Syntakt OS (notes/17).
2. **The composer.** `compose.py` and `composer.js` extract each ticked engine from the Syntakt file and place it like
   a committed object. `relocate.py`'s loop over `R_68K_32` becomes a loop over the recipe's relocations. notes/61:
   `web_package` has no `syntakt` part today, and `builder.js` already has one for the old tweaks, a start.
3. **The bridge** (`tools/machines/syntakt_bridge/`): one per engine today, generated per combination. It becomes
   part of the package (committed object) or a part of the builder.
4. **The governor and the SRAM borrowing** (notes/26, 28, 29) become parts the composer adds when an engine is
   ticked. Their SRAM claims go into the claims (notes/65), against CHORD Tables (`chordtables`, SRAM pool), the
   Track FX and the effect slots. The CPU page and the governor both time the audio function at `0x40059382`. One of
   them must read the other, or they must share a dispatcher.
5. **The flasher.** The card asks for the Syntakt file (the old card's `needs: "syntakt"`), and `REF_MAINOS` holds
   combinations of engines with the other cards.
6. **Proofs.** `test_syntakt_machines.py` and the engine proofs adapted to the composed build (notes/60 §8 already
   lists them), as `test_compose.py` does for SAMPLE.

Done in steps, one engine first (SY TOY, the smallest, or SD VINTAGE, the first proven), alone and with SAMPLE. Then
the others, then the governor.

### 1.3a What the code here already gives `[FAIT]` (2026-11-09)

- **The relocator's format is not tied to ELF.**
  - `relocate.layout()` turns an object into a dict: `code`, `relocs` as `(position, base, addend)`, `symbols` and
    `bss`.
  - `relocate.apply()` places any such dict, and the registrar takes it from there.
  - A Syntakt engine therefore needs no new placement code. It needs a function that builds the same dict from the
    user's Syntakt file plus a committed recipe: the ranges, and each relocation as an offset into the extracted blob.
    In Python, and in `composer.js`.
- **The recipe exists.**
  - `gen_sdvintage_exact.py` computes the closure of an engine's functions and finds its addresses (`SEGMENTS`).
  - `gen_syntakt_engines.py` adds the catalogue of five engines (`CATALOG`: update, render, knobs, extra roots,
    immediates checked by hand).
  - The tweaks carry the result as `append.parts` (`{"dest", "syntakt": [from, to]}`) and `append.reloc`.
  - `builder.js` already extracts those parts from the Syntakt file in the browser and checks its hash
    (`append.syntakt.os`).
- **The bridge exists.**
  - `tools/machines/syntakt_bridge/bridge_engines.c` is our own code.
  - It takes each engine's update and render as `-DUPD_E` / `-DRND_E`, fixed addresses. The base of the SRAM
    replica is `ST_SRAM`, also a fixed address.
  - In a package these become patch points: undefined symbols that `apply()` resolves from `values`, as
    `cm_machine_index` is for SAMPLE.
- **The proofs exist.**
  - `test_sdvintage_exact.py` and `test_syntakt_machines.py` compare the engines with the Syntakt sample for sample.
  - They can run against the composed build as `test_compose.py` runs SAMPLE.

**The payload is mostly shared:**

| Tweak | Extracted from the Syntakt file | CHORD tables moved | Our code |
|---|---|---|---|
| `21-sdvintage-exact` (SD alone, SDRAM only; tested on the device, 30/09) | 157 804 B | 0 | 512 B |
| `24-syntakt-sd` | 189 314 B | 35 096 B | 2 997 B |
| `24-syntakt-sd-cp-toy-bits-swarm` | 202 428 B | 35 096 B | 5 225 B |

- About 158 KB is the base every engine needs: the audio program's tables and the image of its SRAM.
- Each further engine adds 0 to 7 KB.
- **So five self-contained packages would carry the base five times, which cannot fit.**
- The shape is one shared part `syntakt` (the base, with the user's file, added once when any engine is ticked) and
  five thin engine packages (manifest, the bridge's object, the engine's own ranges).

**The budget.**
- The payloads share `0x401aa140..0x40200000`, 351 936 B.
- The build with all fourteen cards stores its payload in 132 304 B. Composed here with `compose.py`, it gives MAIN
  OS `398b05d1…`, the build tested on the device (notes/80 §6.1). That leaves about 219 KB.
- The base with all five engines, in SDRAM only (no CHORD tables moved), is about 207 KB. It fits, about 12 KB under
  the limit.
- The move to SRAM (notes/28, 35 KB more) does not fit next to all fourteen cards. It also clashes with CHORD
  Tables' SRAM pool.
- Whether the base can shrink is not known yet: the large table `0x40028438..0x4003be38` (80 KB) may be needed by some
  engines only. That needs the closure computed per engine.

**Order:**
1. SD VINTAGE as a package, in SDRAM only, without the governor: notes/17's form, which played on the device. Proven
   identical to `21-sdvintage-exact` in emulation.
2. The split into the shared part and engine packages, with a second engine.
3. `composer.js`.
4. The governor as a part, with the CPU page: both time `0x40059382`.

### 1.4 The id space

notes/62 §4.6: added machines have stable ids 6 to 11, six in all. SAMPLE holds 6. SAMPLE and the five engines fill
the space; any machine after them needs the layout grown first (8 slots, 14 marks on the MACHINES page).

### 1.5 Which engines

model27's, as in `gen_syntakt_engines.py`. Supermodel has moved on (Swarm LITE, among others), but those changes are
Supermodel's own code, under the licence question of §2.1.

## 2. Sequence Genie

### 2.1 Licence `[FAIT]`

- Supermodel was forked from 18nelli18/Modded-Cycles before upstream's commit `d64c3f6` (2026-10-07, "Put
  Modded-Cycles under the MIT License"). The commit is not in its history, and Supermodel has no `LICENSE` file at
  its root.
- GitHub shows the repository as MIT because of `LICENSE-Punch.txt`, a third-party notice (copyright 100demons).
- Supermodel's README says its sources are "all MIT licensed", but that sentence is about the projects it credits.
- The Genie's files (`tools/machines/seq_gen/*.s`, `docs/flasher/genie-*.js`, `genie_router.s`) carry no licence
  header. Its PROVENANCE row calls the code "ce projet" / "assemblage original Codex", at AveyCole's request. The only
  licensed part is the page built after Model-TG (MIT, `70b39dd`), which model27 already relies on.
- **So:** under hard rule 7, model27 cannot port, transcribe or closely paraphrase the Genie. Two ways out:
  1. AveyCole adds a licence (MIT, like the rest of this family) to Supermodel;
  2. or AveyCole gives a written OK for model27 (`CONTRIBUTING.md`).
- A clean-room rewrite from the behaviour alone (the user-facing description, not the code) is a third way, but it is
  weaker. The notes describe the algorithm closely (the xorshift seed, the Euclid formula), and reading the sources
  first taints the room.

### 2.2 What the Genie is in Supermodel `[FAIT]`

- **The page:** SETTINGS + PAGE. It holds:
  - Scale and Root;
  - note, velocity, decay and pan ranges;
  - density;
  - Both / Rhythm / Notes regeneration;
  - mutation;
  - Euclid hits and rotation;
  - Generate and a one-level Undo;
  - PLAY/STOP and FUNC + pads (mutes) inside the page.
- **During playback,** generation is queued to the selected track's next loop or 16-step page (`queue.s`, notes/56
  and 103). The seed is mixed with the DMA timer on the first generation (notes/132).
- **SETTINGS + TRACK:** a random sound (notes/54 §2).
- **Size:** 6.6 to 9.5 KB in 104 to 138 writes, in about 51 freed sprite masks.
- **Every variant `requires` `model-tg` or `model-tg-st`:** the main ones, `-queued`, and `experimental/*`. The
  hook's `old` bytes are Model-TG's jump to `key_hook` (`0x4007240c`). Scale and Root are Model-TG's variables. In the
  main flasher, the timing and seed fixes are overlays on Supermodel's MIDI OUT Bank build (`docs/flasher/genie-*.js`),
  not a separate tweak.
- **Hardware:** tested by AveyCole (10/10, 11/10, 16/10 and 18/10/2026).

### 2.3 The way: a part `seqgen`

Once the licence allows it:

- **Shape.** `tools/machines/seqgen/part.json`, sources in C, or assembly where the original is assembly. Built by
  `tools/gen_parts.py`, added by `compose.py` when its card is ticked. Its own generator writes the tweak for the
  card, as `gen_transpose.py` does.
- **Scale state.** The part's own Scale and Root, in place of Model-TG's. The open question: kept in RAM like
  Supermodel's (lost at power-off), or saved, with a claim on a pattern byte (notes/65).
- **Keys.** The key-code getter `0x4007240c` is RESAMPLE's `code` hook today, written once. Two parts on it need a
  chain or a dispatcher (`tools/machines/dispatch/`, notes/63 §4).
- **Gestures.** SETTINGS + PAGE and SETTINGS + TRACK go through `claims.py` against `trig-preview` (held trig +
  PAGE), `transpose` (MACHINES + TRIG), the shortcut layer and RESAMPLE (SETTINGS + RECORD).
- **The page.** Built as the CPU page and RESAMPLE build theirs, after Model-TG.
- **Placement.** From model27's free masks (`REGISTRY.md`), not Supermodel's.
- **Writing the pattern.** Trigs, notes and locks go through the OS's notifications, as notes/53 §3 found.
- **Proofs.** `tools/emu/test_seqgen.py` on the OS's own sequencer, after Supermodel's `test_seq_gen*.py`. Those
  proofs are tests, so whether they may be ported is the same licence question.
- **Scope to decide.** Supermodel's whole Genie, or the core first: scale, ranges, density, Generate and Undo with the
  transport stopped, then queued generation, mutation and Euclid. The random sound (SETTINGS + TRACK) only knows the
  six stock machines' descriptors (notes/54 §2) and would need the added machines'.

## 3. Order of work proposed

1. Now, at no cost: run the emulator proofs here (on the branch `nicolai`). Nothing in §1 or §2 can be
   proven without them.
2. Ask AveyCole about the licence (§2.1). That takes time to answer, and nothing else waits on it.
3. The Syntakt engines as a package kind, one engine first (§1.3). It is entirely this repository's code and
   licence.
4. Sequence Genie as a part (§2.3), when §2.1 is settled.

## 3a. Roadmap (the maintainer, 2026-11-09)

0. **Emulator proofs and CI** (on the branch `nicolai`). Every step below is proven with them.
1. **Payload compression in the package builder.**
   - `build.py` and `builder.js` already compress a payload (`append.compress: "aplib"`, notes/50), and `stub.S` has
     the `APLIB` variant.
   - Still to do:
     - `compose.py` and `composer.js` emit it;
     - the registrar's boot stub (`gen_registrar.py`) gets the depacking path;
     - the claims and `REF_MAINOS` follow.
   - Compress only a build that would not fit otherwise, so that the builds already tested on the device stay
     byte-identical.
2. **ACID as a package** (Open303, MIT; `gen_acid.py`, notes/54). ACID has its own filter; its CPU cost goes next to
   the Track FX.
3. **MACRO as a package** (Braids, MIT; `gen_macro.py`, notes/43 and 55). Compiling it needs a clone of
   `pichenettes/eurorack`.
4. **More machine slots, 6 → 8** (notes/62 §4.6).
   - SAMPLE is already an optional card. What it keeps is its stable id 6, which saved projects refer to.
   - So the slots grow, rather than SAMPLE's id being given away.
   - Needed before the Syntakt engines: ACID, MACRO and five engines are seven machines, and five slots are free.
5. **The Syntakt engines as a package kind** (§1.3, §1.3a), with the governor as a part next to the CPU page.
6. Not ordered yet:
   - more filter types as Track FX (state-variable band-pass and notch, ladder…);
   - Sequence Genie (§2, licence first).

Each step ships `experimental`, and becomes `tested` only after the maintainer's test on their Model:Cycles.

## 4. Decisions for the maintainer

- Do the Syntakt engines take the last five ids, or is the id space grown first (§1.4)?
- Do the engines need the Syntakt file as a second upload in the flasher, as the old card did? There is no other way
  without committing Elektron's code.
- Genie: wait for a licence, or rewrite clean-room?
- Genie: Scale and Root saved with the pattern, or in RAM only?
