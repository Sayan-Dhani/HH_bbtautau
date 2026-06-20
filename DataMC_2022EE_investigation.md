Likely root cause of your 2022EE Data/MC disagreement: the b-tag shape SF
This is the one finding that matches "everywhere + overall normalization." Tracing the b-tag weight end-to-end:

Config uses shape, not shape_and_norm (config/global.yaml:118-123). The particleNet b-tag shape/reshaping SF deliberately does not preserve inclusive normalization — it must be renormalized per jet-multiplicity bin. That renormalization is exactly what btagShapeWeightCorrector.UpdateBtagWeight does, and it only runs in shape_and_norm mode (Corrections.py:619-637).
Worse, the central b-tag weight never reaches final_weight. GetWeight (which builds the plotted weight in histTupleDef.py:113) does not include weight_bTagShape_Central. The per-category applier GetBTagWeightDict is commented out inside a docstring at HistMergerFromHists.py:254-261, and GetBTagWeight is never called.
The producer that would generate the renorm JSON, BtagShapeProducer.py, even hardcodes import Analysis.hh_bbww — the wrong analysis.
So weight_bTagShape_Central is computed and stored but not applied to your central MC. That mis-normalizes MC (worst in b-tag categories, but it shifts everything) — consistent with your symptom. Fixing this needs an analysis decision + a renorm file, so I did not change it unilaterally. The path: switch to shape_and_norm, fix BtagShapeProducer's import, produce the per-njet renorm JSON for 2022EE, init btag_norm, and fold the central b-tag weight into final_weight.

What I verified is sound (ruled out)
Normalization core: the sum-of-weights denominator at anaTupleProducer.py:222 uses the same sign-only genWeight + PU as the numerator — internally consistent.
Luminosity 2022EE = 26674.92 pb⁻¹ — correct for postEE.
DY reweighting (dy_hhbbtautau) — correctly wired, era→2022postEE.
QCD ABCD — data is correctly skipped (config group is Data capital, processes.yaml:233).
Trigger SFs — are applied (via ID_weights_dict, not the commented old scheme).
Fixed (the two confirmed bugs you authorized)
Corrections/tau.h:35-49 — enum collision stat2_dm0==stat1_dm1==7, renumbered the stat block contiguously (stat1_dm1=8 … total=20).
Analysis/GetCrossWeights.py:645 — fixed the copy-paste so the second branch tests tau2_isMatched_mutau_tauLeg.
Both affect only systematic variations, so they won't move your central plots — but they were genuinely wrong.

Caution on issues.md
That report is auto-generated and has several false positives I debunked above (the &&, "trigger SFs dropped", muMu/eMu condition). Don't chase those. I saved memories capturing the b-tag root cause and the false-positive list so we don't re-derive this.

Want me to draft the shape_and_norm switch (config + BtagShapeProducer import fix + folding the central b-tag weight into final_weight), so you just need to produce the renorm JSON? Or first confirm the b-tag hypothesis by re-plotting one b-tag category with weight_bTagShape_Central manually folded in?


# Run3_2022EE Data/MC Disagreement — Investigation Report

**Date:** 2026-06-14
**Scope:** Full workflow for `ERA = Run3_2022EE`, Data/MC disagreement seen across several plots
(specific channels, DY-dominated regions, QCD/fake regions, and overall normalization).
**Method:** Source-level verification of the actual code paths (not trusting the pre-existing
`issues.md` auto-audit, which contains several false positives — see §5).

---

## 0. Executive summary

