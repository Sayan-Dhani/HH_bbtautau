7. The inclusive (pre-b-tag) disagreement — a second cause
Feedback from the analyst: disagreement is also seen in the inclusive region, before any b-tag requirement. The b-tag SF (§1) is applied only in resolved b-tag categories, so it cannot explain the inclusive disagreement — there is a second, all-events cause.

What was verified consistent (so NOT the inclusive cause):

Weight assembly: weight_base = weight_gen·weight_lumi·weight_xs·(PU)/denom, with numerator and denominator both using sign-only genWeight + PU (§4).
Luminosity (26674.92 pb⁻¹), data lumi-mask (lumiFile, FLAF/config/Run3_2022EE/global.yaml:22), and MET filters (applyMETFlags, FLAF/Common/BaselineSelection.py:81) are all configured.
DY stitching is active (the *DY_processors anchor in config/processes.yaml attaches an MCStitcher to the inclusive+jet-binned+PtLL-binned DY M50 samples), and the stitcher's per-bin denominator uses the same sign-only convention as the event weight (FLAF/Processors/MCStitching.py:182-212) — so the inclusive+exclusive DY samples are not naively double-counted.
Trigger/ID SFs applied (§4); QCD ABCD correct (§4); no process double-counts (§2).
Therefore the inclusive cause is most likely one of:

An input value the audit cannot validate without reference numbers:
cross-sections (FLAF/config/crossSections13p6TeV.yaml) for a dominant background (DY, TT, W+jets);
the PU profile JSON for 2022EE (a wrong data PU profile shifts MC shape and, if mis-applied, normalization);
the DY stitching bin cross-sections (FLAF/config/Processors/stitching_DY_amcatnlo_Vpt_NpNLO.yaml) — verify Σ(bin xs) = total xs and that the bin selections cover the actual sample list;
lepton (tau/μ/e) ID/iso/trigger SF maps.
Physics-level normalization — DY and TT from pure MC×σ routinely disagree with data by O(10–30%) and are normalised to data in control regions / the fit. This is expected, not a bug.
To localize it, the decisive input is the per-process breakdown of the inclusive plot:

If one process (e.g. DY or TT) is high/low → check that process's xs / stitching / reweighting.
If everything is off by a roughly flat factor → suspect PU or a global normalization input.
If it's a shape difference (not normalization) → suspect PU profile or an object-level data/MC difference (energy scale, ID efficiency).
7.1 Observation (2026-06-15): M_vis^ττ, OS-Iso inclusive, all 6 channels
Data sits below MC by a roughly flat ~20% (Obs/Bkg ≈ 0.7–0.85) in every channel — including the pure-DY channels (μμ, ee) and the pure-tt channel (eμ). A deficit that is independent of background composition is a global normalization effect, not DY/TT mismodeling (which would hit different channels differently). The Z peak is at the correct mass and the M_vis shape is well modeled, so it is a normalization offset, not a shape/scale problem. This is inclusive (pre-b-tag), so it is independent of the §1 b-tag issue, and it is not QCD (negligible in μμ/ee, which still show the deficit).

Leading explanation: MC is normalised to more luminosity than the processed data represents. MC norm = lumi(26.67 fb⁻¹) × σ × ε × SFs; a uniform ~25% excess across all processes points to the common factor. 26.67 × 0.8 ≈ 21 fb⁻¹ ⇒ a ~5 fb⁻¹ shortfall (≈ one 2022 era), most plausibly from missing/failed data files or a lumi/golden-JSON coverage mismatch (the config lists all eras E/F/G and all PDs, so this would be a processing/availability gap, not a config gap).

Decisive check: compute the integrated lumi of the actually-processed data (brilcalc over the recorded runLumiRanges, FLAF/AnaProd/anaTupleProducer.py:379; or verify every data era/PD produced the expected anaTuple files) and compare to 26.67 fb⁻¹. Quick test: scale the whole MC stack by ~0.8 — if Obs/Bkg flattens to ~1 in all six channels, the cause is purely global (lumi/data) and any residual per-channel structure is the real DY/TT physics to be fit.

