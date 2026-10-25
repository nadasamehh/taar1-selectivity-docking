# Phase 5 Diagnostic Note: Anomalous Positive Docking Scores Against Apo-TAAR1 (8ZSJ)

## 1. The problem

While extending the docking panel from the TAAR1 homology model into the experimental
structures (Phase 5), three drugs returned strongly **positive** (repulsive) Vina affinity
scores against the apo-state TAAR1 structure (PDB 8ZSJ):

| Drug | Score vs 8ZSJ (kcal/mol) |
|---|---|
| Clozapine | +5.066 |
| Quetiapine | +4.808 |
| Olanzapine | +3.353 |

A positive Vina score is unusual. Vina almost always finds *some* pose with a negative
(favorable) score, even for a poor fit, so a result this positive is a signal that
something in the setup or the underlying biology is unusual enough to investigate before
the number is accepted into the selectivity matrix.

All three drugs are structurally related — bulky, flexible, tricyclic/piperazine-type
antipsychotics — which made a shared, mechanistic explanation plausible rather than three
unrelated coincidences.

## 2. What we needed to rule in or out

A favorable-looking pose with an unfavorable score can come from several distinct causes,
each requiring a different fix:

1. **A file-handling bug** — wrong file loaded, stale ligand contamination, or a
   receptor/ligand mismatch.
2. **A receptor-preparation error** — missing hydrogens, wrong protonation states, or a
   corrupted PDBQT causing spurious clashes.
3. **A genuine electrostatic clash** — e.g. two protonated/charged groups (ligand amine vs.
   a charged histidine) forced close together.
4. **A genuine steric clash** — the ligand physically too large for a collapsed/closed
   pocket in this particular receptor conformation.
5. **Internal ligand strain** — Vina twisting the molecule into an unfavorable internal
   conformation to fit, without necessarily clashing with the protein.

Only (4), and to a lesser extent (5), would represent a real biological finding worth
reporting. (1)–(3) would mean the number should be discarded, corrected, or re-run before
it goes into the spreadsheet.

## 3. Diagnostic process (clozapine, used as the representative case)

### Step 1 — Confirm the correct files were being compared
Before analysis, the receptor and ligand objects loaded in PyMOL were confirmed by name
(`Cleaned_8ZSJ`, `Clozapine_8zsj_FINAL`) and the top-ranked Vina pose (mode 1) was isolated
into its own single-state object (`clozapine_mode1`) to make sure all subsequent
measurements referred to the actual docked, best-scoring pose rather than an input file or
a different mode.

### Step 2 — Rule out gross atomic overlap
A strict 1.5 Å all-atom clash check returned **zero atoms**, which on its own can be
misleading — it rules out literal overlapping coordinates but does not rule out a real
steric or electrostatic problem, since Vina's repulsive term rises steeply well before
1.5 Å.

### Step 3 — Visual inspection
Surface rendering of the receptor with the docked ligand shown as sticks confirmed the
ligand was seated inside a visible pocket-like dent on the protein surface, not floating
in space or obviously buried through solid protein. This ruled out the simplest
file-loading error (cause 1) and pointed toward the pose being geometrically plausible at
a glance, which made the positive score more puzzling rather than less.

### Step 4 — Identify what pocket the ligand actually occupied
A residue-neighbourhood scan (`within 4 Å` of the ligand) found:

```
HIS 87, HIS 175, ARG 179, GLY 180, VAL 184, PHE 185, PHE 186,
PHE 267, THR 271, ASP 274, PRO 275, TYR 279, PRO 283, ASN 286,
ASP 287, ILE 290
```

Two of the five canonical TAAR1 pocket residues (Phe186, Phe267) were present, confirming
the ligand was in the real orthosteric pocket region, not an unrelated cavity elsewhere on
the receptor (which would have pointed to a box-placement error).

### Step 5 — Check the salt-bridge distance
The ligand's closest amine nitrogen to Asp103 (the residue responsible for the TAAR1 salt
bridge) measured **8.96 Å** — far outside the ~4 Å required for a salt bridge. This showed
the ligand was seated at the shallow/entrance region of the pocket rather than reaching the
deep orthosteric site, consistent with a pocket that is more closed in this apo
conformation than in the ligand-bound active state (8W88).