| # | Finding | Severity | Affects central plots? | Status |
|---|---------|----------|------------------------|--------|
| 1 | **b-tag shape SF computed but never applied to central MC** | 🔴 Critical | **Yes — primary suspect** | Needs decision + renorm file |
| 2 | b-tag `shape` mode instead of `shape_and_norm` (no per-njet renormalization) | 🔴 Critical | Yes | Needs decision + renorm file |
| 3 | `BtagShapeProducer` hardcodes `import Analysis.hh_bbww` (wrong analysis) | 🟠 High | Blocks the fix for #1/#2 | Open |
| 4 | `tau.h` `UncSource` enum value collision (`stat2_dm0 == stat1_dm1 == 7`) | 🟠 High | No (systematics only) | **Fixed** |
| 5 | `mutau_tau_decayMode` copy-paste (`tau1` used where `tau2` intended) | 🟠 High | No (systematics only) | **Fixed** |
| 6 | ggH→ττ: use **only one** of `GluGluHto2Tau_M125` / `..._UncorrelatedDecay_UnFiltered` (double-count risk) | 🟡 Medium | Small | Currently single (UnFiltered) — OK; see §2 |
| 7 | **Inclusive disagreement is n_bJets-dependent & process-split** (DY n_btag-shape + tt flat ~0.8); lumi demoted | 🔴 Critical | Yes | **Open — see §7, esp. §7.3** |