7.2 Confirmation (2026-06-15): η(b₁) and M_vis^HH, OS-Iso inclusive
η(b₁): Obs/Bkg is FLAT in η (~0.65–0.85) in all 6 channels ⇒ a pure vertical normalization offset, not a shape/detector effect. In particular it is flat in the endcaps (|η|>1.5), so it is not the 2022-postEE ECAL-endcap issue. This is the fingerprint of a luminosity scale and strongly corroborates §7.1 (global MC over-normalization ⇒ check processed-data lumi).
M_vis^HH: Obs/Bkg RISES with mass (~0.7 at 300 GeV → ~0.85–0.9 above 600 GeV) ⇒ a secondary, smaller shape tilt: MC relatively too high at low HH mass, where DY (soft/low-p_T) dominates. Likely residual DY shape / p_T modeling — to be addressed after the global normalization, not before.
Net: two independent variables agree — a dominant flat normalization offset (η-flat ⇒ lumi/data) plus a small low-mass DY residual. Fix the normalization first (§7.1 lumi check).

7.3 REVISION (2026-06-15): n_bJets resolves it — NOT a flat offset
The n_bJets distribution (OS-Iso inclusive, all 6 channels) revises §7.1–§7.2. Obs/Bkg is not flat in n_bJets — it rises with b-jet multiplicity, and the rise is process-dependent:

Channel	Obs/Bkg vs n_bJets (bins 2,3,4)
μμ, ee (pure DY)	0.65 → 0.85 → ~1 (strong rise, reaching ≳1 at high n_bJets)
eτh, μτh, τhτh (DY+tt)	0.7 → 0.8 → 0.9 (rise)
eμ (pure tt)	0.82 → 0.81 → 0.80 (≈ flat)
Consequences:

Luminosity/data-completeness is DEMOTED. A lumi mismatch is one multiplicative factor → flat in every variable, including n_bJets. It is not flat ⇒ lumi is not the dominant cause. The "flat in η / M_vis" of §7.1–§7.2 was a projection artifact: those variables are dominated by the huge n_bJets=2 bin and merely reported its ~0.8 value. n_bJets is the variable that resolves the structure. (Lumi check retained only as a cheap sanity test.)
The disagreement is b-tag-multiplicity dependent and process-split, i.e. (at least) two causes:
DY: n_btag-shape (strong rise) → the missing b-tag shape SF (§1) and/or the DY reweighting (which is binned in n_btags) not correcting DY heavy-flavor/mistag content. ⇒ §1 acts inclusively too, via the b-tag-derived n_bJets.
tt: flat ~0.8 (eμ) → tt normalization (cross-section / CR scale factor / fit) — possibly physics-level rather than a code bug.
Revised priority: fix the b-tag SF (§1) first and re-plot n_bJets (expect the DY rise to flatten); then verify the DY reweighting is applied with the correct n_btag binning; then assess the residual tt ~0.8 normalization. This supersedes the §7.1 "lumi first" recommendation.


Could you please explain this more ? And what I need to check in order to confirm it @DataMC_2022EE_investigation.md
Let me explain §7 in plain terms, then give you the concrete checks. I have the full report in context, so I'll work from that.

What §7 is actually claiming
§7 is about a second, separate Data/MC problem from the b-tag one in §1.

§1 problem lives only in the resolved b-tag categories (res1b/res2b). The b-tag shape weight is computed but never folded into the histogram weight, so those categories are mis-normalized.
§7 problem is seen in the inclusive region — before any b-tag cut. The §1 weight is never applied there anyway (GetBTagWeight only returns it for resolved, non-baseline categories), so §1 cannot be the explanation. Hence "a second cause."
The whole of §7 is then a story of the diagnosis changing as more plots came in. That's the important thing to understand — the early subsections are partly superseded:

Subsection	What was seen	Conclusion at the time
§7.1	M_vis flat ~20% low in all 6 channels	"Looks global → blame luminosity"
§7.2	η(b₁) flat, M_vis^HH rises	"Confirms flat offset → luminosity"
§7.3	n_bJets is NOT flat — it rises, process-dependent	Supersedes the above: not lumi
Why §7.3 overturns the "luminosity" idea
This is the key logical step. A luminosity / missing-data error is a single multiplicative number. If MC is normalized to 26.67 fb⁻¹ but the data you actually processed only covers ~21 fb⁻¹, every MC histogram is too high by the same factor — and that factor is flat in every variable you ever plot. It would be flat in M_vis, flat in η, and flat in n_bJets.

But §7.3 found Obs/Bkg rises with b-jet multiplicity, and the rise depends on the channel/process:

