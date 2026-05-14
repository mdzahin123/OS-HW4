# HW4 — CMPSC 472 (Fall 2025)

**Author:** Muhammad Danish Zahin Bin Rafizal
**Email:** mjr7066@psu.edu
**Course:** CMPSC 472 — Operating Systems

## Overview

This assignment is a concurrent-programming simulation of the **Snow White** story, written in BACI's concurrent C dialect (`.cm`). Seven processes (the Queen, the Huntsman, Snow White, the Old Peddler, and three Dwarfs) run in parallel inside a `cobegin` block, and the correct ordering of story events is enforced entirely through semaphores.

The point of the exercise is to use synchronization primitives (`wait` / `signal`, mutexes, barriers, and a small FIFO queue) to coordinate independent processes so that the narrative still unfolds in a sensible order despite the concurrency.

## Files

| File | Description |
|---|---|
| `mrafizal-hw4.cm` | The BACI concurrent C source for the simulation |
| `ReadMe.txt` | Original short note from the author |
| `README.md` | This file |

## Story Flow Enforced by Semaphores

The simulation walks through the following narrative beats, in order:

1. The Queen orders the Huntsman to kill Snow White.
2. The Huntsman takes Snow White to the forest; she follows.
3. The Huntsman lets her escape and brings a stag's heart back to the Queen.
4. Snow White finds the dwarfs' house and falls asleep.
5. All three dwarfs arrive home and discover her — a **barrier** ensures all three are present before the last one wakes her.
6. The dwarfs let her stay **in strict order D1 → D2 → D3** (enforced by chained semaphores).
7. After all three have agreed, the dwarfs wash up.
8. Snow White prepares lunch for each dwarf in the order they finished washing — handled with a small **FIFO queue** (`lunch_queue`, `queue_head`, `queue_tail`) guarded by a mutex.
9. All three dwarfs eat and head back to work.
10. The Queen, realizing the trick, disguises herself as the Old Peddler and knocks on the door.
11. Snow White opens the door, accepts the poisoned apple, and falls into a coma.
12. The dwarfs race home — a **mutex-protected check** picks the first dwarf to arrive, who finds her dead and calls the others.
13. Once all three have arrived (counted via another mutex), each puts Snow White in the casket.

## Synchronization Techniques Used

- **Binary mutex semaphores** — `print_mux` for atomic `cout` output, plus `barrier_mux`, `home_mux`, `arrival_mux`, and `lunch_queue_mux` for shared-variable protection.
- **Ordering semaphores** — chained `let_stay1/2/3` and `d1/d2/d3_can_let_stay` enforce the D1 → D2 → D3 sequence.
- **Barrier** — `dwarf_count` plus `barrier_wait` makes all three dwarfs converge before the last one wakes Snow White.
- **FIFO queue** — `lunch_queue[3]` with head/tail indices serves dwarfs lunch in arrival order.
- **Counting via multi-signal** — semaphores like `sw_in_house` and `can_come_home` are signalled three times to release all three dwarfs.
- **Race condition by design** — random `Delay()` calls before the "come home" check make it nondeterministic which dwarf arrives first.

## Building and Running (Local — Windows)

This program is written for **BACI** (Ben-Ari Concurrency Interpreter), which compiles `.cm` source to a bytecode `.pcd` file and then runs it on the BACI interpreter.

1. **Download BACI for Windows.** Get it from the course materials on Canvas, or from Bill Bynum's BACI page at William & Mary (the original distributor). The distribution contains `bacc.exe`, `bainterp.exe`, and usually a GUI wrapper `baci.exe`.

2. **Extract it somewhere stable**, e.g. `C:\BACI\`. After extraction you should have:
   ```
   C:\BACI\bacc.exe
   C:\BACI\bainterp.exe
   C:\BACI\baci.exe        (optional GUI)
   ```

3. **Run it without touching PATH.** In PowerShell, `cd` to the homework folder and call the executables by full path:
   ```powershell
   cd "C:\Users\danis\Downloads\HW4 Cmpsc 472\HW4 Cmpsc 472"
   C:\BACI\bacc.exe mrafizal-hw4.cm
   C:\BACI\bainterp.exe mrafizal-hw4
   ```
   - `bacc` compiles the source and produces `mrafizal-hw4.pcd`.
   - `bainterp` runs that bytecode — pass the filename **without** the `.pcd` extension.

4. **Optional — add BACI to your PATH** so you can just type `bacc` and `bainterp`:
   - Press **Win**, search "Environment Variables", open *Edit the system environment variables*.
   - Click **Environment Variables…** → under *User variables*, select `Path` → **Edit…** → **New** → paste `C:\BACI` → OK out.
   - **Close and reopen PowerShell.** Then:
     ```powershell
     cd "C:\Users\danis\Downloads\HW4 Cmpsc 472\HW4 Cmpsc 472"
     bacc mrafizal-hw4.cm
     bainterp mrafizal-hw4
     ```

5. **Or just use the GUI.** Double-click `baci.exe`, open `mrafizal-hw4.cm`, click *Compile*, then *Run*. No PowerShell needed.

Run the program a few times — the random `Delay()` calls produce different interleavings, so which dwarf finds Snow White dead first will change between runs.

### Expected output

A successful run starts with the story intro, interleaves the seven processes' actions, and ends with `To be continued.`:

```
Once upon time, lived  lovely little princess named SW .
Q orders H: kill SW
D1 goes to work
D2 goes to work
D3 goes to work
H takes SW to forest
SW follows H
H lets SW run
SW escapes, finds house, sleeps
...
D1 puts SW in casket
D2 puts SW in casket
D3 puts SW in casket
To be continued.
```

The exact order of interleaved lines (which dwarf goes to work first, who finds Snow White dead, etc.) will differ on every run — that's the point.

## Notes from the Author

> The string is really close to the maximal capacity of the BACI program, so if it's full, try deleting some of the strings in the print statements. I tried many times and it looks OK — afraid the code might suddenly be full on a different machine even though I ran it fully on the remote Sun Lab.

In other words: BACI has a tight limit on string-literal storage, and the `cout` messages have been kept short on purpose (e.g. `"D1 eats, go work"` rather than a full sentence). If you modify the code and the compiler complains about exceeding capacity, shortening some output strings is the fix.

## Tunable Parameters

- `DELAY` (set to `50` in `main`) controls the upper bound on the random delay between actions. Increasing it spreads events out further in time and makes race conditions easier to observe; decreasing it tightens the simulation.