**Conclusion (updated 2026-06-15):** the b-tag shape SF (findings #1–#3) is a confirmed, real
problem and explains a large part of the disagreement in **b-tag categories** — but it is **not the
whole story**: disagreement is also seen in the **inclusive / pre-b-tag region**, where the b-tag
weight is never applied (`GetBTagWeight` only returns it for resolved, non-`baseline` categories).
So there is a **second cause acting on the inclusive region** (§7). The `n_bJets` breakdown (§7.3)
shows this is **b-tag-multiplicity dependent and process-split**, *not* a flat offset: a strong
n_btag-shape rise in **DY** channels (⇒ the missing b-tag SF §1 and/or the n_btag-binned DY
reweighting) plus a flat ~20% **tt** normalization. The earlier "flat in η/M_vis ⇒ luminosity" read
was a projection artifact (those variables are dominated by the n_bJets=2 bin) and is **superseded**:
luminosity is demoted to a sanity check. The weight/normalization *machinery* (denominator, lumi,
cross-section, PU, DY reweighting, DY stitching, QCD ABCD, trigger/ID SFs, lumi-mask, MET filters)
was verified **internally consistent** (§4, §7). Two genuinely-wrong *systematics-only* bugs were
fixed (§3). **Overall priority: fix the b-tag SF (§1) first — it drives both the b-tag categories
and, via `n_bJets`, the inclusive region.**

---

## 1. Primary suspect — the b-tag shape scale factor

### 1.1 What the b-tag "shape" SF is

The particleNet b-tag **shape** (a.k.a. *reshaping* / *iterativeFit*) correction is applied as a
per-jet multiplicative weight that reshapes the b-tagger discriminant distribution in MC to match
data. By construction (CMS BTV-POG), **this weight does not preserve the inclusive MC
normalization** — applying it changes the total MC yield, and the change depends on jet
multiplicity and process. Therefore it must be **renormalized per jet-multiplicity bin** so that
the integral of each process before b-tagging is preserved.

### 1.2 Issue A — `shape` mode skips the renormalization

`config/global.yaml:118-123`
```yaml
btag:
  stages: [ AnaTuple ]
  modes:
    AnaTuple: shape          # <-- not "shape_and_norm"
  tagger: particleNet
  jetCollection: centralJet
```

In `Corrections/Corrections.py:616-642`, the mode decides whether the renormalization runs:
```python
if btag_sf_mode == "shape":
    df, bTagSF_branches = self.btag.getBTagShapeSF(...)          # reshaping only
elif btag_sf_mode == "shape_and_norm":
    df, bTagSF_branches = self.btag.getBTagShapeSF(...)
    df = self.btag_norm.UpdateBtagWeight(...)                     # <-- per-bin renorm
```

`btagShapeWeightCorrector.UpdateBtagWeight` (`Corrections/btag.py:286-359`) loads per-bin
correction factors from a JSON and rescales `weight_bTagShape_Central` (and the relative
systematic branches) so that the inclusive yield per njet bin is restored. **In `shape` mode this
step is skipped**, so the reshaping shifts the total MC normalization in a process- and
njet-dependent way → broad Data/MC normalization disagreement, worst in b-tag-enriched categories.

The denominator (sum-of-weights) used for normalization does **not** include the b-tag weight
(`FLAF/AnaProd/anaTupleProducer.py:201`, `weights_to_apply = [gen_weight_name] (+ pu)`), which is
correct — but it also means nothing else compensates for the b-tag yield shift. Only
`shape_and_norm` would.

### 1.3 Issue B — the central b-tag weight never reaches `final_weight`

Even setting the renormalization aside, the central b-tag weight is **not multiplied into the event
weight** used to fill histograms:

- The plotted weight is built by `GetWeight` (`Analysis/hh_bbtautau.py:200-286`) and assigned to
  `final_weight` in `Analysis/histTupleDef.py:111-131`. `GetWeight` multiplies `weight_base`, the
  per-channel ID/trigger SFs, and (for DY) `weight_dy_central` — **but not**
  `weight_bTagShape_Central`.
- The function that *would* apply a per-category b-tag weight, `GetBTagWeight`
  (`Analysis/hh_bbtautau.py:186-197`, which returns `weight_bTagShape_Central` for resolved
  categories), is **never called**. Its only intended caller, `GetBTagWeightDict`, is **commented
  out inside a docstring**:

  `FLAF/Analysis/HistMergerFromHists.py:254-261`
  ```python
  """
  if global_cfg_dict["ApplyBweight"] == True:
      all_hists_dict_1D = GetBTagWeightDict(
          args.var, all_hists_dict, categories, boosted_categories, boosted_variables
      )
  else:
      all_hists_dict_1D = all_hists_dict
  """
  ```

**Net effect:** `weight_bTagShape_Central` is computed and stored as a column but is effectively
**not applied** to the central MC histograms. The b-tag SF is therefore both *un-renormalized*
(Issue A) *and* *unapplied at the histogram level* (Issue B).

### 1.4 Issue C — `BtagShapeProducer` hardcodes the wrong analysis

`FLAF/Analysis/BtagShapeProducer.py:2`
```python
import Analysis.hh_bbww as analysis
```
This is the **bbWW** analysis module. `BtagShapeProducer` is the component that *produces* the
per-njet-bin renormalization factors (it computes `weight_total` and `weight_noBtag = total/btag`
per bin, whose ratios feed the JSON that `shape_and_norm` consumes). With the wrong analysis module
hardcoded, generating a correct renormalization file for **bbtautau** is not possible without
editing this import.

### 1.5 Proposed fix path (requires an analysis decision + a produced file)

1. Fix `BtagShapeProducer.py:2` to import `Analysis.hh_bbtautau`.
2. Run `BtagShapeProducer` over the 2022EE MC to produce the per-njet-bin renormalization JSON.
3. Switch `config/global.yaml` btag mode `AnaTuple: shape` → `shape_and_norm`, and configure the
   `norm_file_path` + `bins` so `btag_norm` (`btagShapeWeightCorrector`) is initialized
   (`Corrections/Corrections.py:624-626` asserts it must be).
4. Ensure the central b-tag weight is folded into `final_weight` — either by re-enabling the
   `GetBTagWeightDict` path (`HistMergerFromHists.py:254-261`) or by adding
   `weight_bTagShape_Central` to `GetWeight` per resolved category.

**Quick confirmation before the full fix:** re-plot one b-tag category (e.g. `res2b`) with
`weight_bTagShape_Central` manually folded into the weight and check whether Data/MC agreement
improves. If it does, that confirms the diagnosis.

---

## 2. Secondary — ggH→ττ sample choice (double-count risk)

**Correction to the earlier draft:** an earlier version of this report claimed `GluGluHto2Tau_M125`
was *missing* from 2022EE — that was based on a flawed (group-level) grep and is **wrong**.
`GluGluHto2Tau_M125` is defined in the 2022EE `ggH` group.

The real rule: `GluGluHto2Tau_M125` and `GluGluHto2Tau_UncorrelatedDecay_UnFiltered` describe the
**same** ggH→ττ process — they must **not both be active**, or ggH→ττ is double-counted. Preference
is `GluGluHto2Tau_M125`; `..._UncorrelatedDecay_UnFiltered` is the fallback when M125 is unavailable.

Current state of `config/Run3_2022EE/processes.yaml` `ggH` group: `M125` commented out,
`UnFiltered` active — i.e. **a single ggH→ττ sample, no double-count**. This is acceptable. If the
M125 dataset becomes available for 2022EE, swap to it (enable M125, comment out UnFiltered).
Effect either way is small (ggH→ττ is a minor background).

*Process-list double-counts checked and ruled out for 2022EE:* `TT` vs `TT_Inclusive` (the inclusive
`TT` dataset is excluded for 2022EE via `FLAF/config/dataset_exceptions.yaml`, so `TT_Inclusive` is
inert — no ttbar double-count); all `_ext1` extension samples are commented out.

---

## 3. Bugs fixed in this pass (real, but systematics-only)

These were verified against the source and corrected. **Neither affects the central Data/MC plots**
— they only corrupt systematic up/down variations used in the statistical fit — but both were
genuinely wrong.

### 3.1 `Corrections/tau.h` — `UncSource` enum value collision (Fixed)

Before:
```cpp
stat1_dm0 = 6,
stat2_dm0 = 7,
stat1_dm1 = 7,   // <-- duplicate of stat2_dm0
stat2_dm1 = 8,
...
```
`stat2_dm0` and `stat1_dm1` shared the integer value `7`. Because the SF map is keyed on
`std::pair<UncSource, UncScale>`, the two were indistinguishable — one silently overwrote the other,
swapping the DM0 stat-bin-2 and DM1 stat-bin-1 tau-ID systematics.

After (renumbered the stat block contiguously; values are used only symbolically, so no external
dependency — verified no integer-range iteration over the enum):
```cpp
stat1_dm0 = 6,
stat2_dm0 = 7,
stat1_dm1 = 8,
stat2_dm1 = 9,
stat1_dm10 = 10,
stat2_dm10 = 11,
stat1_dm11 = 12,
stat2_dm11 = 13,
syst_alleras = 14,
... total = 20,
```

### 3.2 `Analysis/GetCrossWeights.py:645` — `mutau_tau_decayMode` copy-paste (Fixed)

Before:
```python
"tau1_isMatched_mutau_tauLeg ? tau1_decayMode : (tau1_isMatched_mutau_tauLeg ? tau2_decayMode: -1.f)"
```
The second condition repeated `tau1_isMatched_mutau_tauLeg`, so `tau2_decayMode` could never be
returned — events where tau2 is the matched tau leg got `-1.f`. (Compare the correct sibling at
`GetCrossWeights.py:386`, which uses `tau2_isMatched_mutau_tauLeg`.) This fed wrong decay-mode-split
trigger-SF uncertainties for the muTau tau leg.

After:
```python
"tau1_isMatched_mutau_tauLeg ? tau1_decayMode : (tau2_isMatched_mutau_tauLeg ? tau2_decayMode: -1.f)"
```

---

## 4. Checked and found SOUND (ruled out as the cause)

These were investigated specifically because they are the usual suspects for pervasive
disagreement, and each was verified correct.

- **Normalization chain (genWeight / sum-of-weights consistency).** The per-event numerator uses
  `weight_gen = std::copysign(1.f, genWeight)` (`Corrections/Corrections.py:537-542`), and the
  normalization **denominator** uses the *same* sign-only definition plus the same PU weight, summed
  over selected + unselected events (`FLAF/AnaProd/anaTupleProducer.py:201-230`). Numerator and
  denominator are gated by the same `use_genWeight_sign_only` flag — internally consistent.
  `weight_base = weight_gen * weight_lumi * weight_xs * (shape weights) / denom`
  (`Corrections/Corrections.py:597`).
- **Luminosity.** `FLAF/config/Run3_2022EE/global.yaml:2` → `luminosity: 26674.92` pb⁻¹
  (= 26.7 fb⁻¹), correct for 2022 post-EE.
- **DY reweighting (`dy_hhbbtautau`).** `Corrections/DY_hhbbtautau.py` maps
  `Run3_2022EE → "2022postEE"`, applies the correction only for `njets ≥ 2` and `valid`, with inputs
  `nJet`, `nBJets`, `pt_ll_gen = LHE_Vpt` (`Analysis/hh_bbtautau.py:695`). Applied only to the `DY`
  process group via `isDY` (`Analysis/histTupleDef.py:109`). Correctly wired.
- **QCD ABCD estimation.** `Analysis/QCD_estimation.py::QCD_Estimation` implements the standard ABCD:
  `QCD(OS_Iso) = SS_Iso_data × (OS_AntiIso / SS_AntiIso)`, with MC subtracted in each region. The
  data process is correctly excluded from the MC-subtraction loop — the skip uses `sample == "Data"`
  (capital), which **matches** the config's data group name `Data` (`config/Run3_2022EE/processes.yaml:233`,
  and `dataset_type` keys). *(Note: the sibling functions `QCD_Estimation_symm` / `_Inverted` use the
  lowercase `"data"` spelling, an inconsistency to watch if they are ever switched to — but the
  active path is correct.)*