μμ, ee (pure DY): 0.65 → 0.85 → ~1 (strong rise)
eμ (pure tt): 0.82 → 0.81 → 0.80 (basically flat)
A single lumi number can't produce a slope, and certainly can't produce a different slope for DY vs tt. So lumi is demoted to a cheap sanity check. The earlier "flat in η/M_vis" was a projection artifact: those variables are completely dominated by the n_bJets=2 bin, so a 1-D η plot just reports that one bin's ~0.8 value and hides the structure.

The revised picture is two overlapping causes:

DY — the rise vs n_bJets is the fingerprint of a b-tag/heavy-flavor modeling problem → the missing b-tag shape SF (§1) acting inclusively via the b-tag-derived n_bJets, and/or the DY reweighting (which is binned in n_btags) not correcting DY's heavy-flavor content.
tt — flat ~0.8 in eμ → a plain normalization offset (cross-section / control-region scale / fit), likely physics-level, not a code bug.
What to check to confirm it
Here's the decisive ordered list. Each step either confirms or kills a hypothesis.

1. Confirm it really is process-split (the foundation of §7.3).
The whole revision rests on the per-channel n_bJets table. Re-make n_bJets, OS-Iso inclusive, as a stacked per-process plot (not just total Obs/Bkg) for at least the three "pure" channels:

μμ or ee → should be ~100% DY
eμ → should be ~100% tt
If μμ rises and eμ stays flat, §7.3 is confirmed. If they actually behave the same, you're back to a global (lumi) effect.

2. Cheap lumi sanity check (kills or keeps the global hypothesis).
Scale the entire MC stack by 0.8 and overlay:

If n_bJets flattens to ~1 in all channels → it was global after all (lumi), reopen §7.1.
If M_vis flattens but n_bJets still has a residual slope → lumi is at most partial; the slope is the real effect.
Properly: run brilcalc over the actually-processed runLumiRanges (recorded at FLAF/AnaProd/anaTupleProducer.py:379) and compare the integrated lumi to 26674.92 pb⁻¹. A ~5 fb⁻¹ shortfall would mean missing/failed data files.

3. Confirm the DY cause — does the b-tag SF flatten the DY rise?
This is the §1 fix being tested inclusively. In a pure-DY channel (μμ), re-plot n_bJets with weight_bTagShape_Central folded into the weight:

If the 0.65→1 rise flattens → confirms the DY rise is the missing b-tag shape SF.
If it persists → the DY reweighting binning is the suspect; verify dy_hhbbtautau is applied with the correct n_btag binning (inputs nJet/nBJets/LHE_Vpt, Analysis/hh_bbtautau.py:695), and that the n_btag axis definition matches between the correction map and the event.
4. Confirm the tt cause — is eμ a flat offset independent of n_bJets?
eμ is ~pure tt. If Obs/Bkg there is flat ~0.8 vs n_bJets and vs other variables, it's a tt normalization issue. Check the TT cross-section in FLAF/config/crossSections13p6TeV.yaml, and confirm TT_Inclusive is genuinely inert for 2022EE (it should be excluded via dataset_exceptions.yaml per §2 — verify no ttbar double-count). A flat ~0.8 in pure tt is also a normal MC×σ vs data offset that gets absorbed by the fit, so this one may be "expected, not a bug."

Bottom line: the single most decisive plot is step 1 — the per-process n_bJets in the pure channels. Everything else (lumi scale, b-tag SF fold-in, tt xs) is a follow-up that the shape of that plot tells you whether to pursue.

Want me to draft the b-tag-SF-folded-in weight for step 3 so you can produce that confirmation plot directly?


image.png
1182×1153

image.png
1182×1153

image.png
1182×1153

image.png
1182×1153

image.png
1182×1153

image.png
1182×1153
These are the MET_pt in OS_ISO_Inclusive for 6 channel
These six MET plots are genuinely informative — they split cleanly by process and they show something the 1‑D η/M_vis plots couldn't. Let me read them honestly, channel by channel, then say what it means.

What each ratio panel actually shows
Channel	Dominant process	Obs/Bkg vs MET	Shape
eμ	pure tt (green)	~0.80 at 0 → 0.83 (60) → 0.75 (200)	Flat ~0.8
ee	pure DY (blue)	0.75 → dip to ~0.60 at MET≈55 → 0.77 (120) → 0.7	U-shaped dip
μμ	pure DY (blue)	0.72 → dip to ~0.58 at MET≈65 → 0.78 (200)	U-shaped dip
eτh	DY+tt+W	0.85 → 0.70, then ~flat	mild fall then flat
μτh	DY+tt+W	0.65 → rises to 0.77	gentle rise
τhτh	QCD (grey)+DY	~0.85–0.90 low MET → falls to ~0.6 high MET	best at low MET
The two new facts
1. The pure-tt channel (eμ) is flat ~0.8. This nails down the tt piece exactly as §7.3 predicted: a clean ~20% normalization offset, no MET shape. That's a cross-section / CR-scale / fit-level issue, very likely physics, not a code bug.

