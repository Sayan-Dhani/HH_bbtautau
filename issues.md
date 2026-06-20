Now I have all the information needed to write a comprehensive issues report.

# HH→bbττ Analysis Repository: Comprehensive Issues Report

## Table of Contents

1. [AnaProd/](#anaprod)
2. [Analysis/](#analysis)
3. [Corrections/](#corrections)
4. [Config/](#config)
5. [FLAF/](#flaf)
6. [include/ (C++ Headers)](#include-c-headers)
7. [HHbtag/](#hhbtag)
8. [HHKinFit2/](#hhkinfit2)
9. [StatInference/](#statinference)
10. [inference/ (dhi)](#inference-dhi)
11. [Docs/](#docs)
12. [Cross-Cutting Issues](#cross-cutting-issues)

---

## AnaProd/

### [CRITICAL] Broken relative import in `LegacyVariables.py`
**File:** `AnaProd/LegacyVariables.py`, line 2
**Problem:** `from .Utilities import *` is a relative import that requires `AnaProd/` to be a proper Python package with an `__init__.py`. No `__init__.py` exists in `AnaProd/`. This file will raise `ImportError: attempted relative import with no known parent package` if imported from any external context.
**Impact:** Any code path that reaches this file will crash at import time.
**Fix:** Either add `__init__.py` to `AnaProd/` or change the import to `from FLAF.Common.Utilities import *` (matching what `baseline.py` does). Given `addLegacyVariables.py` correctly uses `FLAF.Common.LegacyVariables`, this file is a broken duplicate and should be removed or corrected.

### [CRITICAL] `NameError` in `NNInterface.py` when used as a module
**File:** `AnaProd/NNInterface.py`, lines 297–298 and 434
**Problem:** `PrepareDfForDNN` and `DataFrameBuilderForHistograms` are imported inside the `if __name__ == "__main__"` block (line 434: `from Analysis.hh_bbtautau import *`). However, `run_inference_for_tree()` at lines 297–298 calls both these names unconditionally without any import. If `NNInterface.py` is imported as a module (not run as a script), `run_inference_for_tree` will raise `NameError: name 'PrepareDfForDNN' is not defined` at the call site.
**Impact:** Silent breakage of the module import path; only works when run directly as `python NNInterface.py`.
**Fix:** Move the `from Analysis.hh_bbtautau import *` to the top of the file, or add a local import at the top of `run_inference_for_tree`.

### [HIGH] Bare `from interface import *` requires fragile `sys.path` state
**File:** `AnaProd/NNInterface.py`, line 13
**Problem:** `from interface import *` is a bare import that only succeeds if `AnaProd/` is explicitly on `sys.path`. It does not use the full package path `from AnaProd.interface import *`, making the import resolution entirely dependent on the working directory or manual `sys.path` manipulation. Furthermore, this will silently shadow symbols from `Analysis/interface.py` if both modules happen to be on the path.
**Impact:** Import failures in any production context where the working directory is not `AnaProd/`.
**Fix:** Change to `from AnaProd.interface import NNInterface, DotDict, Era` or ensure `sys.path` is set before the import.

### [HIGH] Hardcoded AFS path to another user's workspace
**File:** `AnaProd/interface.py`, line 327
**Problem:** The `__main__` test block contains `model_path="/afs/cern.ch/work/a/acagnott/Hbbtautau/HH_bbtautau/test/data/model_fold0_moe"` — an absolute path to user `acagnotta`'s personal AFS space.
**Impact:** Running this test block on any other user's account or machine will fail with a `FileNotFoundError`. AFS paths outside the user's own quota are not universally readable.
**Fix:** Replace with `os.path.join(os.environ["ANALYSIS_PATH"], "test/data/model_fold0_moe")` as done in `Analysis/interface.py`.

### [HIGH] `spin`, `mass`, and `era` DNN inputs are silently discarded
**File:** `AnaProd/interface.py`, lines 262, 279–280; `Analysis/interface.py` (same issue)
**Problem:** `NNInterface.predict()` accepts `spin`, `mass`, and `era` parameters, but the lines that would include them in the input tensor are commented out:
```python
# cont_ones * mass
# cat_ones * era.value
# cat_ones * spin
```
The parameters are accepted, an `Era` enum lookup is performed in `NNInterface.py`, but none of the values reach the model.
**Impact:** If the model was designed to be conditioned on resonant mass, spin, or era, the predictions are wrong. The `Era` enum lookup in `NNInterface.py` wastes computation every event. Developers may incorrectly believe these inputs are active.
**Fix:** Either remove the parameters and the `Era` lookup, or uncomment the tensor population lines and verify the model supports these inputs.

### [MEDIUM] Stale CLI example comment in `NNInterface.py`
**File:** `AnaProd/NNInterface.py`, line 1
**Problem:** The first line shows a stale invocation example using `--EraName e2018 --Mass 400 --Spin 2 --PairType 2`. The actual `argparse` at line 443 defines `--period`, `--mass`, `--spin` — no `--EraName` and no `--PairType` flags.
**Impact:** Misleads users running the script from documentation or history.
**Fix:** Update the comment to reflect the actual CLI interface.

### [MEDIUM] `addLegacyVariables.py` is dead code
**File:** `AnaProd/addLegacyVariables.py`
**Problem:** `applyLegacyVariables(df)` is defined but no other file in the repository calls it (confirmed by grep). The function body correctly uses `FLAF.Common.LegacyVariables`, but since it is never invoked, legacy variable computation (MT2, KinFit, SVFit) is not performed in the standard pipeline.
**Impact:** Either legacy variables are not being computed (pipeline gap) or this file is genuinely obsolete. Either way, it is confusing.
**Fix:** Either wire `applyLegacyVariables` into the appropriate pipeline stage, or remove the file with a comment explaining which pipeline step handles these variables.

### [MEDIUM] Event-by-event Python loop for DNN inference in `NNInterface.py`
**File:** `AnaProd/NNInterface.py`, lines 364–374
**Problem:** The inference loop iterates one event at a time, constructing size-1 numpy arrays and calling TF inference per event. This is O(N) Python-level TF calls instead of a single vectorized batch call.
**Impact:** For a typical analysis dataset of O(10^6) events with 5 folds, this is ~5×10^6 individual TF calls — orders of magnitude slower than the vectorized approach used in `Analysis/DNN_application.py`.
**Fix:** Process events in batches (e.g., batch size 1000–10000) using the same pattern as `Analysis/DNN_application.py`.

### [MEDIUM] Two different `CMSSW_BASE` env vars for the same initialization
**File:** `AnaProd/anaTupleDef.py`, `Initialize()` function
**Problem:** The `Initialize()` function uses `$FLAF_CMSSW_BASE` to locate the shared library (`libHHToolsHHbtag.so`) but uses `$CMSSW_BASE` to locate the HHbtag model directory. These two environment variables may point to different CMSSW installations if the user has customized the environment.
**Impact:** Could silently load the shared library from one CMSSW version but the DNN models from another, leading to incompatible model/library pairs and runtime crashes.
**Fix:** Use a single, consistent env var for both paths — either `$FLAF_CMSSW_BASE` throughout or document precisely when the two differ.

### [LOW] `Era` and `DotDict` classes duplicated across `AnaProd/interface.py` and `AnaProd/NNInterface.py`
**File:** `AnaProd/NNInterface.py`, lines 17–29
**Problem:** `Era` and `DotDict` are defined in `AnaProd/interface.py` and also re-defined locally in `NNInterface.py` after `from interface import *`. The re-definition is redundant.
**Impact:** Maintenance burden — future changes to `Era` must be made in both files.
**Fix:** Remove the local re-definition in `NNInterface.py` and rely on the import.

### [LOW] No `__init__.py` while `anaTupleDef.py` uses `import AnaProd.baseline`
**File:** `AnaProd/` directory
**Problem:** There is no `__init__.py` in `AnaProd/`, yet `anaTupleDef.py` imports `AnaProd.baseline` as a package. This works as a namespace package only because `$ANALYSIS_PATH` (the repo root) is on `sys.path`. This is an implicit convention that could silently break.
**Impact:** Fragile imports in non-standard execution environments.
**Fix:** Add an `__init__.py` to `AnaProd/` to make it an explicit package.

---

## Analysis/

### [CRITICAL] Logic bug in `defineTriggersCentralWeights` — always-true condition
**File:** `Analysis/GetCrossWeights.py`, lines 428 and 475
**Problem:** The guard condition is written as:
```python
if "muMu" or "eMu" in dfBuilder.config["channels_to_consider"]:
```
Python evaluates this as `bool("muMu") or ("eMu" in ...)`, which is always `True` because `"muMu"` is a non-empty string. The intended logic `("muMu" in channels) or ("eMu" in channels)` is never correctly tested.
**Impact:** The muMu/eMu trigger weight block is always executed regardless of what channels are configured, potentially applying muMu/eMu trigger weights to analyses that do not include these channels, which could silently corrupt event weights for tauTau-only or eTau-only analyses.
**Fix:**
```python
if "muMu" in dfBuilder.config["channels_to_consider"] or "eMu" in dfBuilder.config["channels_to_consider"]:
```

### [CRITICAL] Copy-paste bug in `mutau_tau_decayMode` — tau2 branch unreachable
**File:** `Analysis/GetCrossWeights.py`, line 645
**Problem:** The expression is:
```python
"tau1_isMatched_mutau_tauLeg ? tau1_decayMode : (tau1_isMatched_mutau_tauLeg ? tau2_decayMode: -1.f)"
```
Both the first and second condition check `tau1_isMatched_mutau_tauLeg`. The second branch intended to check `tau2_isMatched_mutau_tauLeg`. As written, `tau2_decayMode` can never be returned — events where tau2 is the matched tau leg get `-1.f` instead of the correct decay mode.
**Impact:** Wrong decay-mode-resolved trigger SF uncertainty weights for the tau leg in muTau events where the second leg carries the tau match. The `weight_{dm}_mutau_tauleg_Central` and Up/Down values will be wrong for those events.
**Fix:** Change the second condition to `tau2_isMatched_mutau_tauLeg`.

### [HIGH] Trigger weights commented out of `GetWeight()`
**File:** `Analysis/hh_bbtautau.py`, lines 266 and 278
**Problem:** The lines `weights_list.extend(trg_weights_dict[channel])` are commented out with the remark `## currently commented because there are no trigger weights ??`. Trigger SFs are defined in `GetCrossWeights.py` and are applied conditionally via `histTupleDef.py` only when `"trigger" in global_params.get("corrections", {})`, creating a dual-path logic that is fragile.
**Impact:** If the trigger correction config key is absent or the conditional path is not taken, trigger SFs are silently dropped from the event weight, introducing a bias in all normalized histograms.
**Fix:** Either restore the `extend` call or document clearly (with a config-file check) which mechanism controls trigger SF application and ensure only one path is active.

### [HIGH] Mutable default argument in `DataFrameBuilderForHistograms.__init__`
**File:** `Analysis/hh_bbtautau.py`, line 631
**Problem:** `def __init__(self, ..., colToSave=[])`. The empty list `[]` is a single object created once at class definition time and shared across all instances that do not pass an explicit `colToSave`.
**Impact:** Classic Python pitfall — if any instance modifies `colToSave` in-place (via `.append()` etc.), subsequent instances will inherit those mutations, causing incorrect or inflated column lists being snapshotted to ROOT files.
**Fix:** Change to `colToSave=None` and add `self.colToSave = colToSave if colToSave is not None else []` in the body.

### [HIGH] `GetBTagWeight()` references undefined local variable `btag_wps`
**File:** `Analysis/hh_bbtautau.py`, line 191
**Problem:** Line 191 uses `btag_wps[cat]` where `btag_wps` is not a local variable in `GetBTagWeight()`. The correct reference is `global_cfg_dict["btag_wps"][cat]`, used correctly on line 190. Line 191 contains a bare `btag_wps` that would raise `NameError` if reached.
**Impact:** Any code path that calls `GetBTagWeight(applyBtag=True)` will crash with a `NameError`. The `applyBtag=True` path is apparently not exercised in production currently, masking the bug.
**Fix:** Change `btag_wps[cat]` to `global_cfg_dict["btag_wps"][cat]` on line 191.

### [HIGH] `interface.py` `__main__` block passes `era` kwarg to a function that does not accept it
**File:** `Analysis/interface.py`, line 334
**Problem:** The `__main__` test block calls `nn0(...)` with `era=Era.Run3_2022EE`, but `NNInterface.predict()` does not list `era` as a parameter (the line `# era: Era,` at line 131 is commented out).
**Impact:** Running the test block `python Analysis/interface.py` raises `TypeError: predict() got an unexpected keyword argument 'era'`.
**Fix:** Either uncomment the `era` parameter in `predict()` and implement its use, or remove the `era=` argument from the test call.

### [MEDIUM] `make_stackplots.py` hardcodes placeholder EOS paths
**File:** `Analysis/make_stackplots.py`, lines 5–6
**Problem:** `indir = f"/eos/user/u/username/HH_bbtautau_Run3/histograms/{ver}/{era}/merged/"` and similar — `username` is a literal placeholder, not a variable.
**Impact:** The script cannot be run without manual editing. It also lacks an `argparse` interface.
**Fix:** Replace `username` with `os.environ.get("USER")` or add CLI argument parsing for input/output directories.

### [MEDIUM] DNN hardcodes `mass=400, spin=2` in `DNN_application.py`
**File:** `Analysis/DNN_application.py`, line 100
**Problem:** `inputs = convert_to_numpy(valid_array, self.period, 400, 2)` always passes `mass=400` and `spin=2` regardless of the signal hypothesis being processed.
**Impact:** If the model is ever updated to use mass/spin conditioning, all non-SM signal hypotheses would be scored with wrong conditioning inputs. Currently harmless since the model ignores them, but a latent correctness hazard.
**Fix:** Pass the actual process mass and spin from the dataset configuration.

### [LOW] `GetWeight()` has a second commented-out trigger weight block
**File:** `Analysis/hh_bbtautau.py`, line 278
**Problem:** A second `weights_to_apply.extend(trg_weights_dict[channel])` is also commented out, creating two separate stale code paths for trigger SFs.
**Impact:** Maintenance confusion — unclear which of the two disabled paths was the intended one.
**Fix:** Remove the duplicate commented-out block once the trigger SF mechanism is resolved.

---

## Corrections/

### [CRITICAL] Enum value collision in `tau.h` — `stat1_dm1` and `stat2_dm0` share value 7
**File:** `Corrections/tau.h`, lines 36–37
**Problem:** The `UncSource` enum has:
```cpp
stat2_dm0 = 7,
stat1_dm1 = 7,
```
Both enumerators have the same integer value 7. Since the uncertainty map uses `std::pair<UncSource, UncScale>` as a key, `stat1_dm1` and `stat2_dm0` are indistinguishable — they map to the same key. Any lookup for `stat1_dm1` will silently retrieve the `stat2_dm0` entry (or vice versa depending on insertion order).
**Impact:** The DM1 stat uncertainty bin 1 and the DM0 stat uncertainty bin 2 are swapped in all tau ID SF calculations. This is a physics correctness bug affecting per-decay-mode systematic variations, which are used in the final statistical fit.
**Fix:** Assign unique values to all enum members. The next value after the existing sequence should be 8 for `stat1_dm1`.

### [HIGH] 2025 tau POG folder is an empty string
**File:** `Corrections/CorrectionsCore.py`, line 66
**Problem:** `pog_folder_names["TAU"]["2025_Summer24"] = ""`. Any attempt to initialize tau corrections for 2025 data will construct an empty file path, causing a file-not-found error or silently loading no corrections.
**Impact:** All tau ID SFs and tau energy scales are broken for 2025 data.
**Fix:** Either point to the correct path when available, or add a runtime guard that raises a clear `NotImplementedError("Tau corrections not yet available for 2025")`.

### [HIGH] 2025 electron ID and trigger SF files are empty strings
**File:** `Corrections/CorrectionsCore.py`, line 96 area
**Problem:** `ele_files_names["2025_Summer24"]` has empty strings for `eleID`, `eleHLT`, and `eleID_highPt`. Constructing electron correction providers for 2025 will silently fail or crash.
**Impact:** Electron ID SFs and trigger SFs are unavailable for 2025 data — uncorrected event weights.
**Fix:** Add a guard similar to the tau case, or populate with correct paths when they become available.

### [HIGH] `Run3_2026` mapped to 2025 conditions as a temporary patch
**File:** `Corrections/CorrectionsCore.py`, line 112
**Problem:** `period_names["Run3_2026"] = "2025_Summer24"` is labeled `# TEMPORARY PATCH TO CHECK THAT WORKS FOR 2026 DATA`. This means 2026 data uses 2025 corrections — wrong luminosity, wrong PU profile, wrong energy scales.
**Impact:** Any analysis run on 2026 data will use incorrect correction files unless this is updated. It is a research blocker that can be silently missed.
**Fix:** Remove the patch once real 2026 conditions are available. Add an explicit `raise` if 2026 data is processed in production before real corrections exist.

### [HIGH] Tau trigger SFs for 2024 and 2025 use 2023postBPix as placeholder
**File:** `Corrections/triggersRun3.py`, lines 77–79
**Problem:** `tau_filename_dict["2024_Summer24"] = "2023_postBPix"` and similarly for 2025. All three future eras use 2023 post-BPix tau trigger SFs.
**Impact:** Trigger SFs for 2024 and 2025 data are systematically wrong — they use corrections derived from different detector conditions.
**Fix:** Obtain and register actual 2024/2025 tau trigger SF files. Add a warning at initialization time so analysts are aware of the substitution.

### [HIGH] FatJet corrections missing for 2024 and 2025
**File:** `Corrections/fatjet.py`, `fatjet_corr_dict`
**Problem:** The dictionary only contains entries for 2022 preEE/postEE and 2023 preBPix/postBPix. Accessing this dict for 2024 or 2025 raises a `KeyError`.
**Impact:** Any analysis enabling FatJet corrections for Run3_2024+ will crash at correction initialization.
**Fix:** Either add entries for 2024/2025 when available, or add a guard: `if period not in fatjet_corr_dict: raise NotImplementedError(...)`.

### [HIGH] 2025 JERC uses Winter25 folder under a Summer24 key — naming mismatch
**File:** `Corrections/CorrectionsCore.py`, lines 20 and 36
**Problem:** `pog_folder_names["JERC"]["2025_Summer24"]` points to `Run3-25Prompt-Winter25-NanoAODv15/...` even though the key is `Summer24`. The comment acknowledges this is a `# TMP PATCH` because the JME group only published Winter25 files.
**Impact:** The mismatch between the key name (`Summer24`) and actual folder (`Winter25`) will confuse developers trying to update or validate which corrections are applied for 2025.
**Fix:** Either rename the key to `2025_Winter25` and update all references, or leave the patch with a clearer comment and a TODO tracking the correct key.

### [HIGH] Singleton initialization pattern silently skips re-initialization
**File:** `Corrections/*.py` (all providers with `initialized = False` class variable)
**Problem:** Each correction provider uses a class-level `initialized = False` flag. If two analyses with different configurations (e.g., different periods) are initialized within the same Python session (possible in multi-era processing), the second `Initialize()` call for any provider is silently skipped.
**Impact:** The second era's corrections are evaluated using the first era's configuration — wrong JSON files, wrong WPs, wrong period labels. This is especially dangerous when running batch jobs that reuse the Python process across datasets.
**Fix:** Either raise an error on attempted re-initialization with different parameters, or switch to instance-level state rather than class-level state.

### [MEDIUM] `btagShapeWeightCorrector` branch name parsing fragility
**File:** `Corrections/btag.py` (btagShapeWeightCorrector)
**Problem:** The syst name is extracted as `b.split("_")[2]` from branch names like `weight_bTagShape_{syst}_rel`. This assumes the syst name has no underscores. A syst name containing an underscore (e.g., `lfstats_1`) would return only the first token, silently mis-mapping the systematic.
**Impact:** If any future btag systematic name contains an underscore, the normalization correction would be applied to the wrong systematic branch.
**Fix:** Use a more robust parsing approach: strip the known prefix `weight_bTagShape_` and suffix `_rel` from the branch name instead of relying on positional splitting.

### [MEDIUM] PUJetID uses a deprecated/non-canonical cvmfs path
**File:** `Corrections/puJetID.py`, line 11
**Problem:** `PUJetID_JsonPath = "/cvmfs/cms.cern.ch/rsync/cms-nanoAOD/jsonpog-integration/POG/JME/{}/jmar.json.gz"`. All other corrections use `cms-griddata.cern.ch/cat/metadata/` as the canonical path. The `rsync` path is older and may not be kept up to date.
**Impact:** Could access stale or outdated PUJetID efficiency maps if the `rsync` mirror lags behind `cms-griddata.cern.ch`.
**Fix:** Update to `"/cvmfs/cms-griddata.cern.ch/cat/metadata/JME/{}/jmar.json.gz"` to match all other corrections.

---

## Config/

### [CRITICAL] `Run3_2024` missing from all trigger threshold dictionaries in `global.yaml`
**File:** `config/global.yaml`, lines 267–380
**Problem:** The `singleMu_th`, `singleEle_th`, `singleTau_th`, `eTau_th`, `muTau_th`, and `uncs_to_exclude` dictionaries all enumerate periods up to `Run3_2023BPix` but have no `Run3_2024` entry.
**Impact:** Processing `Run3_2024` data/MC will either raise a `KeyError` when the config is accessed, or silently fall through to an undefined value, giving wrong pT thresholds for trigger preselection — a fundamental physics selection error.
**Fix:** Add `Run3_2024` entries to all threshold dictionaries. The `btag` correction in `Run3_2024/global.yaml` already accounts for the new tagger, so the thresholds should follow.

### [HIGH] `ditaujet` trigger filter bit uses bitwise `&` instead of logical `&&`
**File:** `config/Run3_2022EE/triggers.yaml`, line 54
**Problem:** The filter bit selection string is:
```
TrigObj_id==15 && (TrigObj_filterBits&8)!=0 && (TrigObj_filterBits&2048)!=0 & (TrigObj_filterBits&16384)!=0
```
The last operator is a single `&` (bitwise AND of the integer result of `(TrigObj_filterBits&2048)!=0`) instead of `&&` (logical AND). The boolean result of `!=0` is `0` or `1`, so `& (TrigObj_filterBits&16384)!=0` performs a bitwise AND of 0/1 with the integer `(TrigObj_filterBits & 16384)`, not the intended logical AND. All other eras and all other trigger lines use `&&`.
**Impact:** The ditaujet trigger matching condition is evaluated incorrectly for Run3_2022EE data. Events that should fail the filter bit check may pass (or fail) incorrectly, biasing trigger efficiency measurements.
**Fix:** Change the single `&` to `&&`.

### [HIGH] `signal_types` in `global.yaml` only lists `HHnonRes` — resonant signals not processable
**File:** `config/global.yaml`, `signal_types` section
**Problem:** Only `HHnonRes` is active. `GluGluToRadion`, `GluGluToBulkGraviton`, `VBFToRadion`, `VBFToBulkGraviton` are all commented out. The era configs (`datasets.yaml`, `processes.yaml`) fully define resonant signal samples.
**Impact:** Running the pipeline in its current state will not process resonant signals even though the datasets are configured. The resonant analysis cannot run without modifying `global.yaml`.
**Fix:** Uncomment the resonant `signal_types` entries, or provide documentation/comments explaining that they must be enabled for a resonant analysis run.

### [HIGH] No plot YAML for `Run3_2024`
**File:** `config/plot/` directory
**Problem:** Plot configuration YAML files exist for Run2_2016, Run2_2016_HIPM, Run2_2017, Run2_2018, Run2_all, Run3_2022, Run3_2022EE, Run3_2023, Run3_2023BPix — but nothing for `Run3_2024`.
**Impact:** Any attempt to produce stack plots for Run3_2024 data will fail when the plotting code tries to load the era-specific luminosity text and channel labels.
**Fix:** Create `config/plot/Run3_2024.yaml` with the appropriate `lumi_text`, `channel_text`, and `customregion_text` entries.

### [MEDIUM] `DY_M_10to50` missing `name` field in `Run3_2022/processes.yaml`
**File:** `config/Run3_2022/processes.yaml`
**Problem:** The `DY_M_10to50` sub-process entry lacks a `name` field. All other era files (Run3_2022EE, 2023, 2023BPix, 2024) define `name: "DY M10to50"`.
**Impact:** The process may be displayed without a label in plots, or the code processing the `name` field may raise a `KeyError` for this era.
**Fix:** Add `name: "DY M10to50"` to the corresponding entry in `config/Run3_2022/processes.yaml`.

### [MEDIUM] Non-resonant EFT coupling points inconsistently commented out in `Run3_2022EE`
**File:** `config/Run3_2022EE/processes.yaml`
**Problem:** Several ggF HH coupling points active in `Run3_2022/processes.yaml` (`kl_0p00_c2_1p00`, `kl_1p00_c2_0p10/0p35/3p00/m2p00`, `kl_2p45`) are commented out in `Run3_2022EE` but not in 2023/2023BPix.
**Impact:** The EFT scan coverage is inconsistent across eras, which could produce biased combined limits if 2022EE contributes different coupling points than 2022 pre-EE or 2023.
**Fix:** Ensure consistent EFT coverage across all Run3 eras, or add explanatory comments for why specific points are excluded from 2022EE.

### [MEDIUM] `GluGluHto2Tau_M125` missing from `Run3_2022EE/processes.yaml` ggH group
**File:** `config/Run3_2022EE/processes.yaml`
**Problem:** `GluGluHto2Tau_M125` is present in `Run3_2022/processes.yaml` under the `ggH` process group but absent from `Run3_2022EE` with no explanatory comment.
**Impact:** Single Higgs ggH→ττ background is missing from Run3_2022EE histograms, underestimating the total signal background model.
**Fix:** Verify whether the dataset is available for 2022EE and add it if so. If deliberately excluded, add a comment.

### [MEDIUM] `custom_CI_Signal` absent from `Run3_2024/processes.yaml`
**File:** `config/Run3_2024/processes.yaml`
**Problem:** CI testing requires a `custom_CI_Signal` process entry. The 2024 file is noted to lack one.
**Impact:** Continuous integration tests cannot be run for the Run3_2024 configuration, meaning code changes could break 2024 processing without detection.
**Fix:** Add a `custom_CI_Signal` process entry pointing to a small test sample, or mock one for CI purposes.

### [MEDIUM] `eTau_th` and `muTau_th` are null for all periods
**File:** `config/global.yaml`, cross-trigger threshold sections
**Problem:** Both `eTau_th` and `muTau_th` are defined as `null` for all eras despite cross-trigger entries (`etau`, `mutau`) being active in the trigger YAML files.
**Impact:** The cross-trigger offline pT threshold (the minimum pT for the lepton leg in a cross-trigger) is not configured. Depending on how `null` is handled in the selection code, this either means no pT cut is applied (too permissive) or a crash occurs when the threshold is accessed.
**Fix:** Set the correct per-era pT thresholds for cross-triggers, or document that cross-triggers use only the single-lepton threshold.

### [LOW] Run2 directories missing `datasets.yaml`, `processes.yaml`, `global.yaml`, `triggers.yaml`
**File:** `config/Run2_2016/`, `config/Run2_2016_HIPM/`, `config/Run2_2017/`, `config/Run2_2018/`
**Problem:** Run2 era directories contain only `weights.yaml`. All other configuration files that exist for Run3 eras are absent.
**Impact:** Run2 data cannot be processed using the per-era config resolution mechanism. Developers must reconstruct the required configuration manually, which is error-prone.
**Fix:** Add the missing YAML files for Run2 eras, even if they mirror the global defaults with only the era-specific overrides.

### [LOW] Identical `weights.yaml` across all Run3 periods
**File:** `config/Run3_2022/weights.yaml`, `config/Run3_2022EE/weights.yaml`, `config/Run3_2023/weights.yaml`, `config/Run3_2023BPix/weights.yaml`
**Problem:** All four files are byte-for-byte identical. The separate files presumably exist to allow per-era customization but none has been applied.
**Impact:** Low maintenance burden but potential confusion — a developer editing one file may not realize they need to update all four.
**Fix:** Consider extracting a single shared Run3 `weights.yaml` in `config/` and only having per-era overrides, or at minimum add a comment stating the files are intentionally identical.

### [LOW] `customregion_text` missing from older plot YAMLs
**File:** `config/plot/Run2_2016.yaml`, `config/plot/Run2_2016_HIPM.yaml`, `config/plot/Run2_2017.yaml`, `config/plot/Run2_2018.yaml`, `config/plot/Run2_all.yaml`, `config/plot/Run3_2022.yaml`
**Problem:** Run3_2022EE and Run3_2023BPix define `customregion_text` for QCD control regions, but the listed files do not.
**Impact:** Any code that accesses `customregion_text` for these eras will raise a `KeyError`.
**Fix:** Add `customregion_text` entries to all plot YAML files, or make the code that accesses this key use `.get("customregion_text", "")` with a safe default.

---

## FLAF/

### [HIGH] `HistFromNtupleProducerTask.requires` appends a generator instead of list elements
**File:** `FLAF/Analysis/tasks.py`, lines 453–462
**Problem:**
```python
reqs.append(
    HistTupleProducerTask.req(...)
    for prod_br in prod_br_list
)
```
`reqs.append(generator)` appends a single generator object to the list. LAW iterates task requirements as a flat list of task instances. A generator object is not a task instance.
**Impact:** The histogram producer task's requirements are silently broken — the upstream `HistTupleProducerTask` is never seen as a dependency. This causes the workflow to either fail with a confusing error or run without its dependencies being completed.
**Fix:**
```python
reqs.extend(
    HistTupleProducerTask.req(...)
    for prod_br in prod_br_list
)
```

### [HIGH] `CrossSectionDB.evaluateExpression` uses `eval()` on config content
**File:** `FLAF/Common/CrossSectionDB.py`, line 66
**Problem:** `result = eval(expr, {}, self.values)` evaluates cross-section expressions from YAML files using Python `eval`. While the `{}` global namespace limits some risks, the local namespace `self.values` is exposed.
**Impact:** Malformed or malicious YAML cross-section entries could execute arbitrary Python expressions. Even without malicious intent, a typo in a cross-section expression can cause unexpected behavior that is hard to debug.
**Fix:** Use a safer expression parser (e.g., `ast.literal_eval` for simple cases, or `simpleeval` library), or restrict to only arithmetic operations with a whitelist.

### [MEDIUM] `EnableImplicitMT` is commented out — single-threaded only
**File:** `FLAF/AnaProd/anaTupleProducer.py`, line 8
**Problem:** `# ROOT.EnableImplicitMT(1)` is commented out. `EnableThreadSafety()` is called but multi-threaded RDataFrame is disabled.
**Impact:** Significant performance loss — all RDataFrame processing runs single-threaded on multi-core HTCondor worker nodes, wasting available CPU resources.
**Fix:** Uncomment and test `ROOT.EnableImplicitMT()` (or with an appropriate thread count). Note: enabling MT requires thread-safety audits of all `gInterpreter.Declare` headers and Python callbacks.

### [MEDIUM] `type(x) == str` / `type(x) == list` / `type(x) == dict` comparisons instead of `isinstance`
**File:** `FLAF/Common/Setup.py`, lines 225, 376, 405; `FLAF/Common/Utilities.py` (similar)
**Problem:** Explicit `type()` equality checks are used instead of `isinstance()`. This breaks for subclasses of `str`, `list`, or `dict` (e.g., `OrderedDict`, YAML-loaded types).
**Impact:** Fragile type dispatching that may silently fail for derived types returned by YAML loaders or custom config objects.
**Fix:** Replace `type(x) == str` with `isinstance(x, str)` throughout.

### [MEDIUM] `delete_after_merge` hardcoded to `False`
**File:** `FLAF/Analysis/tasks.py`, line 578
**Problem:** `delete_after_merge = False  # var == self.global_config["variables"][-1] --> find more robust condition`. The comment indicates this is an incomplete implementation of cleanup logic for intermediate HistTuple files.
**Impact:** Intermediate per-variable histogram files are never deleted after merging, accumulating disk usage over multiple runs. On EOS/CERNBOX this can exhaust quota.
**Fix:** Implement the cleanup condition or expose it as a task parameter.

### [LOW] `HWWCand` constructor calls `leg_charge.resize(num_legs)` twice
**File:** `FLAF/include/HHCore.h`, lines 118–119
**Problem:** `leg_charge.resize(num_legs)` is called on consecutive lines in the `HWWCand` constructor.
**Impact:** Harmless (second resize is a no-op since the size is already correct), but clearly a copy-paste error from the similar `leg_type.resize()` / `leg_p4.resize()` calls, and `leg_rawIso` or another field is likely missing its resize call.
**Fix:** Audit all fields resized in the constructor and add/remove resize calls as needed.

### [LOW] `HWWCand::_channel` private method is dead code
**File:** `FLAF/include/HHCore.h`, line 177
**Problem:** `HWWCand` has a private `_channel()` method using `std::index_sequence` that is defined but never called — `channel()` uses a manual `switch` statement instead.
**Impact:** Dead code adds maintenance confusion.
**Fix:** Remove the unused `_channel()` method.

### [LOW] `Setup._global_instances` caches by repr of customisations tuple
**File:** `FLAF/Common/Setup.py`
**Problem:** `Setup.getGlobal(...)` caches instances by an 8-tuple key that includes `repr(customisations)`. If `customisations` is a list vs. a tuple, the repr differs, silently creating separate `Setup` instances for semantically identical configurations.
**Impact:** Unnecessary memory usage and potential configuration divergence in multi-era batch processing.
**Fix:** Normalize `customisations` to a canonical form (e.g., sorted tuple of strings) before using it as a cache key.

---

## include/ (C++ Headers)

### [CRITICAL] `kin_fit::FitResults` is defined in both `KinFitInterface.h` and `KinFitNamespace.h` — ODR risk
**File:** `include/KinFitInterface.h` and `include/KinFitNamespace.h`
**Problem:** Both files independently define `struct kin_fit::FitResults` with identical bodies. `KinFitInterface.h` does not include `KinFitNamespace.h`. If both headers are loaded in the same ROOT interpreter session (which is possible since study scripts load `KinFitNamespace.h` and the production code loads `KinFitInterface.h`), there are two definitions of the same struct — an ODR (One Definition Rule) violation.
**Impact:** In practice the definitions are currently identical, so the violation is latent. Any future change to one struct that is not mirrored in the other will cause a silent inconsistency or a hard-to-diagnose crash.
**Fix:** Remove the `FitResults` definition from `KinFitInterface.h` and have it include `KinFitNamespace.h` instead:
```cpp
#include "KinFitNamespace.h"
```

### [HIGH] `KinFitNamespace.h` uses `std::numeric_limits` without `#include <limits>`
**File:** `include/KinFitNamespace.h`
**Problem:** `KinFitNamespace.h` uses `std::numeric_limits<int>::lowest()` in the `FitResults` constructor default but includes no standard headers. It relies on `<limits>` having been pulled in transitively by whatever preceded it.
**Impact:** If `KinFitNamespace.h` is the first or only header in a translation unit (e.g., in a new study script), the compilation will fail with `error: 'numeric_limits' is not a member of 'std'`.
**Fix:** Add `#include <limits>` to `KinFitNamespace.h`.

### [HIGH] Silent exception swallowing in `KinFitInterface.h::FitImpl`
**File:** `include/KinFitInterface.h`, `FitImpl` function
**Problem:**
```cpp
} catch (std::exception&) {
}
```
All exceptions from `HHKinFit2` (numerical failures, covariance matrix singularity, etc.) are silently swallowed, returning a default `FitResults` with `convergence = INT_MIN`.
**Impact:** Systematic fit failures produce no diagnostic output whatsoever. Events with consistently failing fits (e.g., due to wrong input conventions or MC/data differences) are silently assigned `convergence = INT_MIN` and excluded from analysis — with no way to identify the cause or rate.
**Fix:** At minimum, add a counter or a debug-mode `std::cerr` output:
```cpp
} catch (std::exception& e) {
    if (verbosity > 0) std::cerr << "KinFit exception: " << e.what() << std::endl;
}
```

### [MEDIUM] `kinFit_result` struct column causes ROOT snapshot serialization issues
**File:** `include/KinFitInterface.h`; `AnaProd/LegacyVariables.py`
**Problem:** `GetKinFit` includes `kinFit_result` (the raw `kin_fit::FitResults` struct) in the returned column list. Non-primitive struct columns require ROOT dictionary registration for `TTree` snapshot. The study scripts have a commented-out workaround: `# if "kinFit_result" in col_names_cache: col_names_cache.remove("kinFit_result")`.
**Impact:** ROOT file snapshots including `kinFit_result` may fail to serialize correctly or produce unreadable branches, depending on whether the dictionary is registered.
**Fix:** Remove `kinFit_result` from the snapshot column list entirely, since all scalar derived columns (`kinFit_convergence`, `kinFit_m`, `kinFit_chi2`) are already returned separately.

---

## HHbtag/

### [MEDIUM] `GetHHBtagScore` parameter is named `Jet_deepFlavour` but receives `Jet_btagPNetB`
**File:** `FLAF/include/HHbTagScores.h`, line 121; `AnaProd/baseline.py`, line 263
**Problem:** The C++ function signature declares the b-tagger score argument as `const RVecF &Jet_deepFlavour`, but the call site in `baseline.py` passes `Jet_btagPNetB` (ParticleNet B score). For v3, ParticleNet is the correct input, but the parameter name is a holdover from v1/v2 where DeepFlavour was used.
**Impact:** The code is functionally correct for v3 (the correct values are passed), but the misleading parameter name makes code review difficult and could cause confusion when switching between model versions.
**Fix:** Rename the parameter to `Jet_btagScore` in the C++ function signature, or add a version-conditional rename.

### [MEDIUM] HHbtag model version hardcoded to 3
**File:** `AnaProd/anaTupleDef.py`, `Initialize()` function
**Problem:** `HHBtagWrapper::Initialize(path, 3)` hardcodes version 3 with no config-file-driven override.
**Impact:** Switching to a different model version (e.g., a v4 trained on Run3 2023BPix conditions) requires a code edit rather than a config change.
**Fix:** Read the HHbtag version from the global config YAML and pass it to `Initialize()`.

### [LOW] Run2 periods map to era_id=0 (2022preEE) when using v3 model
**File:** `FLAF/include/HHbTagScores.h`, `PeriodToHHbTagInput` mapping
**Problem:** All Run2 periods are mapped to era_id=0 (2022preEE) for the v3 model since v3 was trained only on Run3 data.
**Impact:** HHbtag scores for Run2 are computed with a mismatched era input. The model may produce suboptimal discrimination for Run2 samples. This is documented behavior but should be prominently flagged.
**Fix:** Add an explicit comment or warning at initialization if v3 is used on Run2 data, and investigate whether a v2 model should be used for Run2 processing.

---

## HHKinFit2/

### [HIGH] Memory leak in `HHKinFitMasterHeavyHiggs::fit()` on exception path
**File:** `HHKinFit2/src/HHKinFitMasterHeavyHiggs.cpp`
**Problem:** All fit objects (tau1Fit, tau2Fit, b1Fit, b2Fit, metFit, heavyHiggs, higgs1, higgs2, plus constraints) are allocated with `new` inside the hypothesis loop. The cleanup `delete` statements are at the end of the loop body. However, `continue` statements appear before the cleanup block (on lines ~316, ~340, ~457, ~507, ~558), bypassing the `delete` calls.
**Impact:** Every event where the fit hits an early-exit `continue` leaks all allocated fit objects. In a high-statistics analysis (O(10^6) events), this can cause significant memory growth, potentially crashing jobs.
**Fix:** Use RAII wrappers (`std::unique_ptr`) for all fit objects, or use a `goto cleanup` pattern to ensure deallocation is always reached.

### [HIGH] `gRandom` global overwrite without deleting old object
**File:** `HHKinFit2/src/HHKinFitMasterHeavyHiggs.cpp`, lines 197 and 1013
**Problem:** In the `istruth=true` branch: `gRandom = new TRandom3(0)` overwrites ROOT's global random generator without `delete`-ing the old `gRandom`. This leaks the old object and may corrupt other code relying on `gRandom` (e.g., any other ROOT random number usage in the same process).
**Impact:** Memory leak and potential interference with ROOT's global random state in any analysis that uses `gRandom` elsewhere.
**Fix:** Save and restore `gRandom`, or use `delete gRandom; gRandom = new TRandom3(0);`, or preferably use a local `TRandom3` instance instead of modifying the global.

### [MEDIUM] "ALARM" debug message left in production code
**File:** `HHKinFit2/src/HHKinFitMasterHeavyHiggs.cpp`, line 776
**Problem:** `std::cout << "--------ALARM! ALARM! SOMETHING HAS GONE HORRIBLY WRONG!-------"` is present in what appears to be a fallthrough/error case.
**Impact:** This message will appear in production job logs whenever the alarm condition is triggered, polluting log files and potentially causing false alarms in log monitoring systems.
**Fix:** Replace with a properly formatted error message or a `throw std::runtime_error(...)`.

### [MEDIUM] Tau visible mass forced to tau pole mass — incorrect for hadronic decays
**File:** `HHKinFit2/src/HHKinFitMasterHeavyHiggs.cpp`, constructor
**Problem:** The constructor calls `SetMkeepE` to forcibly reset both tau visible masses to the tau lepton pole mass (1.77682 GeV), overwriting the input `TLorentzVector` mass. For hadronic tau decays, the visible system mass is the hadron system mass (pion/rho/a1 mass), not the tau lepton mass.
**Impact:** This is a known approximation in the kinematic fit, but it introduces a systematic bias in the Higgs mass constraint. The visible tau mass affects the kinematic constraint solving, so this overwrite may degrade the di-Higgs mass resolution.
**Fix:** Document this approximation prominently at the call site (`KinFitInterface.h`) and consider using decay-mode-specific visible masses as an improvement.

### [MEDIUM] Hardcoded b-jet resolution parametrization — Run2-era constants
**File:** `HHKinFit2/src/HHKinFitMasterHeavyHiggs.cpp`, `GetPFBJetRes` function
**Problem:** The b-jet energy resolution is computed from a hardcoded eta/Et-binned parametrization that was derived from an older CMS analysis era. No resolution is returned (falls back to `de=10`) for `|eta| >= 2.5`.
**Impact:** For Run3 data with updated detector conditions, the hardcoded Run2-era resolution parametrization may be systematically incorrect. Forward jets (`|eta| >= 2.5`) get a fixed fallback resolution.
**Fix:** Either pass the JER-table-derived resolution from the analysis framework (which is already done when external `sigmaEbjet` arguments are provided), or update the parametrization to Run3 JER conditions.

---

## StatInference/

### [CRITICAL] Hardcoded path to another user's AFS area in config
**File:** `StatInference/config/x_hh_bbtautau_run2.yaml`, line 7
**Problem:** `hist_bins: /afs/cern.ch/work/v/vdamante/FLAF/StatInference/config/hhbbtautau_binning.json` — an absolute path to user `vdamante`'s personal AFS workspace.
**Impact:** Running datacard production from any account other than `vdamante`'s, or after their AFS space expires, will fail with a `FileNotFoundError`. This is a blocked pipeline dependency.
**Fix:** Replace with a relative path or an environment-variable-based path:
```yaml
hist_bins: ${ANALYSIS_PATH}/StatInference/config/hhbbtautau_binning.json
```

### [HIGH] Era naming mismatch for `QCD_norm` uncertainties
**File:** `StatInference/config/x_hh_bbtautau_run2.yaml`
**Problem:** `QCD_norm` uncertainty entries specify `eras: Run2_2018` (no underscore between "Run2" and "_2018" — actually it uses the string `Run2_2018` which on closer inspection is correct syntax, but must be checked against the actual era key used in the top-level config). The top-level era list in the same file uses the same `Run2_2018` key. The issue is whether the `eras` filter in `DatacardMaker` matches the era key exactly. Any subtle discrepancy (trailing space, case difference) would cause the uncertainty to be silently skipped.
**Impact:** QCD normalization uncertainties may not be applied to any era's datacards, underestimating the systematic uncertainties on the QCD background.
**Fix:** Add a validation step in `DatacardMaker` that warns when a declared era in an uncertainty entry does not match any configured era.

### [HIGH] `process.scale` uses `eval()` on YAML config content
**File:** `StatInference/dc_make/process.py`, line 69
**Problem:** `scale = eval(scale)` evaluates a string read directly from a YAML configuration file.
**Impact:** A malformed or malicious YAML entry could execute arbitrary Python code during datacard creation. Even without security concerns, `eval` errors produce confusing stack traces that obscure the actual YAML problem.
**Fix:** Parse arithmetic expressions with `ast.literal_eval` for simple numeric literals, or use the `simpleeval` library for expressions. At minimum add input validation.

### [HIGH] `optimize_binning.py` raises `NameError` for non-standard `num_original_bins`
**File:** `StatInference/bin_opt/optimize_binning.py`, lines 21–31
**Problem:** `min_step`, `step_int_scale`, and `max_value_int` are only defined inside two `if` branches checking `num_original_bins == 1000` or `== 5000`. No `else` branch or default assignment exists. If `bin_optimization.yaml` specifies any other value, subsequent code on line 35 (`x = np.zeros(max_value_int)`) raises `NameError: name 'max_value_int' is not defined`.
**Impact:** The binning optimization is completely non-functional for any bin count other than 1000 or 5000, with a cryptic NameError as the symptom.
**Fix:** Add an `else` clause that either raises a clear `ValueError("Unsupported num_original_bins value: {}".format(...))` or computes the correct step size generically.

### [HIGH] `law/tasks.py` imports `dhi.tasks.resonant` at module load time
**File:** `StatInference/law/tasks.py`, line 12
**Problem:** `from dhi.tasks.resonant import MergeResonantLimits` is a top-level import. If the `dhi` package (the `inference/dhi` submodule) is not installed or not on `PYTHONPATH`, this import fails, making the entire `tasks.py` module unusable — including `CreateDatacardsTask` which has no dependency on `dhi`.
**Impact:** A developer who has not set up the inference environment cannot use any LAW task from this file, including the basic datacard creation task.
**Fix:** Move the `dhi` import inside `ResonantLimitsTask` class body or use a lazy import:
```python
def run(self):
    from dhi.tasks.resonant import MergeResonantLimits
    ...
```

### [MEDIUM] `optimize_channel.py` potential `NameError` for `other_cat_file`
**File:** `StatInference/bin_opt/optimize_channel.py`, around line 292–295
**Problem:** `other_cat_file` is assigned only inside `for file in os.listdir(best_dir):` when a matching file is found. After the loop, `if not os.path.isfile(other_cat_file):` references it. If `os.listdir(best_dir)` finds no matching file, `other_cat_file` is never assigned, and Python raises `NameError` — or worse, uses a stale value from a previous outer loop iteration.
**Impact:** Multi-category sequential optimization crashes with `NameError` or silently uses the wrong file from the previous category's best result.
**Fix:** Initialize `other_cat_file = None` before the loop and check for `None` after:
```python
other_cat_file = None
for file in os.listdir(best_dir):
    if ...: other_cat_file = f"{best_dir}/{file}"
if other_cat_file is None or not os.path.isfile(other_cat_file):
    raise RuntimeError(...)
```

### [MEDIUM] `computeLimits.py` uses legacy datacard naming scheme
**File:** `StatInference/bin_opt/computeLimits.py`
**Problem:** This script hardcodes `categories = ['res2b', 'res1b', 'boosted', ...]` and `hh_{}_{}_13TeV.txt` as the datacard naming pattern. The current `create_datacards.py` produces `datacard_{process_name}.txt`. The naming schemes are incompatible.
**Impact:** Running `computeLimits.py` on current analysis outputs will fail to find any datacard files.
**Fix:** Update to use the current naming convention, or add a `--naming-scheme` CLI argument.

### [MEDIUM] `rebinAndRunLimits.py` silently ignores process/nuisance removal failures
**File:** `StatInference/bin_opt/rebinAndRunLimits.py`
**Problem:** The `remove_processes` and `remove_parameters` steps catch all exceptions and print warnings, allowing the optimization loop to continue with invalid datacards.
**Impact:** Zero-integral or negative-integral processes that should be removed remain in the datacard, causing `combine` to fail later with a confusing error, or — worse — producing numerically unstable limit results that are silently accepted by the optimizer.
**Fix:** Re-raise exceptions from process/nuisance removal, or add a validation step that checks for zero-integral processes before running combine.

---

## inference/ (dhi)

### [HIGH] `HHModelTask.__init__` raises unconditionally for `allow_empty_hh_model = False`
**File:** `inference/dhi/tasks/combine.py`, lines 87–90
**Problem:**
```python
if not self.allow_empty_hh_model:
    raise Exception(f"{self!r}: hh_model is not allowed to be empty")
else:
    print(f"{self!r}")
```
This raises an exception whenever `allow_empty_hh_model` is `False`, regardless of whether `self.hh_model` is actually empty. The intended guard should be `if not self.allow_empty_hh_model and not self.hh_model`. As written, any task subclass that sets `allow_empty_hh_model = False` (to enforce that a model must be provided) will always raise on construction.
**Impact:** Task subclasses designed to require an HH model cannot be instantiated at all. The `else: print(...)` branch (which should be the non-error path) is entered only when the model is allowed to be empty, which inverts the intended semantics.
**Fix:**
```python
if not self.allow_empty_hh_model and not self.hh_model:
    raise Exception(f"{self!r}: hh_model is not allowed to be empty")
```

### [HIGH] Malformed `raise Exception` in combine.py — tuple message instead of string
**File:** `inference/dhi/tasks/combine.py`, line 1213
**Problem:**
```python
raise Exception(f"Required POI ", self.pois, "not on choices list", DatacardTask.ALL_POIS)
```
`Exception(...)` with multiple positional arguments stores them as a tuple. The error display will be `('Required POI ', [...], 'not on choices list', [...])` rather than a readable message.
**Impact:** When this exception is raised, the error message is unreadable. Developers debugging POI configuration issues will see a raw tuple.
**Fix:**
```python
raise Exception(f"Required POI {self.pois} not on choices list {DatacardTask.ALL_POIS}")
```

### [MEDIUM] Debug `print(f"{self!r}")` in every `HHModelTask` instantiation
**File:** `inference/dhi/tasks/combine.py`, line 90
**Problem:** `print(f"{self!r}")` in the `else` branch of `HHModelTask.__init__` is executed every time any `HHModelTask` subclass is instantiated. Since the majority of `dhi` tasks inherit from `HHModelTask`, this prints a representation of every task to stdout.
**Impact:** Spams the console with task repr strings during workflow execution, making real output hard to find in logs.
**Fix:** Remove the `print` statement.

### [MEDIUM] Combine version mismatch between `.setups/flaf.sh` and `setup.sh`
**File:** `inference/.setups/flaf.sh` (DHI_COMBINE_VERSION=v10.4.2); `inference/setup.sh` (default v9.1.0)
**Problem:** The analysis-specific setup uses combine v10 while the framework default is v9. Any v10-specific API incompatibilities in workspace creation or limit commands will appear only at runtime, not at setup time.
**Impact:** If a developer uses `setup.sh` defaults instead of sourcing `.setups/flaf.sh`, they will run v9 combine on datacards designed for v10, potentially producing wrong results.
**Fix:** Document the required combine version prominently and add a version check at task startup.

### [MEDIUM] `inference/data/` is empty — no analysis outputs present
**File:** `inference/data/` directory
**Problem:** The `inference/data/` directory contains only a `.installed` marker. No datacards, workspaces, or limit results exist.
**Impact:** The inference pipeline cannot be run or validated from the current repository state. All inference tasks will fail on their `InputDatacards` dependency.
**Fix:** Document the expected file paths for datacards and provide instructions for generating them from upstream analysis outputs, or configure the `StatInference/law/tasks.py::CreateDatacardsTask` as the proper upstream dependency.

---

## Docs/

### [LOW] `docs/analysis.md` references `Common/SaveHisto.txt` — wrong file extension
**File:** `docs/analysis.md`, HHBTag training skim section
**Problem:** The documentation refers to `python Common/SaveHisto.txt`, which uses a `.txt` extension for a Python script. No such file was found.
**Impact:** Developers following the documentation to create HHbtag training skims will get a `FileNotFoundError` or `SyntaxError`.
**Fix:** Verify the correct script name and path; it should likely be `Common/SaveHisto.py`.

### [LOW] Interactive plot browser instructions reference generic username placeholder
**File:** `docs/interactive_plot_browser.md`
**Problem:** The documentation uses `/eos/user/u/username/` as the target path. Unlike `make_stackplots.py`, this is documentation (expected to be generic), but it does not clearly distinguish the literal `username` from a variable.
**Impact:** Minor confusion for new users; low severity.
**Fix:** Explicitly note that `username` should be replaced with the user's CERN account name, e.g., `username=$(id -un)`.

---

## Cross-Cutting Issues

### [CRITICAL] `FLAF/Common/LegacyVariables.py` does not exist — `addLegacyVariables.py` import will fail
**File:** `AnaProd/addLegacyVariables.py`, line 3; `AnaProd/LegacyVariables.py`
**Problem:** `addLegacyVariables.py` imports `FLAF.Common.LegacyVariables`. This module does not exist at `FLAF/Common/LegacyVariables.py`. The only `LegacyVariables.py` is at `AnaProd/LegacyVariables.py` (which itself has a broken relative import). If the import were attempted, it would raise `ModuleNotFoundError: No module named 'FLAF.Common.LegacyVariables'`.
**Impact:** The entire legacy variable computation pipeline (MT2, KinFit mass, SVfit mass) is broken and cannot be invoked through the expected module path. Since `applyLegacyVariables` is also never called, this issue is dormant — but it means the legacy variables are not being computed in the analysis at all.
**Fix:** Either create `FLAF/Common/LegacyVariables.py` as a wrapper or move `AnaProd/LegacyVariables.py` to the correct location, and fix the relative import.

### [HIGH] Hardcoded absolute paths scattered across multiple components
**Files and locations:**
- `AnaProd/interface.py` line 327: `/afs/cern.ch/work/a/acagnott/...`
- `StatInference/config/x_hh_bbtautau_run2.yaml` line 7: `/afs/cern.ch/work/v/vdamante/...`
- `Analysis/make_stackplots.py` lines 5–6: `/eos/user/u/username/...`

**Problem:** Multiple files contain absolute AFS or EOS paths belonging to specific users or containing placeholder `username` strings.
**Impact:** These paths will silently fail or cause `FileNotFoundError` for all other users, in CI, and after the original users' accounts expire. Analysis portability is broken.
**Fix:** Replace all absolute user-specific paths with environment variable references (`$ANALYSIS_PATH`, `$USER`, etc.) or relative paths. Add a pre-commit check that flags absolute `/afs/cern.ch/work/` or `/eos/user/` paths outside of configuration files with a designated `user_custom.yaml` override mechanism.

### [MEDIUM] No automated test coverage for physics-critical computation paths
**Problem:** The repository has no unit tests or integration tests for the core physics computation paths: b-tag SF calculation, trigger SF formula correctness, DNN inference output shape, kinematic fit convergence rates, QCD ABCD transfer factors, or datacard yield tables.
**Impact:** Regressions in any of the above (e.g., the `stat1_dm1`/`stat2_dm0` enum collision, the `mutau_tau_decayMode` copy-paste bug, the always-true muMu/eMu condition) will not be caught before producing analysis results.
**Fix:** Add pytest-based unit tests for at minimum: (1) tau ID SF lookup with known inputs, (2) trigger SF OR combination formula, (3) QCD ABCD yield transfer factor formula, (4) DNN input tensor construction.

### [MEDIUM] No `Run3_2024` entries in `Corrections/fatjet.py`, no `Run3_2024` plot YAML, and missing threshold dict entries — all point to incomplete Run3_2024 support
**Problem:** Run3_2024 support is partial across multiple components: trigger thresholds missing in `global.yaml`, no plot YAML, no FatJet correction entries, and `custom_CI_Signal` missing from `processes.yaml`. Together, these mean that while the datasets and histogram-level config exist for 2024, the analysis pipeline will fail at multiple points when actually run.
**Impact:** A 2024 analysis run will fail in at least four independent places. There is no single fix.
**Fix:** Conduct a systematic audit of all period-keyed dictionaries, YAML files, and code branches to ensure Run3_2024 is fully covered.

### [LOW] `run_inference_on_uncertainty_trees` function entirely commented out
**File:** `AnaProd/NNInterface.py`, lines 229–282
**Problem:** The function responsible for running DNN inference on systematic variation trees is entirely commented out, with no replacement.
**Impact:** DNN scores for systematic variation trees (JES Up/Down, TES Up/Down, etc.) cannot be computed via `NNInterface.py`. This means systematic uncertainty variations on DNN discriminants are not evaluated via this code path.
**Fix:** Either re-implement and uncomment `run_inference_on_uncertainty_trees`, or document that systematic variation DNN scores are handled by a different mechanism (e.g., `Analysis/DNN_application.py`).