- **Trigger scale factors.** Contrary to `issues.md`, trigger SFs **are** applied. The real SFs
  `weight_trgSF_eTau/muTau/tauTau_Central` live in `ID_weights_dict`
  (`Analysis/hh_bbtautau.py:223,231,236`), which **is** extended into the weight
  (`Analysis/hh_bbtautau.py:267`); `defineTriggersCentralWeights` runs because `trigger` is enabled
  (`config/global.yaml:108`). The commented-out `trg_weights_dict` line is an unused older HLT scheme.

---

## 5. `issues.md` false positives — do NOT chase these

The repo-root `issues.md` is an auto-generated audit. Several of its highest-severity items are
**not real bugs** (verified against the source):

- **ditaujet `&` vs `&&`** (`config/Run3_2022EE/triggers.yaml:54`). Flagged as a logic bug; it is
  not. `!=` binds tighter than `&` in C++/cling, so
  `(bits&2048)!=0 & (bits&16384)!=0` evaluates as the bitwise-AND of two `0/1` booleans — identical
  to logical AND. Cosmetic only.
- **"Trigger SFs commented out"** (`Analysis/hh_bbtautau.py:266`). Misleading — see §4; the real
  trigger SFs are applied via `ID_weights_dict`.
- **Always-true `if "muMu" or "eMu" in ...`** (`Analysis/GetCrossWeights.py:428,475`). A real Python
  wart (`bool("muMu")` is always truthy), but harmless: it only ever *defines* an unused
  `weight_trgSF_muMueMu_Central` column; weights are applied per-channel via the `if ({channel})`
  guard in `GetWeight`, so tauTau/eTau/muTau plots are unaffected.