2. The two pure-DY channels (ee, μμ) both show the same U-shaped dip — Obs/Bkg bottoms out at ~0.58–0.60 around MET ≈ 50–70 GeV and recovers at both low and high MET. This is a genuine MET shape, reproduced independently in two channels, so it's real. DY (Z→ℓℓ) has no genuine MET — its MET is entirely detector recoil/resolution. A data-below-MC dip at intermediate MET is the classic fingerprint of missing or wrong Z‑recoil / MET modeling in DY. This is new: it's not the n_bJets/heavy-flavor effect of §7.3, it's a MET-recoil effect on top.

One caveat that slightly rehabilitates the "global" idea
Notice that both pure channels share a common ~0.8 floor: eμ (pure tt) sits at flat 0.8, and ee/μμ (pure DY) oscillate around ~0.7 with the dip riding on top. A ~20% deficit appearing in pure-tt and pure-DY alike is exactly what a shared multiplicative factor (luminosity / processed-data shortfall) would also produce. §7.3 demoted lumi because n_bJets wasn't flat — but these plots show the baseline offset really is common across processes. So the honest decomposition now is:

a common ~20% floor (could be global: lumi/data completeness — still worth the cheap test), plus
tt: nothing extra → pure normalization, plus
DY: a MET-recoil shape (the dip) and the n_bJets rise from §7.3.
What to check to confirm
DY MET/recoil modeling (the dip — highest-value new check). Find out whether any Z‑recoil correction or MET‑phi/XY correction is applied to DY for 2022EE, and whether it's even wired in. If none is applied, the mid-MET dip in ee/μμ is explained. Decisive plot: overlay ee/μμ MET with and without the recoil correction — the dip should fill in.

The common ~0.8 floor (cheap, do it once). Scale the whole MC stack by 0.8: if the baseline of all six channels moves to ~1 (leaving the DY dip and the tt-vs-DY difference as residuals), the floor is global → run brilcalc over the processed runLumiRanges (FLAF/AnaProd/anaTupleProducer.py:379) vs 26674.92 pb⁻¹ to check for a data shortfall.

tt normalization (eμ). Confirm eμ stays flat vs every variable (MET ✓, n_bJets per §7.3). If so, it's a normalization knob: check the TT cross-section in FLAF/config/crossSections13p6TeV.yaml and that TT_Inclusive is inert for 2022EE (no double-count).

DY n_bJets (the §7.3 rise) is separate from this dip. Keep both: the MET dip is a recoil/shape effect; the n_bJets rise is heavy-flavor/b-tag. They need different fixes.

The single most valuable new check is #1 — DY MET-recoil modeling — because the dip is a reproduced, process-specific, shape feature that a normalization fix cannot remove.

Want me to grep the FLAF/Corrections code to see whether a DY MET-recoil correction exists and is applied for 2022EE? That would directly confirm or kill check #1.


image.png
1182×1153

image.png
1182×1153

image.png
1182×1153

image.png
1182×1153

image.png
1182×1153

image.png
1182×1153
This is bb_m_vis in OS_ISO_Inclusive for 6 channel
M_bb is the clincher for separating "shape" from "normalization." Reading the ratio panels:

Channel	Dominant	Obs/Bkg vs M_bb	Shape
eμ	pure tt	~0.70 at low → 0.82 → flat ~0.80 to 500	flat ~0.8
ee	pure DY	flat ~0.70, gentle droop to 0.65	flat ~0.7
μμ	pure DY	flat ~0.68–0.70 across the whole range	flat ~0.7
eτh	DY+tt+W	flat ~0.78, rises toward 1 above 350	flat ~0.78
μτh	DY+tt+W	flat ~0.65–0.72	flat ~0.68
τhτh	QCD+DY	~0.8–1.0, noisy within large unc.	~flat, uncertain
What M_bb adds — three clean conclusions
1. The DY deficit is FLAT in M_bb — the MET dip is gone. In ee/μμ the U‑shaped dip you saw in MET (~0.58 at MET≈60) is absent here; M_bb is flat ~0.7. That's exactly what you'd expect if the dip is a MET-recoil effect: it shows up only in the MET variable and nowhere else. This confirms check #1 from the last message — the DY problem has a genuine MET-recoil component that is localized to MET, not a global event mismodeling.

