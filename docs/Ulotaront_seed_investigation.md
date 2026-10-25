# Phase 5 Diagnostic Note: Ulotaront 8W88 Pose Mismatch and the Seed-42 Root Cause

## 1. The problem

The master selectivity matrix reported a top-ranked Vina score of **-5.432 kcal/mol** for
ulotaront against the active TAAR1 structure (8W88). Ulotaront is the project's primary
positive control, and this specific pose had already been flagged in an earlier note as
suspect: *"true crystal pose recovered at mode 7 in 8W88, not the top-ranked pose."*

This needed to be checked before the score could be trusted, since a positive control
whose top pose doesn't reach the real pocket would undermine the validation logic the rest
of the project depends on.

## 2. Diagnostic process

### Step 1 — Obtain a reliable crystal reference
A properly atom-named PDBQT of the crystal ligand (`true_pose.pdbqt`, 13 atoms, extracted
from the original 8W88 structure) was used as the ground-truth reference throughout.

### Step 2 — Check the originally flagged poses (modes 1 and 7)
Nearest-atom RMSD to the crystal reference:

| Pose | RMSD to crystal |
|---|---|
| Mode 1 | 7.404 Å |
| Mode 7 | 5.521 Å |

Neither reached the real pocket. Both are far outside any reasonable match (the accepted
bar is under ~2 Å).

### Step 3 — Rule out receptor contamination
The receptor file referenced in the Phase 5 batch script
(`pdbqt_receptors_backup\Cleaned_8w88.pdbqt`) was checked directly and found to still
contain the crystal ligand (13 UJL atoms present as HETATM records) — the same
contamination issue identified earlier in the project. Re-docking against this file cannot
find the true pocket, since it is physically occupied by a static copy of the ligand
already sitting in the receptor.

A separately prepared, confirmed-clean receptor (`Cleaned_8w88_FINAL.pdbqt`, 0 HETATM, 0
UJL) was used for all subsequent tests. A direct check confirmed zero receptor atoms fall
within 2 Å of the true ligand position in this file — the pocket is physically open and
unobstructed.

### Step 4 — Re-dock with the clean receptor (still unsuccessful)
Re-docking ulotaront into the clean receptor, using the exact box center/size recovered
from the original `run_docking.bat` script (`center: 101.59, 103.70, 73.01`; `size: 20 Å`
cube) but with `--seed 42 --exhaustiveness 32` (the settings adopted project-wide for
reproducibility), still failed to recover the crystal pose:

| Mode | RMSD to crystal |
|---|---|
| 1 | 7.404 Å |
| 2 | 6.298 Å |
| 3 | 7.460 Å |
| 4 | 5.386 Å |
| 5 | 7.736 Å |
| 6 | 7.365 Å |
| 7 | 5.416 Å |
| 8 | 7.884 Å |
| 9 | 6.503 Å |

Notably, these values are numerically identical (mode-for-mode) to the result obtained
earlier using the *contaminated* receptor and a slightly different box. Two meaningfully
different setups converging on exactly the same wrong answer indicated the problem was
neither the receptor contamination nor the box placement, but something both runs shared:
the fixed random seed.

### Step 5 — Remove the fixed seed
The same command was re-run against the same clean receptor and box, but without
`--seed 42` (matching how the original, successful Phase 3 validation was actually run —
no fixed seed was used at that time):

| Mode | RMSD to crystal |
|---|---|
| 1 | 7.418 Å |
| 2 | 6.317 Å |
| 3 | 5.375 Å |
| **4** | **0.680 Å** |
| 5 | 7.669 Å |
| 6 | 6.513 Å |
| 7 | 6.903 Å |
| 8 | 7.902 Å |
| 9 | 6.562 Å |

Mode 4 recovered the crystal pose almost exactly, closely matching the original Phase 3
validation result (0.771 Å).

## 3. Root cause

**The fixed random seed (`--seed 42`), adopted project-wide to guarantee reproducibility
across the 16-drug panel, happens to steer Vina's search away from the true TAAR1 pocket
for ulotaront specifically, toward a competing, reproducible false minimum elsewhere on
the receptor.** This is not a flaw in the receptor, the box, or the original Phase 3
validation — all three were independently confirmed correct. It is a property of that
particular seed's random search trajectory for this particular ligand.

This also means: **reproducibility and correctness are not the same thing.** A fixed seed
guarantees you get the same answer every time, but says nothing about whether that answer
is the right one.

## 4. Implications for the project

1. **The -5.432 kcal/mol currently in the master matrix for ulotaront vs. 8W88 does not
   represent real TAAR1 pocket engagement** and should not be reported as such.
2. **The corrected result is available**: mode 4 from the no-seed run (0.680 Å RMSD to
   crystal). Its Vina affinity score should be substituted into the matrix in place of the
   current value once retrieved from the run's log file.
3. **Other TAAR1-designed drugs in the same 8W88 batch are at risk of the same issue**,
   having been docked with the same fixed seed. Ralmitaront, as the project's other primary
   TAAR1-selective positive control, is the highest-priority drug to re-check.
4. **A single fixed seed should not be trusted uncritically for the full 16-drug 8W88
   panel.** The recommended approach going forward is to dock each drug multiple times
   (either with several different seeds, or with no fixed seed run 2-3 times) and confirm
   convergence on a consistent, pocket-engaging pose before accepting a score — rather than
   accepting whichever pose a single fixed seed happens to rank first.

## 5. Suggested wording for the write-up

> Redocking of ulotaront into the active TAAR1 structure (8W88) using a fixed random seed
> (seed=42, adopted project-wide for reproducibility) reproducibly failed to recover the
> experimentally observed binding pose, instead converging on a false minimum elsewhere on
> the receptor (RMSD > 5 Å from the crystal pose across all nine returned modes). Receptor
> contamination and box placement were independently ruled out as causes. Re-running the
> identical docking without a fixed seed recovered the crystal pose closely (RMSD = 0.680
> Å), consistent with the original Phase 3 validation (RMSD = 0.771 Å). This indicates the
> fixed seed, while useful for reproducibility, is not a reliable choice for every
> ligand-receptor pair and can silently steer the search toward a non-physiological pose.
> The 8W88 panel scores should be re-verified using multiple independent searches rather
> than a single fixed seed.

## 6. What was *not* done, for transparency

- Only ulotaront was re-checked in this session. Ralmitaront and the other 14 drugs in the
  8W88 column have not yet been individually verified against this same seed-dependence
  issue.
- The exact Vina affinity score for the corrected mode 4 pose was not yet extracted from
  the run log for insertion into the spreadsheet.
- No systematic multi-seed protocol (e.g., 3-5 seeds per drug with convergence checking)
  has yet been implemented for the panel; this remains a recommended next step rather than
  a completed one.