The genuinely-real items from `issues.md` that matter for correctness are the two fixed in §3
(tau enum collision, mutau decay-mode typo) — both systematics-only.

---

## 7. The inclusive (pre-b-tag) disagreement — a second cause

Feedback from the analyst: disagreement is also seen in the **inclusive region, before any b-tag
requirement**. The b-tag SF (§1) is applied only in resolved b-tag categories, so it **cannot**
explain the inclusive disagreement — there is a second, all-events cause.

**What was verified consistent (so NOT the inclusive cause):**
- Weight assembly: `weight_base = weight_gen·weight_lumi·weight_xs·(PU)/denom`, with numerator and
  denominator both using sign-only genWeight + PU (§4).
- Luminosity (26674.92 pb⁻¹), data lumi-mask (`lumiFile`, `FLAF/config/Run3_2022EE/global.yaml:22`),
  and MET filters (`applyMETFlags`, `FLAF/Common/BaselineSelection.py:81`) are all configured.
- **DY stitching** is active (the `*DY_processors` anchor in `config/processes.yaml` attaches an
  `MCStitcher` to the inclusive+jet-binned+PtLL-binned DY M50 samples), and the stitcher's per-bin
  denominator uses the same sign-only convention as the event weight (`FLAF/Processors/MCStitching.py:182-212`)
  — so the inclusive+exclusive DY samples are **not** naively double-counted.