2. M_bb being flat rules out the b-jet 4-vector as the DY culprit. If DY's disagreement came from b-jet energy scale/resolution or a b-mass mismodeling, M_bb would be tilted (peak shifted, slope). It's flat. So JES/JER/b-tag-energy is not driving the DY deficit — the DY problem is normalization + MET-recoil + the n_bJets/heavy-flavor rise, not b-jet kinematics.

3. DY channels sit at ~0.70, the tt channel at ~0.80 — these differ, so it is NOT a single luminosity factor. This is the sharpest statement yet against the §7.1 "lumi" idea. A pure lumi/data shortfall is one number → pure-DY and pure-tt channels would land at the same ratio. They don't: DY is ~30% high, tt is ~20% high. That ~10-point gap is a process-dependent over-normalization of DY relative to tt — physics/reweighting, not a global scale.

The consolidated picture across all three variables
Variable	DY channels (ee/μμ)	tt channel (eμ)
n_bJets (§7.3)	rises 0.65→1	flat ~0.8
MET	U-dip, min ~0.58 at mid-MET	flat ~0.8
M_bb	flat ~0.70	flat ~0.80
So DY carries three distinct effects — a flat ~30% over-normalization, a MET-recoil dip, and an n_btag rise — while tt is a clean flat ~20% offset and nothing else. That's a process-split diagnosis with DY clearly the more structured problem.

Refined decisive checks (in priority order)
Is the DY reweighting (dy_hhbbtautau) actually moving these inclusive plots, and by how much? This is now the single most valuable check. Plot ee/μμ M_bb (or any flat variable) with and without weight_dy_central. If DY jumps from ~0.7 toward ~1, the reweighting is the lever and the residual is its mis-tuning; if it barely moves, the reweighting isn't reaching the inclusive selection (recall it only applies for njets≥2 & valid, Analysis/hh_bbtautau.py:695) — which would itself explain a flat DY excess.

DY MET-recoil (the dip) — check whether any Z-recoil/MET-XY correction exists and is applied to DY for 2022EE.

tt flat 0.8 — TT cross-section in FLAF/config/crossSections13p6TeV.yaml; confirm TT_Inclusive is inert (no double-count).

I can grep the FLAF/Corrections + Analysis code right now to answer both #1 (does weight_dy_central actually reach the inclusive selection?) and #2 (does a DY MET-recoil correction exist for 2022EE?). Want me to do that?

@Corrections/Corrections.py  @Corrections/data/DY_corr_hbt/hbt_corrections_v2.json.gz  @Corrections/DY_hhbbtautau.py @Corrections/DY_hhbbtautau.h 
Sayan@lxplus904:~/HH_bbtautau/Corrections/data/DY_corr_hbt $ correction summary hbt_corrections_v2.json.gz                                 
────────────────────────── Corrections in file hbt_corrections_v2.json.gz ──────────────────────────
CorrectionSet (schema v2)
Corrections derived for the hh2bbtautau analysis.
📂
└── 📈 dy_weight (v1)
    DY weights derived in the phase space of the hh2bbtautau analysis, supposed to correct njet and 
    ptll distributions, as well as correlated quantities.
    Node counts: Category: 6, Binning: 450, Formula: 1125
    ╭────────────────────────────────────────── ▶ input ───────────────────────────────────────────╮
    │ era (string)                                                                                 │
    │ Era name.                                                                                    │
    │ Values: 2022postEE, 2022preEE, 2023postBPix, 2023preBPix, 2024                               │
    ╰──────────────────────────────────────────────────────────────────────────────────────────────╯
    ╭────────────────────────────────────────── ▶ input ───────────────────────────────────────────╮
    │ njets (int)                                                                                  │
    │ Number of jets in the event.                                                                 │
    │ Values:                                                                                      │
    ╰──────────────────────────────────────────────────────────────────────────────────────────────╯
    ╭────────────────────────────────────────── ▶ input ───────────────────────────────────────────╮
    │ ntags (int)                                                                                  │
    │ Number of (PNet) b-tagged jets.                                                              │
    │ Values:                                                                                      │
    ╰──────────────────────────────────────────────────────────────────────────────────────────────╯
    ╭────────────────────────────────────────── ▶ input ───────────────────────────────────────────╮
    │ ptll (real)                                                                                  │
    │ Gen level pT of the dilepton system [GeV].                                                   │
    │ Range: [-inf, inf), overflow ok                                                              │
    ╰──────────────────────────────────────────────────────────────────────────────────────────────╯
    ╭────────────────────────────────────────── ▶ input ───────────────────────────────────────────╮
    │ syst (string)                                                                                │
    │ Systematic variation.                                                                        │
    │ Values: nominal, stat_btag0_down, stat_btag0_up, stat_btag1_down, stat_btag1_up,             │
    │ stat_btag2_down, stat_btag2_up, stat_down, stat_up, syst_down, syst_gauss_down,              │
    │ syst_gauss_up, syst_linear_down, syst_linear_up, syst_up                                     │
    ╰──────────────────────────────────────────────────────────────────────────────────────────────╯
    ╭──── ◀ output ────╮
    │ weight (real)    │
    │ DY event weight. │
    ╰──────────────────╯
