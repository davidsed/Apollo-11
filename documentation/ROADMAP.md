# Enhancement Roadmap

How to take a proposed change to the flight software in this fork from **idea** to **demonstrated in simulation**. It is written so that anyone, a person or a future Claude session, can pick it up cold.

The first enhancement to go through it is [Stop the rendezvous radar draining the CPU during descent](luminary099/descent/enhancements/rr-cpu-drain.md). The phases are generic and apply to any enhancement.

> **Status key:** ✅ done · 🟡 in progress · ⬜ not started · ❓ blocked on a decision or open question
>
> Commands below come from the Virtual AGC documentation and source (see [References](#references)). **None has been run on this machine yet.** Treat them as a starting point and record what actually worked in the [Log](#log).

## Where we are

| Phase | Goal | Status |
|---|---|---|
| [0. Toolchain](#phase-0-toolchain) | Build yaYUL, yaAGC and Tools on macOS | ⬜ |
| [1. Baseline rope](#phase-1-baseline-rope) | Prove the source assembles **byte-identical** to the 1969 rope | ⬜ ❓ fork drift |
| [2. Simulation harness](#phase-2-simulation-harness) | Run Luminary099 headless in a descent state, scriptably | ⬜ |
| [3. Reproduce the issue](#phase-3-reproduce-the-issue) | Observe the original problem (for example the 1202s) in the harness | ⬜ |
| [4. Implement](#phase-4-implement-the-enhancement) | Patch the source, assemble, check memory | ⬜ |
| [5. Verify](#phase-5-verify-ab) | A/B compare the original and patched ropes under the same scenario | ⬜ |
| [6. Document and merge](#phase-6-document-and-merge) | Update the docs and enhancement page, merge to `main` | ⬜ |

**Current enhancement:** [rr-cpu-drain](luminary099/descent/enhancements/rr-cpu-drain.md). Recommended order (revised after reading the later Luminary code):
1. **Option 1:** backport Apollo 12's PCR 848 (8 instructions; a known-good answer).
2. **Option 3:** new alarm 00516.
3. **Option 2:** modelled on Apollo 13's PCR 1013, as a stretch goal.

---

## How to resume (read this first)

1. Read the [Where we are](#where-we-are) table and the latest [Log](#log) entry.
2. Read the current enhancement page.
3. Follow the repo workflow in the root [README](../README.md#about-this-fork): branch from `main` as `feature/<name>`, then merge back to `main` with `--no-ff`.
4. Work on the first phase that isn't ✅. Every phase ends with **exit criteria**. Don't start the next phase until they're met, and record the outcome in the [Log](#log).
5. Keep the toolchain **outside this repo**. Workspace layout:
   ```
   ~/Documents/GitHub/davidsed/apollo-agc/
   ├── Apollo-11/     this fork: docs, our patches, harness code
   └── virtualagc/    Virtual AGC: toolchain plus every Luminary version (shallow clone)
   ```
   Commit only source changes, harness code (under `tools/`) and docs to `Apollo-11`.

---

## Phase 0: Toolchain

**Goal:** command-line tools working on macOS. The GUI tools are optional.

There is no Homebrew formula and no prebuilt macOS binary ([download page](https://www.ibiblio.org/apollo/download.html)), so build from source:

```bash
xcode-select --install                       # if needed
# Already cloned (shallow) at ~/Documents/GitHub/davidsed/apollo-agc/virtualagc. To refresh: git -C <that dir> pull
cd ~/Documents/GitHub/davidsed/apollo-agc/virtualagc

# Command-line only, no wxWidgets needed (target names from the top-level Makefile):
make MACOSX=yes FORCE_clang=yes yaYUL yaAGC Tools

# Full build including the GUI DSKY (needs wxWidgets 3.2 built from source on Sequoia+):
# make cc=clang CC=clang++ MACOSX=yes FORCE_clang=yes clean install
```

**Notes**
- The macOS system Tcl/Tk is broken, so LM_Simulator would need Tcl/Tk 9 built from source. We don't plan to use LM_Simulator: it has no trajectory or radar model.
- **ApoDisKey** is a native Swift DSKY for Sonoma and Sequoia, if a visible DSKY helps during debugging.
- Apple Silicon: yaYUL and yaAGC are plain C, so they should build for arm64. *Unverified.*

**Exit criteria**
- [ ] `yaYUL/yaYUL --help`, `yaAGC/yaAGC --help` and `Tools/oct2bin` all run.
- [ ] The build steps that actually worked are recorded in the Log.

---

## Phase 1: Baseline rope

**Goal:** know *exactly* which source we patch, and that it produces the real 1969 rope.

**Step 1: reference build (virtualagc's copy).** The mission Makefile assembles the source and compares it against an octal dump of the original rope:

```bash
cd ~/Documents/GitHub/davidsed/apollo-agc/virtualagc
make -C Luminary099
# Under the hood (Makefile.inc):
#   yaYUL --unpound-page MAIN.agc > MAIN.agc.lst          -> MAIN.agc.bin, MAIN.agc.symtab
#   Tools/oct2bin < Luminary099.binsource -> Luminary099.bin
#   diff -s MAIN.agc.bin Luminary099.bin                   expect: "Files ... are identical"
```

Compare against `Luminary099.binsource`, **not** `AP11ROPE.binsource`. Its check is commented out pending proofing.

**Step 2: this fork's copy.**

```bash
cd ~/Documents/GitHub/davidsed/apollo-agc/Apollo-11/Luminary099
../../virtualagc/yaYUL/yaYUL --unpound-page MAIN.agc > /tmp/fork-MAIN.lst
cmp MAIN.agc.bin ../../virtualagc/Luminary099/Luminary099.bin && echo IDENTICAL
rm -f MAIN.agc.bin MAIN.agc.symtab                       # build outputs; don't commit them
```

**❓ Known risk: fork drift.** This repo's `Luminary099/` differs from virtualagc's.
- Files have been renamed, for example `R60_62.agc` vs `R60,R62.agc`.
- Some labels are different: `1406P00` vs `1406POO`, `W.INDI` vs `W.IND1`, `NETTOTKP` vs `NEGTOTKP`.
- `STERN` is a comment here but a label there.
- There are extra `EQUALS` lines in `ERASABLE_ASSIGNMENTS.agc`.
- The git history shows community fixes: "Fix YaYUL Fatal Errors in Luminary099 (#665)" and "Common fixes (#691)".

These are probably just label changes, but **byte-identity isn't proven**.

**Decision 1: which source do we patch?** Record the choice in the Log.

| If step 2 gives… | Then |
|---|---|
| Identical binary | Patch this fork's `Luminary099/` directly. Simplest option, and the docs' line links stay valid. |
| Different binary | Use `Tools/ropediff.py` to see which words differ. If the differences matter, patch a copy of virtualagc's source instead, for example vendored at `Luminary099-vagc/`, and note in the docs which tree the line links refer to. |

**Exit criteria**
- [ ] The reference build reports identical.
- [ ] The fork build result is recorded, and Decision 1 is made.
- [ ] The baseline `.bin` and `.symtab` are kept outside the repo, or under an ignored `build/` folder.

---

## Phase 2: Simulation harness

**Goal:** a repeatable, scriptable way to run Luminary099 in a powered-descent state on macOS.

**Why we need a custom harness**

| Tool | Flies a descent? | Models the CPU cost of counter increments? | Platform |
|---|---|---|---|
| yaAGC + yaDSKY2 | No: computer only, no vehicle | **Yes.** An injected counter increment uses up an instruction cycle | macOS |
| LM_Simulator | No: IMU and switch panels only, "not a lunar-lander simulator" | n/a | macOS (needs Tcl/Tk 9) |
| NASSP (Orbiter) | **Yes.** It ships `Luminary099.bin` and an `Apollo 11 - 10 - PDI T-15m` scenario | **No.** CDU counts are written straight into memory, which costs no CPU time | Windows only |

**Recommended: a headless C harness around `agc_engine`** (lives in this repo under `tools/agc-harness/`).

1. Link virtualagc's `yaAGC/agc_engine.c`, `agc_engine_init.c` and `NullAPI.c`. `EmbeddedDemo.c` is a template.
2. Load the rope (`.bin`) and a **starting state**:
   - **Preferred:** convert NASSP's `Apollo 11 - 10 - PDI T-15m.scn` LM AGC block (`EMEMoooo value` lines) into a yaAGC core-resume file. That gives a real pre-descent erasable state without flying from launch.
   - **Fallback:** a fresh start, followed by setting the key erasables and flags. `set var SYMBOL=value` works in yaAGC's debugger via `--command=FILE`.
3. Drive the DSKY with keycodes on channel 015, for example V37E63E.
4. Provide **just enough vehicle**: accelerometer (PIPA) counts consistent with a descent burn, landing radar data good, and channel 30/31/33 switch bits. Begin with constant thrust. Full dynamics aren't needed to show an Executive overload.
5. Instrument it, using `MAIN.agc.symtab` for addresses:
   - break or watch on `BAILOUT1` and `ALARM`
   - `FAILREG` (alarm codes)
   - the eight `PRIORITY` slots and `VAC1USE`…`VAC5USE` (core-set and VAC use)
   - `MODREG` (current program)
   - a restart counter
   - an idle-time counter (time spent in `DUMMYJOB`) as a duty-cycle estimate
6. Output a run log, CSV or JSON, so A/B runs can be compared by script.

**Alternative: socket peripheral.** Run `yaAGC --port=19797 --interlace=1 …` and a Python peripheral that speaks the 4-byte socket protocol ([developer page](https://www.ibiblio.org/apollo/developer.html)). It's simpler to start, but timing-sensitive. See the rate limit in Phase 3.

**Optional high-fidelity check (needs a Windows machine).** Patch NASSP so rendezvous radar CDU counts go through a cycle-stealing increment, then fly the real PDI scenario with both ropes.

**Exit criteria**
- [ ] The harness runs the **unmodified** rope into P63 with SERVICER cycling every 2 s and no alarms. This is the "quiet baseline".
- [ ] One command produces a run log.

**Main risk:** without vehicle dynamics, guidance may diverge or raise unrelated alarms (for example landing radar alarms). Keep the run short, the first few minutes after ignition, and document any alarms that come from the harness itself.

---

## Phase 3: Reproduce the issue

**Goal:** observe the original problem with the original rope, so the fix can be measured against it.

**For rr-cpu-drain, inject the radar drain.**
- Counters: `CDUT EQUALS 35` (trunnion) and `CDUS EQUALS 36` (shaft), at [ERASABLE_ASSIGNMENTS.agc:120-121](../Luminary099/ERASABLE_ASSIGNMENTS.agc#L120-L121).
- Rate: up to 6,400 increments per second per counter, about 12,800/s in total. Grumman measured up to 13.36% of CPU time ([enhancement doc](luminary099/descent/enhancements/rr-cpu-drain.md#1-the-problem-in-one-paragraph)).
- **Headless harness:** call `UnprogrammedIncrement(State, 035 or 036, 021)` about every 6.7 machine cycles, and skip that cycle's instruction. That mirrors how yaAGC's `ChannelInput` handles an increment.
- **Socket path:** the packet has channel `0200 | 035` (or `036`) and data `021` (+1 count, fast). yaAGC polls sockets every `--interlace` cycles (default 50) and handles at most one increment per poll. The ceiling is roughly 85,333 ÷ (interlace + 1) per second, so 12,800/s needs `--interlace=5` or lower. *Derived from source, not tested.*

**Scenarios to run**
1. Quiet baseline (no injection), as in Phase 2.
2. Injection at 13% drain.
3. Injection plus a V16 N68 monitor display requested via the DSKY. This is Aldrin's case: the 1969 crew saw the alarms come up with N68 on screen.
4. Optional: sweep the drain from 5% to 15% to find the threshold where 1202s begin.

**Historical calibration.** In August 1969, Grumman and MIT reproduced the alarms by slewing the radar CDUs at about 6.4k pulses per second during a descent simulation, and they were "essentially reproduced" ([Tillman memo, 13 Aug 1969](https://ibiblio.org/apollo/Documents/Memo-Tillman690813_text.pdf)).

**Exit criteria**
- [ ] 1202 and/or 1201 alarms appear with the original rope under injection, and **not** in the quiet baseline.
- [ ] Time to first alarm, alarm count and restart count are recorded for each scenario.

---

## Phase 4: Implement the enhancement

**Goal:** a minimal, reviewable patch that assembles.

1. Branch from `main`: `feature/<enhancement>-<option>`, for example `feature/rr-drain-opt1-pcr848`.
   - **Where to start for rr-cpu-drain:** Option 1 is a straight backport. Replace `NORRGMON EQUALS DAPT4S` in `Luminary099/T4RUPT_PROGRAM.agc` with the 8 instructions from `Luminary116/T4RUPT_PROGRAM.agc:1278-1286`. The expected result in Phase 5 is "no 1202s with the RR switch out of LGC", which is what Apollo 12–17 achieved.
   - **Reference sources:** the later versions are in the sibling `virtualagc/` clone: `Luminary116/`, `LM131R1/`, `Luminary178/`, `Luminary210/`.
2. Mark every change with a `# FORK: <enhancement> <option>` comment so it can be found with `grep -rn "# FORK:"`.
3. Prefer changing few files, and reuse existing routines and patterns. The enhancement page lists the hook points for each option.
4. Assemble with yaYUL. The binary **will** now differ from the 1969 rope, which is expected. Use `Tools/ropediff.py` to confirm that only the intended banks changed.
5. **Memory budget:** fixed memory was nearly full in 1969. Use the yaYUL listing to record the free words in each bank you touch. If a bank overflows, move code into a bank with space and reach it via `BANKCALL` / `POSTJUMP`, following the existing patterns.
6. Don't touch the areas the enhancement page marks as out of scope.

**Exit criteria**
- [ ] Assembly has no fatal errors, and the ropediff shows changes only in the intended banks.
- [ ] Memory usage is recorded.
- [ ] The patched `.bin` and `.symtab` are saved alongside the baseline ones, outside the repo.

---

## Phase 5: Verify (A/B)

**Goal:** show the fix works, using the same scenario, starting state and injection schedule for both ropes.

| Measure | Original rope | Patched rope | Expected for the option |
|---|---|---|---|
| 1202 / 1201 count | | | Opt 1 or 2: zero |
| Time to first alarm | | | Opt 3: alarm 00516 appears *before* any overload |
| Software restarts | | | Opt 1 or 2: zero |
| Duty cycle (idle estimate) | | | Opt 1: drain gone |
| Skipped SERVICER passes | n/a | | Opt 2: some, logged |
| Navigation error vs quiet baseline | | | Opt 2: within tolerance (to define) |
| Abort path still works | | | Run P70/P71 and confirm the radar becomes usable again in ascent |

**Exit criteria**
- [ ] The table is filled in from run logs, and those logs are saved in the repo under `documentation/.../results/`, or summarized there.
- [ ] Every regression is explained or fixed.

---

## Phase 6: Document and merge

- Update the enhancement page: change its status, add a "Results" section, and record which option shipped.
- If routines changed, update the affected program and event-chain pages, including any line links that moved.
- Update the [Where we are](#where-we-are) table and the [Log](#log).
- Merge to `main` with `--no-ff`, using the workflow in the root README.

---

## Open questions

| # | Question | Blocks | Notes |
|---|---|---|---|
| Q1 | Does the fork's `Luminary099/` assemble byte-identical to the 1969 rope? | Phase 1 | Decision 1 depends on it |
| ~~Q2~~ ✅ | Does *holding* channel 12 bit 1 (ZERO RR CDU) stop RR counter pulses? | Option 1 | **Answered:** yes. Luminary 116 holds the bit "TO PREVENT MEMORY CYCLE SNATCHING BY MANIC RR CDUS" ([enhancement §4](luminary099/descent/enhancements/rr-cpu-drain.md#apollo-12-the-pcr-848-radar-fix)) |
| Q3 | Can the harness reach a stable P63 without vehicle dynamics? | Phase 2–3 | Constant-thrust PIPA feed first |
| Q4 | Can a NASSP scenario state be converted to a yaAGC resume file? | Phase 2 | `EMEMoooo` lines → core-dump format (`MakeCoreDump` in `agc_engine_init.c`) |
| ~~Q5~~ ✅ | Is the Luminary 116 / 131 source available in virtualagc for reference? | Options 1–2 | **Answered:** yes. PCR 848 is in `Luminary116/T4RUPT_PROGRAM.agc`; PCR 1013 is in `LM131R1` (the flown Apollo 13 build, not the first Luminary 131 build) and carried into 178/210 |

## Log

Newest first. One entry per working session: what was tried, what happened, and what was decided.

| Date | Phase | Entry |
|---|---|---|
| 2026-10-11 | Setup | Workspace reorganized: this repo moved to `~/Documents/GitHub/davidsed/apollo-agc/Apollo-11`, with a shallow clone of `virtualagc` alongside it. |
| 2026-10-11 | Research | Compared later Luminary versions. Found PCR 848 (Apollo 12 RR CDU zero hold, Luminary116 T4RUPT `NORRGMON`), PCR 1013 (Apollo 13 P66 pass omission and alarm 1466, LM131R1), Apollo 14 `VACRLEAS` and SERVDURN/DUMLOOPS telemetry, and Apollo 15–17 `CHANBKUP`. Q2 and Q5 answered. Recommendation reordered: PCR 848 backport first. |
| 2026-10-11 | Planning | Roadmap written. Research found: no Homebrew or prebuilt tools; the fork's source has drifted from virtualagc (Q1); yaAGC models cycle stealing but NASSP doesn't; Virtual AGC alone can't fly a descent, hence a custom headless harness. Nothing built or run yet. |

## References

- Virtual AGC: download and build: https://www.ibiblio.org/apollo/download.html
- yaYUL: https://www.ibiblio.org/apollo/yaYUL.html
- yaAGC: https://www.ibiblio.org/apollo/yaAGC.html
- Socket protocol and port numbers: https://www.ibiblio.org/apollo/developer.html
- yaDSKY: https://www.ibiblio.org/apollo/yaDSKY.html
- LM_Simulator (scope and limits): https://ibiblio.org/apollo/yaTelemetry.html
- virtualagc source: https://github.com/virtualagc/virtualagc (`Makefile`, `Makefile.inc`, `Luminary099/`, `yaAGC/`, `Tools/ropediff.py`)
- NASSP (Project Apollo for Orbiter): https://github.com/orbiternassp/NASSP (branch `Orbiter2016`)
- Tillman memos on reproducing the Apollo 11 alarms, Aug 1969: https://ibiblio.org/apollo/Documents/Memo-Tillman690813_text.pdf, https://ibiblio.org/apollo/Documents/Memo-Tillman690809_text.pdf