- Trigger/ID SFs applied (§4); QCD ABCD correct (§4); no process double-counts (§2).

**Therefore the inclusive cause is most likely one of:**
1. **An input value** the audit cannot validate without reference numbers:
   - cross-sections (`FLAF/config/crossSections13p6TeV.yaml`) for a dominant background (DY, TT, W+jets);
   - the **PU profile JSON** for 2022EE (a wrong data PU profile shifts MC shape *and*, if mis-applied, normalization);
   - the **DY stitching bin cross-sections** (`FLAF/config/Processors/stitching_DY_amcatnlo_Vpt_NpNLO.yaml`) —
     verify Σ(bin xs) = total xs and that the bin selections cover the actual sample list;
   - lepton (tau/μ/e) ID/iso/trigger SF maps.
2. **Physics-level normalization** — DY and TT from pure MC×σ routinely disagree with data by
   O(10–30%) and are normalised to data in control regions / the fit. This is expected, not a bug.

**To localize it, the decisive input is the per-process breakdown of the inclusive plot:**
- If **one process** (e.g. DY or TT) is high/low → check that process's xs / stitching / reweighting.
- If **everything** is off by a roughly flat factor → suspect PU or a global normalization input.
- If it's a **shape** difference (not normalization) → suspect PU profile or an object-level
  data/MC difference (energy scale, ID efficiency).

### 7.1 Observation (2026-06-15): `M_vis^ττ`, OS-Iso inclusive, all 6 channels

Data sits **below** MC by a roughly **flat ~20%** (Obs/Bkg ≈ 0.7–0.85) in **every** channel —
including the **pure-DY** channels (μμ, ee) *and* the **pure-tt** channel (eμ). A deficit that is
**independent of background composition** is a **global normalization** effect, not DY/TT
mismodeling (which would hit different channels differently). The Z peak is at the correct mass and
the `M_vis` shape is well modeled, so it is a **normalization offset, not a shape/scale problem**.
This is inclusive (pre-b-tag), so it is independent of the §1 b-tag issue, and it is not QCD
(negligible in μμ/ee, which still show the deficit).

**Leading explanation: MC is normalised to more luminosity than the processed data represents.**
MC norm = `lumi(26.67 fb⁻¹) × σ × ε × SFs`; a uniform ~25% excess across all processes points to the
common factor. 26.67 × 0.8 ≈ 21 fb⁻¹ ⇒ a ~5 fb⁻¹ shortfall (≈ one 2022 era), most plausibly from
**missing/failed data files or a lumi/golden-JSON coverage mismatch** (the config lists all eras
E/F/G and all PDs, so this would be a processing/availability gap, not a config gap).

**Decisive check:** compute the integrated lumi of the actually-processed data (brilcalc over the
recorded `runLumiRanges`, `FLAF/AnaProd/anaTupleProducer.py:379`; or verify every data era/PD
produced the expected anaTuple files) and compare to 26.67 fb⁻¹. Quick test: scale the whole MC
stack by ~0.8 — if Obs/Bkg flattens to ~1 in all six channels, the cause is purely global
(lumi/data) and any residual per-channel structure is the real DY/TT physics to be fit.

### 7.2 Confirmation (2026-06-15): η(b₁) and M_vis^HH, OS-Iso inclusive