Sayan@lxplus904:~/HH_bbtautau/Corrections/data/DY_corr_hbt $ 

 Check this if they are correctly implemented or not ?
I've read all four. Here's the verdict, split into "is the code correct" (yes) and "does that explain the plots" (partially — and it tells us exactly what the DY correction can and cannot fix).

The mechanical implementation is correct
Checking each link in the chain:

Era mapping ✓. Run3_2022EE → "2022postEE" (DY_hhbbtautau.py:14), and the JSON's era axis lists 2022postEE. Match.
Input signature ✓. The C++ call passes (era, njets, ntags, ptll, syst) in that order (DY_hhbbtautau.py:119-126), which is exactly the JSON's input order (era, njets, ntags, ptll, syst). No transposition.
Guards ✓. DY_hhbbtautau.h:28-36: weight = 1.0 if !isValid or njets < 2, else evaluate. Sensible.
Branch defaults ✓ but worth confirming: njets_branch="nJet", ntags_branch="nBJets", pt_ll_gen="pt_ll_gen" (Corrections.py:200-213). These are defaults — the config can override them. They must point at the right columns (see below).
So there is no bug in these four files. The producer is clean.

But two structural facts explain the plots
1. The DY correction has NO MET dependence — so it structurally cannot fix the ee/μμ MET dip. Its axes are njets, ntags, ptll only. The MET-recoil dip you saw (min ~0.58 at MET≈60 in the pure-DY channels) is in a variable this correction doesn't even see. This confirms the earlier diagnosis: the MET dip is a separate, missing Z-recoil/MET correction, and no amount of tuning dy_weight will remove it.

2. The correction IS binned in ntags — so it is the right lever for the §7.3 n_bJets rise, IF the binning matches. This is the key thing to verify. The JSON ntags is "Number of (PNet) b-tagged jets." The producer feeds it nBJets. You must confirm that nBJets is the PNet-tagged count at the same working point the correction was derived with. If nBJets uses a different tagger or WP (e.g. DeepJet, or a different WP), every event lands in the wrong ntags bin → the correction is applied but mis-binned → the n_bJets rise wouldn't flatten. Same caveat for nJet vs the correction's jet definition.

The decisive runtime check
The plots show DY sitting flat ~0.7 (M_bb, M_vis, η). The question the code can't answer statically is: is weight_dy_central actually moving the DY yield, or is it ≈1? Two cheap checks settle it:

Dump weight_dy_central for DY events — its mean, and its mean split by nBJets (2, 3, 4).

If mean ≈ 1 and flat in nBJets → the correction is doing essentially nothing; DY's ~0.7 offset and n_bJets rise are uncorrected (either it's not reaching these events, or it's a shape-only/unit-normalized correction that doesn't touch normalization).
If mean < 1 and varies with nBJets → it is correcting, and the residual ~0.7 is physics-level DY normalization (to be fit) plus the mis-binning question above.
Confirm two things I can't see from these files (I can grep them now):

that weight_dy_central is actually multiplied into the DY histograms at the hist stage (the investigation claimed histTupleDef.py:109 via isDY — worth confirming it survived the submodule bumps);
how nBJets and the pt_ll_gen branch are actually defined, to check the ntags/PNet-WP match in point 2 above.
Want me to grep FLAF/Analysis + the config for (a) where weight_dy_central enters the final weight and (b) the definitions of nBJets and pt_ll_gen? That closes the last gap between "the code is correct" and "the correction is actually reaching these plots with the right binning.