### Step 6 — Test the electrostatic-clash hypothesis
The shallow position placed the ligand's amine close (3.71 Å) to His175, raising the
possibility of a charge-charge clash (cause 3) if that histidine were in its protonated,
positively charged form. This required checking the receptor's actual PDBQT file (the file
Vina reads, as opposed to the plain PDB used for visualization) for His175's atom types and
partial charges.

Once the correct prepared receptor PDBQT was located, the relevant lines showed:

```
ND1  charge -0.242  type NA   (no attached hydrogen — acceptor)
NE2  charge -0.350  type N,   with explicit HE2 hydrogen (+0.167)
```

Only one ring nitrogen carried a hydrogen — this is the neutral, singly-protonated
(Nε2-H) tautomer of histidine, not the doubly-protonated, positively charged form. **This
ruled out the electrostatic-clash hypothesis**: a neutral histidine does not produce the
kind of large repulsive penalty a charged one would.

### Step 7 — Full non-bonded contact scan
With causes (1)–(3) addressed, a complete scan of every ligand-atom-to-protein-atom
distance within 5 Å was run, rather than relying on the single 1.5 Å clash cutoff or a
single residue's distance. This returned:

- **Closest overall contact: 1.85 Å**, between a ligand atom and the backbone carbonyl
  oxygen of Val184.
- Over twenty additional contacts under 3.2 Å, including close contacts with Asp287,
  Phe267, Ile290, Thr271, and the chlorine substituent against Gly180 and Pro283.

A carbon/nitrogen-to-oxygen distance of 1.85 Å is far inside the normal non-bonded contact
range (~3.0–3.4 Å) and represents genuine, severe steric overlap. The presence of many
simultaneous sub-3.2 Å contacts, rather than one isolated close approach, indicated the
ligand was being forced into a space too small for it — not an isolated coincidental
clash.

## 4. Conclusion

The positive Vina score for clozapine against apo-TAAR1 (8ZSJ) is explained by **genuine
steric incompatibility** between the ligand and the closed apo pocket, most severely at
Val184, with multiple corroborating tight contacts elsewhere in the pocket. It is not
attributable to:

- a file-handling or ligand-contamination error (ruled out in Steps 1–4),
- a box-placement error (ruled out in Step 4 — the ligand was in the correct pocket
  region),
- a receptor protonation/preparation error at His175 (ruled out in Step 6).

This supports treating the result as a real finding: **the apo-state TAAR1 pocket is too
collapsed to accommodate this class of bulky, flexible drug**, in contrast to the
ligand-bound active state (8W88). Quetiapine and olanzapine were not individually put
through this same full diagnostic, but given their close structural similarity to
clozapine (bulky tricyclic/piperazine antipsychotics), the same mechanism is a reasonable,
though unconfirmed, inference for their positive scores as well.

## 5. Suggested wording for the write-up

> Clozapine, quetiapine, and olanzapine — structurally similar, bulky tricyclic/piperazine
> antipsychotics — scored positively (repulsive) against the apo-TAAR1 structure (8ZSJ).
> Pose inspection of clozapine, used as a representative case, ruled out file-handling,
> box-placement, and receptor-protonation errors as explanations, and instead found severe
> steric overlap with pocket-lining residues (notably a 1.85 Å ligand contact with the
> Val184 backbone carbonyl, alongside more than twenty additional sub-3.2 Å contacts). This
> indicates the unfavorable score reflects genuine steric incompatibility between this
> ligand class and the more closed apo-state pocket, rather than a modeling artifact. The
> same mechanism is inferred, but not individually confirmed, for quetiapine and
> olanzapine given their structural similarity to clozapine.

## 6. What was *not* done, for transparency

- Quetiapine and olanzapine were not put through the full seven-step diagnostic
  individually; the conclusion for them is an inference from structural similarity, not a
  directly confirmed result.
- Internal ligand strain (cause 5) was not separately quantified, since the steric-clash
  evidence found in Step 7 was sufficient on its own to explain the score.
- No re-docking or receptor modification was performed as a result of this investigation;
  the existing score is being retained and reported with this explanation rather than
  discarded or recomputed.