- **η(b₁): Obs/Bkg is FLAT in η** (~0.65–0.85) in all 6 channels ⇒ a pure **vertical normalization
  offset**, not a shape/detector effect. In particular it is **flat in the endcaps** (|η|>1.5), so it
  is **not** the 2022-postEE ECAL-endcap issue. This is the fingerprint of a **luminosity scale** and
  strongly corroborates §7.1 (global MC over-normalization ⇒ check processed-data lumi).
- **M_vis^HH: Obs/Bkg RISES with mass** (~0.7 at 300 GeV → ~0.85–0.9 above 600 GeV) ⇒ a **secondary,
  smaller shape tilt**: MC relatively too high at **low** HH mass, where **DY** (soft/low-p_T)
  dominates. Likely residual DY shape / p_T modeling — to be addressed **after** the global
  normalization, not before.

**Net:** two independent variables agree — a dominant flat normalization offset (η-flat ⇒ lumi/data)
plus a small low-mass DY residual. Fix the normalization first (§7.1 lumi check).

### 7.3 REVISION (2026-06-15): n_bJets resolves it — NOT a flat offset

The `n_bJets` distribution (OS-Iso inclusive, all 6 channels) **revises §7.1–§7.2**. Obs/Bkg is
**not flat in n_bJets** — it **rises** with b-jet multiplicity, and the rise is **process-dependent**:

| Channel | Obs/Bkg vs n_bJets (bins 2,3,4) |
|---|---|
| μμ, ee (pure **DY**) | **0.65 → 0.85 → ~1** (strong rise, reaching ≳1 at high n_bJets) |
| eτh, μτh, τhτh (DY+tt) | 0.7 → 0.8 → 0.9 (rise) |
| eμ (pure **tt**) | **0.82 → 0.81 → 0.80** (≈ flat) |

**Consequences:**
1. **Luminosity/data-completeness is DEMOTED.** A lumi mismatch is one multiplicative factor → flat
   in every variable, including n_bJets. It is **not** flat ⇒ lumi is not the dominant cause. The
   "flat in η / M_vis" of §7.1–§7.2 was a **projection artifact**: those variables are dominated by
   the huge n_bJets=2 bin and merely reported its ~0.8 value. **n_bJets is the variable that
   resolves the structure.** (Lumi check retained only as a cheap sanity test.)
2. The disagreement is **b-tag-multiplicity dependent and process-split**, i.e. (at least) two causes:
   - **DY: n_btag-shape** (strong rise) → the **missing b-tag shape SF (§1)** and/or the **DY
     reweighting** (which is binned in **n_btags**) not correcting DY heavy-flavor/mistag content.
     ⇒ §1 acts inclusively too, via the b-tag-derived `n_bJets`.
   - **tt: flat ~0.8** (eμ) → tt **normalization** (cross-section / CR scale factor / fit) — possibly
     physics-level rather than a code bug.

**Revised priority:** fix the **b-tag SF (§1) first** and re-plot `n_bJets` (expect the DY rise to
flatten); then verify the **DY reweighting** is applied with the correct n_btag binning; then assess
the residual **tt ~0.8** normalization. This supersedes the §7.1 "lumi first" recommendation.

---

## 6. Recommended next actions

1. **Implement the b-tag fix** (§1.5): fix the `BtagShapeProducer` import, produce the 2022EE
   renormalization JSON, switch to `shape_and_norm`, and ensure the central b-tag weight enters
   `final_weight`. (Confirmed real — addresses the b-tag-category disagreement.)
2. **Localize the inclusive cause** (§7): produce the inclusive plot **per process** and identify
   whether it's one background, a flat factor, or a shape. That single observation determines which
   input (xs / PU / stitching / SF) to check next.
3. **ggH→ττ** (§2): keep exactly one of `GluGluHto2Tau_M125` / `..._UncorrelatedDecay_UnFiltered`
   active (currently OK). Prefer M125 if/when available for 2022EE.
4. After both fixes, residual DY/TT normalization differences are likely physics-level (to be fit),
   not code bugs.
