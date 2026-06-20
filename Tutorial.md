# HH→bbττ Analysis Repository: Comprehensive Tutorial

## Table of Contents

1. [Overview](#overview)
2. [Repository Structure](#repository-structure)
3. [Getting Started](#getting-started)
4. [Workflow](#workflow)
5. [Component Reference](#component-reference)
   - [FLAF Framework](#flaf-framework)
   - [AnaProd: Ntuple Production](#anaprod-ntuple-production)
   - [Corrections](#corrections)
   - [Analysis: Histogramming](#analysis-histogramming)
   - [HHbtag](#hhbtag)
   - [HHKinFit2](#hhkinfit2)
   - [StatInference and Inference](#statinference-and-inference)
6. [Configuration System](#configuration-system)
7. [Interconnection Map](#interconnection-map)
8. [Known Issues and Caveats](#known-issues-and-caveats)

---

## Overview

This repository implements the CMS search for di-Higgs production in the HH→bbττ final state, where one Higgs boson decays to a bottom quark pair and the other to a pair of tau leptons. The analysis covers both resonant production (a massive BSM resonance X decaying to HH, with spin-0 Radion and spin-2 BulkGraviton hypotheses at masses from 250 to 5000 GeV) and non-resonant production (SM and EFT coupling variations parameterised by κ_λ, κ_t, c_2, C_2V, C_V). Data-taking periods covered include Run 2 (2016 HIPM, 2016, 2017, 2018) and Run 3 (2022, 2022EE, 2023, 2023BPix, 2024).

The di-tau final states considered are eTau, muTau, and tauTau (the three channels with significant signal acceptance), with eE, eMu, and muMu retained as control regions.

The analysis is built on top of **FLAF** (Flexible LAW-based Analysis Framework), a shared CMS analysis framework that provides task orchestration via [LAW](https://github.com/riga/law) (Luigi Analysis Workflow) and handles the common infrastructure for event selection, corrections, and histogramming. The full pipeline takes CMS NanoAOD inputs and produces datacards and statistical limits as output.

The hosted documentation lives at https://cms-flaf.github.io/HH_bbtautau/. This tutorial provides a deeper technical reference for collaborators working directly with the code.

---

## Repository Structure

```
HH_bbtautau/
├── env.sh                   # Environment bootstrap — source this first
├── mkdocs.yml               # MkDocs config for the hosted documentation site
├── README.md                # Minimal landing page pointing to hosted docs
├── HH_bbtautau.code-workspace  # VS Code workspace file
│
├── FLAF/                    # Core analysis framework (git submodule)
│   ├── include/             # C++ headers loaded at runtime via ROOT JIT
│   ├── Common/              # Shared Python modules (Setup, Utilities, BaselineSelection)
│   ├── AnaProd/             # NanoAOD → AnaTuple production tasks and logic
│   ├── Analysis/            # Histogram and cache production tasks
│   ├── Processors/          # Pluggable processors (e.g. MCStitching)
│   ├── RunKit/              # Grid/HTCondor job infrastructure (git submodule)
│   ├── PlotKit/             # Plotting stack (git submodule)
│   ├── config/              # FLAF-level YAML configs (shared across analyses)
│   ├── run_tools/           # Shell/Python helpers, law_customizations.py
│   └── env.sh               # FLAF environment setup (called by top-level env.sh)
│
├── AnaProd/                 # Analysis-specific ntuple production layer
│   ├── anaTupleDef.py       # Main column definitions, HHbtag initialization
│   ├── baseline.py          # Object preselections, candidate construction
│   ├── interface.py         # DNN model interface class (NNInterface)
│   ├── NNInterface.py       # Standalone DNN inference runner (legacy)
│   ├── LegacyVariables.py   # Wrappers for MT2, KinFit, SVfit (broken import)
│   └── addLegacyVariables.py  # Unused orchestration wrapper
│
├── Analysis/                # Histogram-level analysis
│   ├── hh_bbtautau.py       # Core analysis module: selections, weights, categories
│   ├── histTupleDef.py      # FLAF entry point for histogram tuple production
│   ├── GetCrossWeights.py   # Trigger scale factor computation
│   ├── interface.py         # DNN model interface (canonical version)
│   ├── DNN_application.py   # Production DNN inference wrapper (DNNProducer)
│   ├── QCD_estimation.py    # Data-driven QCD estimation (ABCD method)
│   └── make_stackplots.py   # Stack plot generation utility
│
├── Corrections/             # MC-to-data corrections library (git submodule)
│   ├── Corrections.py       # Central orchestrator
│   ├── CorrectionsCore.py   # Shared utilities and period name mappings
│   ├── corrections.h        # C++ base class (CorrectionsBase<T> CRTP singleton)
│   ├── tau.py / tau.h       # Tau energy scale and ID scale factors
│   ├── jet.py / jet.h       # JEC and JER for AK4 and AK8 jets
│   ├── btag.py / btag.h / btagShape.h  # b-tagging scale factors
│   ├── pu.py / pu.h         # Pileup reweighting
│   ├── mu.py / mu.h         # Muon ID/isolation scale factors
│   ├── MuonEnergyScale_corr.py / MuonScaReProvider.h  # Muon ScaRe corrections
│   ├── electron.py / electron.h  # Electron ID SFs and energy scale
│   ├── triggersRun3.py / triggersRun3.h  # Trigger SFs (Run 3)
│   ├── triggers.py / triggers.h  # Trigger SFs (Run 2)
│   ├── DY_hhbbtautau.py / DY_hhbbtautau.h  # DY corrections for this analysis
│   ├── Vpt.py / Vpt.h       # EWK V pT corrections
│   ├── met.py / met.h       # MET propagation of energy scale shifts
│   ├── lumi.py / lumi.h     # Golden JSON luminosity filter
│   ├── puJetID.py / puJetID.h  # Pileup jet ID efficiency
│   ├── JetVetoMap.py / JetVetoMap.h  # Jet veto maps for hot regions
│   └── data/                # Local correction data files (not on cvmfs)
│
├── config/                  # Analysis-level configuration
│   ├── global.yaml          # Master analysis config (639 lines)
│   ├── law.cfg              # LAW task framework config
│   ├── phys_models.yaml     # Physics models (processes per model)
│   ├── processes.yaml       # Global process definitions
│   ├── user_custom.yaml     # Per-user overrides (filesystem paths, variables)
│   ├── ci_custom.yaml       # CI override config
│   ├── Run2_2016/           # Era-specific config (weights.yaml only for Run 2)
│   ├── Run2_2016_HIPM/
│   ├── Run2_2017/
│   ├── Run2_2018/
│   ├── Run3_2022/           # Full config: global, triggers, weights, datasets, processes
│   ├── Run3_2022EE/
│   ├── Run3_2023/
│   ├── Run3_2023BPix/
│   ├── Run3_2024/
│   ├── plot/                # Plot styling and per-variable binning
│   └── nn_models/           # TensorFlow SavedModel files (5-fold MoE ensemble)
│       ├── model_fold0_moe/ through model_fold4_moe/
│       └── hbtres_*/        # Alternative hbtres model family (5 folds)
│
├── include/                 # Analysis-level C++ headers
│   ├── KinFitInterface.h    # Wrapper around HHKinFit2 (kin_fit::FitProducer)
│   ├── KinFitNamespace.h    # Lightweight kin_fit::FitResults declaration
│   └── SVfitAnaInterface.h  # Wrapper around ClassicSVfit (sv_fit::FitProducer)
│
├── HHbtag/                  # HH-specific b-tagger (git submodule)
│   ├── interface/HH_BTag.h  # C++ class declaration
│   ├── src/HH_BTag.cpp      # TensorFlow inference implementation
│   └── models/              # HHbtag_v3_par_0/ and HHbtag_v3_par_1/
│
├── HHKinFit2/               # Di-Higgs kinematic fitter (git submodule)
│   ├── interface/HHKinFitMasterHeavyHiggs.h  # Primary user API
│   └── src/                 # Fit implementation, PSMath optimizer
│
├── StatInference/           # Analysis-specific datacard production (git submodule)
│   ├── dc_make/             # CombineHarvester-based datacard maker
│   ├── bin_opt/             # Bayesian binning optimization
│   ├── fit_val/             # Fit validation diagnostics
│   ├── law/                 # LAW tasks for datacard production
│   ├── config/              # Analysis YAML and binning JSON
│   └── common/              # Shared ROOT/histogram utilities
│
├── inference/               # CMS HH inference framework / dhi (git submodule)
│   ├── dhi/                 # Core inference tasks and physics models
│   ├── setup.sh             # dhi environment setup
│   └── data/                # Output area for workspaces, limits, plots
│
├── Studies/                 # Ad-hoc studies and validation scripts
│   ├── HHBTag/              # HHbtag training skim and validation
│   ├── MassCuts/            # KinFit mass distribution studies
│   └── Triggers/            # Trigger efficiency studies
│
└── docs/                    # MkDocs documentation source
    ├── index.md             # Setup instructions
    ├── anatuple_prod.md     # AnaTuple production commands
    ├── analysis.md          # Post-production analysis commands
    ├── stat_inference.md    # Statistical inference commands
    └── interactive_plot_browser.md
```

---

## Getting Started

### Prerequisites

- Access to CERN AFS and LXPLUS (or compatible CentOS/AlmaLinux9 environment)
- CERN grid certificate and VOMS proxy
- Access to EOS or CERNBox for output storage
- CMS CMSSW environment (pinned to `CMSSW_16_0_6` with `gcc13`)

### 1. Clone the Repository

```bash
git clone --recursive https://github.com/cms-flaf/HH_bbtautau.git
cd HH_bbtautau
```

The `--recursive` flag is essential: it initialises all git submodules (FLAF, RunKit, PlotKit, Corrections, HHbtag, HHKinFit2, StatInference, inference).

### 2. Set Up Your User Configuration

Create `config/user_custom.yaml` with your personal storage paths and analysis preferences:

```yaml
# Filesystem locations for outputs (use EOS/CERNBox paths).
# Any fs_* you omit falls back to fs_default.
fs_default:
  path: /eos/user/x/yourusername/HH_bbtautau
fs_anaTuple:
  path: /eos/user/x/yourusername/HH_bbtautau/anaTuples
fs_anaCacheTuple:
  path: /eos/user/x/yourusername/HH_bbtautau/anaCacheTuples
fs_HistTuple:
  path: /eos/user/x/yourusername/HH_bbtautau/histTuples
fs_plots:
  path: /eos/user/x/yourusername/HH_bbtautau/plots

# Variables to include in histograms (key is `variables`)
variables:
  - tau1_pt
  - tau2_pt
  - b1_pt
  - b2_pt
  - met_pt
  - tautau_m_vis
  - bb_m_vis
  - ggF_DNN_HH

# Disable systematic variations for faster local testing
compute_unc_variations: false
compute_unc_histograms: false
```

### 3. Source the Environment

```bash
source env.sh
```

This script:
- Sets `ANALYSIS_PATH` to the repository root and `HH_INFERENCE_PATH` to `inference/`
- Pins `FLAF_CMSSW_VERSION=CMSSW_16_0_6`
- Calls `FLAF/env.sh` to set up the CMSSW and LCG environment
- Creates symlinks wiring HHbtag, ClassicSVfit, SVfitTF, and HHKinFit2 into the CMSSW working area

You must source this script on every new shell session before running any analysis tasks.

### 4. Initialise the VOMS Proxy

```bash
voms-proxy-init -voms cms -rfc -valid 192:00
```

A 192-hour proxy is recommended for long batch jobs.

### 5. Update the LAW Task Index

```bash
law index --verbose
```

This registers all available LAW tasks so they can be looked up by name.

### 6. Verify the Setup

```bash
law run InputFileTask --period Run3_2022EE --version dev --print-status 0
```

If this prints task status without errors, the environment is correctly configured.

---

## Workflow

The analysis proceeds through five major phases. LAW handles inter-task dependencies automatically when you run a downstream task; the steps are listed here explicitly so you understand what each stage produces.

### Phase 1: Build Input File Lists

```bash
law run InputFileTask --period ${ERA} --version dev
```

**What it does:** Queries DAS (or a local directory) to build per-dataset JSON file lists containing the NanoAOD file paths for each configured dataset. These lists are used by all downstream tasks to locate input data.

**Inputs:** `config/Run3_{era}/datasets.yaml` (DAS paths or local `dirName` entries)
**Outputs:** JSON file lists, staged to the configured filesystem

**ERA** should be one of: `Run2_2016`, `Run2_2016_HIPM`, `Run2_2017`, `Run2_2018`, `Run3_2022`, `Run3_2022EE`, `Run3_2023`, `Run3_2023BPix`, `Run3_2024`

### Phase 2: AnaTuple Production (NanoAOD → Analysis Ntuples)

```bash
law run AnaTupleFileTask            --period ${ERA} --version dev [--workflow htcondor]
law run AnaTupleFileListBuilderTask --period ${ERA} --version dev
law run AnaTupleFileListTask        --period ${ERA} --version dev
law run AnaTupleMergeTask           --period ${ERA} --version dev [--workflow htcondor]
```

Run `AnaTupleMergeTask` directly and let LAW resolve the upstream dependencies. For deepTau v2p5, add `--customisations deepTauVersion=2p5`.

> **Note:** the standalone `AnaCacheTask` step has been **removed**. The denominator used for MC normalization is now computed inside `AnaTupleMergeTask`.

**What it does:** This chain processes each NanoAOD file through the full event selection and variable definition chain, producing flat ROOT ntuples (AnaTuples), and merges them into the final per-dataset files:

- **`AnaTupleFileTask`** — produces one AnaTuple per input NanoAOD file (one entry per event passing the baseline selection, plus all systematic energy scale variations) together with a JSON report holding its metadata (denominator contributions, event counts, validity).
- **`AnaTupleFileListBuilderTask`** — defines the list of final (merged) anaTuples and merges the individual per-file reports into a per-dataset plan.
- **`AnaTupleFileListTask`** — copies the resulting file list locally.
- **`AnaTupleMergeTask`** — merges the per-file anaTuples according to the plan and computes the MC-normalization denominator.

**Inputs:** NanoAOD ROOT files from DAS/EOS, correction JSONs from cvmfs
**Outputs:** Per-file AnaTuples + JSON reports, then merged per-dataset ROOT files under `fs_anaTuple`

For batch execution:
```bash
# Check job status
law run AnaTupleMergeTask --period Run3_2022EE --version dev --print-status 2,0

# Limit parallel HTCondor jobs
law run AnaTupleFileTask --period Run3_2022EE --version dev --workflow htcondor --parallel-jobs 200

# Run only specific branches (file indices)
law run AnaTupleFileTask --period Run3_2022EE --version dev --branches 0,1,2
```

### Phase 3: Analysis Cache Production (Heavy Payload Computations)

```bash
law run AnalysisCacheTask            --period ${ERA} --version ${VERSION_NAME} --producer-to-run ggF_DNN
law run AnalysisCacheAggregationTask --period ${ERA} --version ${VERSION_NAME} --producer-to-aggregate ggF_DNN
```

**What it does:** Computes computationally expensive per-event payloads that are too slow to run during standard histogram production. The primary payload is the DNN score (`ggF_DNN_HH`, `ggF_DNN_TT`, `ggF_DNN_DY`) computed by the 5-fold MoE neural network ensemble. Results are stored as friend ("cache") trees that are attached to AnaTuples during histogram production. Each heavy producer is configured under the `payload_producers` key in `config/global.yaml`, and `AnalysisCacheTask` runs the one selected via `--producer-to-run`. `AnalysisCacheAggregationTask` performs per-process aggregation for producers that need the whole process.

**Inputs:** Merged AnaTuples from Phase 2, TensorFlow SavedModel files in `config/nn_models/`
**Outputs:** Analysis-cache ROOT files (friend trees) under `fs_anaCacheTuple`, containing the DNN score branches

**Note:** You usually do not run this by hand. Any requested variable that is produced by a payload producer is resolved automatically (via the variable→producer map), so the histogram tasks in Phase 4 pull in the required `AnalysisCacheTask` on demand. This replaces the old `AnaCacheTupleTask` / `DataCacheMergeTask` steps and the `need_cache` flag.

Version naming convention: `vXX_deepTauYY_ZZZ` (e.g. `v01_deepTau2p5_Run3`).

### Phase 4: Histogram Production

Run these tasks in sequence (or start from the last one and let LAW resolve):

```bash
# Per-file HistTuples (flat tuples carrying all variables/weights/regions)
law run HistTupleProducerTask --period ${ERA} --version ${VERSION_NAME}

# Per-file histograms filled from the HistTuples
law run HistFromNtupleProducerTask --period ${ERA} --version ${VERSION_NAME}

# Merge histograms into one per variable (per uncertainty + central)
law run HistMergerTask --period ${ERA} --version ${VERSION_NAME}

# Produce the final plots per variable and channel/category/region
law run HistPlotTask --period ${ERA} --version ${VERSION_NAME}
```

**What it does:** Applies event selection (channel flags, trigger requirements, analysis regions, categories), computes event weights (lumi, cross-section, pileup, ID SFs, b-tag SFs), fills histograms for all configured variables in all (channel, region, category) combinations, for all systematic variations, and produces the final plots.

- **`HistTupleProducerTask`** — builds, for each merged anaTuple, a flat "HistTuple" carrying all the variables, weights, channels, regions and categories needed downstream.
- **`HistFromNtupleProducerTask`** — fills the per-file histograms of the configured variables from the HistTuples.
- **`HistMergerTask`** — merges the per-file/per-sample histograms into one histogram per variable (for each norm/shape uncertainty + central).
- **`HistPlotTask`** — produces the final plots for each variable and channel/category/region.

**Inputs:** Merged AnaTuples, optionally analysis-cache friend trees (DNN scores)
**Outputs:** HistTuples and merged histogram ROOT files under `fs_HistTuple`, plots under `fs_plots`

**QCD estimation** is applied during merging:

```python
# Handled automatically within the histogram tasks via QCD_estimation.py
# The ABCD method uses OS/SS × Iso/AntiIso regions to estimate QCD background
```

**Stack plots** (alternative to `HistPlotTask`, after `HistMergerTask`):

```bash
# Edit Analysis/make_stackplots.py to set your paths, then:
python3 Analysis/make_stackplots.py
```

### Phase 5: Statistical Inference

**5a. Create datacards:**

```bash
cmsEnv python3 StatInference/dc_make/create_datacards.py \
  --input PATH_TO_SHAPES \
  --output PATH_TO_CARDS \
  --config StatInference/config/x_hh_bbtautau_run2.yaml
```

Or via LAW:

```bash
law run CreateDatacardsTask --period ${ERA} --version ${VERSION_NAME}
```

**5b. Compute resonant limits:**

```bash
law run PlotResonantLimits \
  --version dev \
  --datacards 'PATH_TO_CARDS/*.txt' \
  --xsec fb \
  --y-log \
  --workflow htcondor
```

**5c. Combine multiple eras/channels and run non-resonant limits:**

```bash
# Combine datacards across eras and channels
combineCards.py card1=... card2=... > combined_datacard.txt

# Build workspace with HH physics model
text2workspace.py combined_datacard.txt -P HiggsAnalysis.CombinedLimit.PhysicsModel:multiSignalModel

# Run non-resonant limits
cmsEnv combine combined_datacard.txt -M AsymptoticLimits -t -1
```

**5d. Optimise histogram binning (optional, time-consuming):**

```bash
# Start the Bayesian optimization server
python3 StatInference/bin_opt/optimize_channel.py \
  --config StatInference/bin_opt/bin_optimization.yaml \
  --channel tauTau --era Run3_2022EE

# Submit workers to HTCondor
python3 StatInference/bin_opt/submitLimitWorkers.py \
  --config StatInference/bin_opt/bin_optimization.yaml
```

---

## Component Reference

### FLAF Framework

#### What it does

FLAF (Flexible LAW-based Analysis Framework) is the shared infrastructure underlying the entire analysis pipeline. It provides:

- All LAW task class definitions for the processing chain
- The `Setup` singleton that loads and merges YAML configuration files
- The `DataFrameWrapper` class that tracks RDataFrame column definitions
- C++ headers with core physics types loaded via ROOT's JIT compiler
- The `anaTupleProducer.py` core function that processes individual NanoAOD files
- HTCondor workflow definitions for batch submission
- The `BundleTask` system that packages code for workers without AFS access

FLAF resolves configuration in priority order: FLAF built-in config → FLAF per-era config → analysis config → analysis per-era config, with later entries overriding earlier ones.

#### Key files

| File | Role |
|---|---|
| `FLAF/Common/Setup.py` | Central config/setup singleton; loads and merges all YAMLs |
| `FLAF/Common/Utilities.py` | `DataFrameWrapper`, `CreateDataFrame`, working-point enums, header loader |
| `FLAF/Common/BaselineSelection.py` | C++ header loader, reco/gen p4 creation, MET flag filters |
| `FLAF/AnaProd/anaTupleProducer.py` | Core per-file production function |
| `FLAF/AnaProd/tasks.py` | LAW task classes for the AnaTuple pipeline |
| `FLAF/Analysis/tasks.py` | LAW task classes for the histogram pipeline |
| `FLAF/run_tools/law_customizations.py` | Base `Task` and `HTCondorWorkflow` classes |
| `FLAF/include/AnalysisTools.h` | Core C++ type system: `LorentzVectorM`, `RVecLV`, `Channel`, `Period`, `Leg` enums |
| `FLAF/include/HHCore.h` | Physics candidate structs: `HTTCand`, `HbbCand`, `HWWCand` |
| `FLAF/include/BaselineRecoSelection.h` | Reco selection algorithms: `GetHTTCandidates`, `GetBestHTTCandidate`, `GetHbbCandidate` |
| `FLAF/Processors/MCStitching.py` | MC stitching processor for DY inclusive+binned samples |

#### Key C++ types

The C++ headers are loaded via `ROOT.gInterpreter.Declare()` at runtime (JIT compilation). The central types you will see throughout the code:

- `LorentzVectorM` — the standard 4-vector type (pt, eta, phi, mass)
- `RVecLV` — `ROOT::VecOps::RVec<LorentzVectorM>`, a vector of 4-vectors
- `HTTCand<N>` — N-legged H→ττ candidate with leg types, indices, p4s, charges, raw isolation, gen-match indices
- `HbbCand` — two-jet H→bb candidate
- `Channel` — computed from the leg types via compile-time integer encoding

#### How to run

FLAF tasks are not run directly — they are invoked through the LAW task names listed in the Workflow section above. The key parameter shared by all tasks is `--period` (era name) and `--version` (output version tag).

#### Inputs and outputs

- **Inputs:** NanoAOD ROOT files, YAML configs, correction JSON/gz files from cvmfs
- **Outputs:** Per-file AnaTuple ROOT files, JSON denominator reports, merged AnaTuple files, histogram ROOT files

---

### AnaProd: Ntuple Production

#### What it does

`AnaProd/` contains the analysis-specific production layer that sits on top of FLAF's generic infrastructure. It defines precisely which physics objects, variables, and selections define this HH→bbττ analysis. The central question it answers is: given a NanoAOD event, which objects should be selected and which branches should be stored?

Key responsibilities:

- **Object preselection** (`baseline.py`): Applies per-object kinematic cuts, defines H→ττ candidates via isolation-ranked ordering, applies third-lepton veto, selects b-jet candidates, defines VBF jet candidates, performs gen-reco jet matching.
- **Variable definitions** (`anaTupleDef.py`): Specifies which NanoAOD branches to save for each object type (tau, muon, electron, jet, fatjet, subjet), and defines all derived variables (channel ID, trigger branches, gen-level quantities for MC, HHbtag scores).
- **HHbtag initialization** (`anaTupleDef.py`): Loads the shared library `libHHToolsHHbtag.so` and initialises the HHBtagWrapper pointing to the v3 model files.
- **DNN interface** (`interface.py`): The canonical `NNInterface` class that loads TensorFlow SavedModels for signal/background classification (HH vs TT vs DY).

The module is loaded dynamically by `FLAF/AnaProd/anaTupleProducer.py` via the `anaTupleDef` config key in `global.yaml`.

#### Key files

| File | Role |
|---|---|
| `AnaProd/anaTupleDef.py` | Central column-definition module; lists of branches to save per object type; `addAllVariables()` called per systematic |
| `AnaProd/baseline.py` | Object preselection functions: `RecoHttCandidateSelection`, `RecoJetSelection`, `DefineHbbCand`, `VBFJetSelection`, `ThirdLeptonVeto` |
| `AnaProd/interface.py` | `NNInterface` class: loads TF SavedModel, applies phi-rotation, returns HH/TT/DY class probabilities |

#### How it works

When `anaTupleProducer.py` processes a NanoAOD file, it:

1. Calls `AnaProd.anaTupleDef.Initialize(setup, dataset_name)` once to load the HHbtag library.
2. For each systematic variation (Central + all energy scale shifts), calls `anaTupleDef.addAllVariables(dfw, syst_name, ...)`, which:
   - Applies the jet veto map
   - Calls `baseline.RecoHttCandidateSelection` to find and rank H→ττ candidates
   - Calls `baseline.DefineHbbCand` to compute HHbtag scores and select the H→bb candidate
   - Defines all output branches (kinematics, tau/lepton/jet observables, gen-matching, channel ID, trigger flags)
3. Snapshots each systematic variation's RDataFrame to a ROOT file, then fuses the variation files into a single file with delta branches.

#### Object preselection summary

- **Taus:** DeepTau v2p5 (or v2p1) WPs: VSe=VVLoose, VSmu=Tight, VSjet=Medium; pT > 20 GeV, |η| < 2.3
- **Muons:** TightId, pT > 19 GeV, |η| < 2.4
- **Electrons:** mvaIso WP80, pT > 24 GeV, |η| < 2.1
- **Jets:** AK4PFPuppi, tight jet ID, pT > 20 GeV, |η| < 2.5; b-tag via ParticleNet (Run 3) or DeepFlavour (Run 2)
- **H→ττ candidate selection:** Best candidate by (rawIso ascending, pT descending, |η| ascending)
- **H→bb candidate selection:** Two jets with highest HHbtag scores passing b-jet candidate filter

#### Inputs and outputs

- **Inputs:** NanoAOD ROOT files (Events tree), correction singleton from `Corrections/`
- **Outputs:** AnaTuple ROOT files with one entry per event passing baseline selection; branches include all objects' 4-vectors, IDs, isolation variables, generator-level quantities, and one entry per systematic variation via delta branches

---

### Corrections

#### What it does

`Corrections/` is a standalone library (git submodule) providing all MC-to-data corrections applied during ntuple production and histogram filling. It uses a consistent pattern: each correction has a Python class (orchestration, initialization) paired with a C++ header (actual evaluation logic). The C++ code is loaded at runtime via ROOT's JIT compiler. All correction data is read from CMS POG JSON files on cvmfs or from local files in `Corrections/data/`.

The central class `Corrections.py::Corrections` acts as the orchestrator. It is initialized once globally via `Corrections.initializeGlobal(...)`, reads which corrections to apply from the config hierarchy, and provides two main entry points:

- `applyScaleUncertainties(df, ana_reco_objects)` — applies all energy scale variations and propagates to MET
- `getNormalisationCorrections(df, ...)` — applies all event-level weight corrections

Every correction produces three variations (Central, Up, Down). Weight branches follow the naming convention `weight_X_Central` and `weight_X_{syst}_rel = weight_X_varied / weight_X_Central`.

#### Corrections applied

| Correction | Files | Data source | Systematics |
|---|---|---|---|
| Tau energy scale (TES) | `tau.py`, `tau.h` | cvmfs TAU POG JSON | DM-dependent: DM0, DM1, 3prong; e/μ fakes |
| Tau ID scale factors | `tau.py`, `tau.h` | cvmfs TAU POG JSON | stat1/2 per DM, syst alleras/year, high-pT |
| Jet energy corrections (JEC) | `jet.py`, `jet.h` | cvmfs JME JERC JSON | Total (default) or 11 regrouped sources |
| Jet energy resolution (JER) | `jet.py`, `jet.h` | cvmfs JME JSON | JER smearing (MC only) |
| b-tagging SFs (shape mode) | `btag.py`, `btagShape.h` | cvmfs BTV JSON | lf, hf, lfstats1/2, hfstats1/2, cferr1/2 |
| Pileup reweighting | `pu.py`, `pu.h` | cvmfs LUM JSON | pu Up/Down |
| Muon ID/isolation SFs | `mu.py`, `mu.h` | cvmfs MUO JSON | MuonID |
| Muon ScaRe (energy scale) | `MuonEnergyScale_corr.py` | Local data/MUO/ | ScaRe Up/Down |
| Electron ID SFs | `electron.py`, `electron.h` | cvmfs EGM JSON | EleID |
| Electron energy scale | `electron.py`, `electron.h` | cvmfs EGM JSON | EleES |
| Trigger SFs (Run 3) | `triggersRun3.py`, `triggersRun3.h` | Mix cvmfs + local | Per channel and decay mode |
| DY corrections | `DY_hhbbtautau.py`, `DY_hhbbtautau.h` | Local data/DY_corr_hbt/ | stat_btag{0,1,2}, syst_gauss, syst_linear |
| EWK V pT corrections | `Vpt.py`, `Vpt.h` | Local data/EWK_Corr_Vpt/ | Vpt, ewcorr, DYWeight |
| MET propagation | `met.py`, `met.h` | N/A | Propagates all object shifts |
| Luminosity / XS / gen weight | inline in `Corrections.py` | Config + CrossSectionDB | N/A |
| Golden JSON lumi filter | `lumi.py`, `lumi.h` | Local data/golden_json/ | Data filter only |
| Jet veto maps | `JetVetoMap.py`, `JetVetoMap.h` | cvmfs JME JSON | Selection veto only |

#### How to configure which corrections are applied

Corrections are assigned per processing stage in `config/global.yaml` under the `corrections` key:

```yaml
corrections:
  AnaTuple:
    - lumi
    - xs
    - gen
    - PU
    - tauES
    - JEC
    - JER
  HistTuple:
    - tauID
    - muon
    - electron
    - btag
    - trigger
    - DY
  AnaTupleMerge:
    - base
```

#### Key files

- `Corrections/Corrections.py` — central orchestrator
- `Corrections/CorrectionsCore.py` — period name mappings (e.g. `Run3_2022EE` → `2022_Summer22EE`)
- `Corrections/corrections.h` — C++ `CorrectionsBase<T>` CRTP singleton base class
- `Corrections/data/` — local correction files (muon ScaRe, DY corrections, trigger SFs, golden JSONs)

#### Inputs and outputs

- **Inputs:** RDataFrame with NanoAOD branches, POG JSON files
- **Outputs:** New RDataFrame columns: `{Obj}_p4_{SystName}` for energy scale variations; `weight_X_Central` and `weight_X_{syst}_rel` for normalisation weights

---

### Analysis: Histogramming

#### What it does

`Analysis/` implements the histogram-level analysis layer: event categorisation, weight assembly, DNN application, QCD estimation, and plot production. It sits between the AnaTuple ROOT files and the statistical inference inputs.

The entry point is `Analysis/histTupleDef.py`, which is called by FLAF's histogram tasks via the `histTupleDef` config key. The actual physics logic is in `Analysis/hh_bbtautau.py`.

#### Key files and their roles

**`Analysis/hh_bbtautau.py`** — the core analysis module:

- `createInvMass()`: Defines kinematic observables on the RDataFrame: τtau and bb visible masses, boosted FatJet softdrop/ParticleNet masses, ΔR/Δη/Δφ between the two Higgs candidates, MET transverse masses.
- `createKeyFilterDict()`: Builds a dictionary mapping `(channel, region, category)` tuples to C++ boolean filter expressions combining channel flags, trigger conditions, region definitions, and category definitions.
- `GetWeight()`: Constructs the per-channel event weight expression combining base weight, tau DeepTau ID SFs, muon tight ID SFs, and electron mvaIso SFs.
- `DataFrameBuilderForHistograms`: Orchestrates all column definitions in dependency order.
- `PrepareDfForHistograms()` / `PrepareDfForDNN()`: Top-level orchestration called by FLAF.

**`Analysis/histTupleDef.py`** — FLAF entry point:

- `GetDfw()`: Constructs a `DataFrameBuilderForHistograms` (distinguishing TT, DY, and other datasets for PNet SF purposes).
- `DefineWeightForHistograms()`: Applies all normalisation corrections and builds the final event weight.

**`Analysis/GetCrossWeights.py`** — trigger scale factors:

- `defineTriggersCentralWeights()`: Defines central trigger SFs per channel using data/MC efficiency ratios with proper OR combination for cross-triggers.
- `defineTriggersWeightsErrors()`: Propagates systematic uncertainties per decay mode and trigger leg.

**`Analysis/interface.py`** — canonical DNN interface:

- `NNInterface`: Loads a TensorFlow SavedModel for one fold, applies phi-rotation to align all 4-vectors to the di-lepton frame, builds composite 4-vectors (H→ττ, H→bb, HH system), assembles 39-feature continuous input tensor plus categorical inputs, returns softmax probabilities for HH, TT, and DY classes.

**`Analysis/DNN_application.py`** — production DNN runner:

- `DNNProducer`: Manages all 5 fold models, calls inference, averages fold predictions, outputs `ggF_DNN_HH`, `ggF_DNN_TT`, `ggF_DNN_DY` columns.

**`Analysis/QCD_estimation.py`** — data-driven QCD:

- `QCD_Estimation()`: ABCD method using OS/SS × Iso/AntiIso regions. Transfer factor = N(data, OS AntiIso) / N(data, SS AntiIso) after MC subtraction. QCD shape taken from SS Iso region scaled by transfer factor.
- `AddQCDInHistDict()`: Orchestrates estimation across all channels, categories, and uncertainty scales.

#### Analysis regions and categories

**Channels:** eTau, muTau, tauTau (signal); eE, eMu, muMu (control)

**Regions:**
- Signal region (SR): OS, Iso
- DYCR: Z-peak control region
- TTCR: top pair control region

**QCD ABCD regions:** OS Iso (A = SR), OS AntiIso (B), SS Iso (C), SS AntiIso (D)

**Categories:**
- `baseline`: inclusive pass of full selection
- `inclusive`: baseline without b-tag requirement
- `boosted`: one AK8 FatJet with ParticleNet HbbvsQCD score above threshold
- `res2b`: two AK4 b-jets passing medium b-tag WP
- `res1b`: exactly one AK4 b-jet passing medium b-tag WP

**Application regions:** defined by single-lepton or di-tau trigger thresholds; pT thresholds vary by era (e.g. muon > 26 GeV for Run 3, electron > 32 GeV for Run 3, tau > 190 GeV for Run 3).

#### DNN input features (39 total)

The `ggF_DNN` network takes:
- MET: pt, phi
- τ1, τ2: pt, eta, phi, mass, charge, decay mode
- b1, b2: pt, eta, phi, mass, DeepFlavB, PNetB, CvsB, CvsL, HHbtag score
- FatJet: pt, eta, phi, mass, PNetXbb
- Categorical: pair type (channel), decay modes, charges, `is_boosted`, `has_bjet_pair`

All 4-vectors are phi-rotated to align τ1 with φ=0 before inference.

#### How to run histogram production

```bash
# Full pipeline (LAW resolves dependencies):
law run HistPlotTask --period Run3_2022EE --version v01_deepTau2p5_Run3 --workflow htcondor

# Check status:
law run HistPlotTask --period Run3_2022EE --version v01_deepTau2p5_Run3 --print-status 4,0

# Or stop at the merged histograms (skip plotting):
law run HistMergerTask --period Run3_2022EE --version v01_deepTau2p5_Run3 --workflow htcondor

# Generate stack plots manually (edit paths in script first):
python3 Analysis/make_stackplots.py
```

The chain is `HistTupleProducerTask → HistFromNtupleProducerTask → HistMergerTask → HistPlotTask`.

#### Inputs and outputs

- **Inputs:** Merged AnaTuple ROOT files (plus analysis-cache friend trees for DNN scores)
- **Outputs:** Per-file HistTuples and merged histograms (one per variable, per systematic variation) under `fs_HistTuple`, and final plots under `fs_plots`

---

### HHbtag

#### What it does

HHbtag is an HH-specific multi-jet b-tagging discriminant that scores each jet in the context of the full event — incorporating the di-tau candidate kinematics, MET, jet kinematics, and the event's decay topology. Unlike standard b-taggers (DeepFlavour, ParticleNet) that score jets in isolation, HHbtag uses the global event context to better identify which jets come from H→bb.

The output is a per-jet score; the two jets with the highest scores are selected as the H→bb candidate legs.

#### Architecture

- **Backend:** TensorFlow SavedModel, wrapped by `HHbtag/src/HH_BTag.cpp` using CMSSW's `PhysicsTools/TensorFlow` wrapper.
- **Dual-model ensemble with parity splitting:** Two models are loaded simultaneously (`HHbtag_v3_par_0` and `HHbtag_v3_par_1`). The model used for each event is determined by `event_number % 2`. This prevents the model from scoring its own training events.
- **Input tensor shape:** `[1, 10, 15]` — one event, up to 10 jets, 15 features per jet. Unoccupied jet slots are zero-padded.
- **Currently active version:** v3 (Run 3, uses ParticleNet b-tag score as input, trained on all channel types).

#### Input features (15 per jet)

| Index | Variable | Description |
|---|---|---|
| 0 | jet_valid | 1 for real jet, 0 for padding |
| 1 | jet_pt | Jet pT |
| 2 | jet_eta | Jet η |
| 3 | rel_jet_M_pt | M(jet) / pT(jet) |
| 4 | rel_jet_E_pt | E(jet) / pT(jet) |
| 5 | jet_htt_deta | η(HTT visible) − η(jet) |
| 6 | jet_btagScore | ParticleNet B score (v3) |
| 7 | jet_htt_dphi | ΔΦ(HTT visible, jet) |
| 8 | sample_year | Era integer (0=2022preEE, 1=2022postEE, 2=2023preBPix, 3=2023postBPix) |
| 9 | channelId | muTau=0, eTau=1, tauTau=2, muMu=3, eE=4, eMu=5 |
| 10–14 | HTT event variables | htt_pt, htt_eta, htt_met_dphi, rel_met_pt_htt_pt, htt_scalar_pt |

Event-level variables (indices 8–14) are repeated identically across all jet slots.

#### How it is called

In `AnaProd/anaTupleDef.py`:

```python
# Initialization (once per job):
HHBtagWrapper.Initialize(model_path, 3)  # 3 = version 3

# Per-event scoring (via RDataFrame Define):
"Jet_HHbtag",
"GetHHBtagScore(Jet_bCand, Jet_idx, Jet_p4, Jet_btagPNetB, MET_pt, MET_phi, HttCandidate, period, event)"
```

The `GetHHBtagScore` function lives in `FLAF/include/HHbTagScores.h`. It reorders jets by ParticleNet score, computes all input features, and calls `HH_BTag::GetScore(parity = event % 2)`.

The two highest-scoring jets become the H→bb candidate:

```python
"HbbCandidate", "GetHbbCandidate(Jet_HHbtag, Jet_bCand, Jet_p4, Jet_idx)"
```

The scores are stored as `b1_HHbtag` and `b2_HHbtag` in the output AnaTuple, and used as features in the downstream `ggF_DNN` network.

#### Key files

- `HHbtag/interface/HH_BTag.h` — C++ class declaration
- `HHbtag/src/HH_BTag.cpp` — TF inference implementation
- `HHbtag/models/HHbtag_v3_par_0/` and `HHbtag_v3_par_1/` — active model files
- `FLAF/include/HHbTagScores.h` — `GetHHBtagScore()` and `GetHbbCandidate()` wrappers
- `AnaProd/anaTupleDef.py` — initialization and `b1/b2_HHbtag` storage
- `Analysis/interface.py` and `Analysis/DNN_application.py` — downstream DNN inputs

#### Inputs and outputs

- **Inputs:** Jet 4-vectors, ParticleNet B scores, H→ττ candidate 4-momentum, MET, era index, event number
- **Outputs:** Per-jet `Jet_HHbtag` score vector; stored in AnaTuple as `b1_HHbtag`, `b2_HHbtag`

---

### HHKinFit2

#### What it does

HHKinFit2 is a kinematic fitter for HH→bbττ events. It performs a constrained chi-squared minimisation to reconstruct the di-Higgs invariant mass (`kinFit_m`), exploiting two physics constraints: the H→ττ system must have invariant mass 125 GeV, and the H→bb system must have invariant mass 125 GeV. The fit additionally enforces MET consistency via the 2×2 MET covariance matrix.

`kinFit_m` is the primary discriminating variable for the resonant HH→bbττ search (X→HH→bbττ), as it directly reconstructs the mass of the heavy resonance.

#### Algorithm

The fitter operates on two free parameters: E(τ1) and E(b1). Given these, the hard mass constraints determine E(τ2) from the H→ττ mass constraint and E(b2) from the H→bb mass constraint. The chi-squared objective has contributions from:

1. **Momentum balance:** `χ²_MET = (ΔpT)^T × Σ_MET^{-1} × (ΔpT)` where ΔpT is the residual transverse momentum of the full (τ1+τ2+b1+b2+MET) system.
2. **b-jet energy pulls:** `χ²_b = (E_fit − E_meas)² / σ_E²` for each b-jet.

The minimiser (`PSMath::PSfit`) is an iterative Newton-step algorithm running up to 10,000 iterations with line-search and Cholesky decomposition. Three starting points are tried to avoid local minima.

#### Interface

The analysis wrapper is in `include/KinFitInterface.h`:

```cpp
// Called once per event from RDataFrame
kin_fit::FitResults result = kin_fit::FitProducer::Fit(
    tau1_p4, tau2_p4, b1_p4, b2_p4, met_p4,
    met_covXX, met_covXY, met_covYY,
    b1_ptRes * E(b1_p4),   // absolute energy resolution
    b2_ptRes * E(b2_p4)
);

// Access results:
result.HasValidMass()  // convergence > 0
result.mass            // fitted di-Higgs mass in GeV
result.chi2
result.probability     // TMath::Prob(chi2, 2)
```

#### Integration in the analysis

The fitter is called in `AnaProd/LegacyVariables.py` via `GetKinFit(df)`, which adds these branches to the AnaTuple:

- `kinFit_convergence` — convergence code (positive = converged)
- `kinFit_m` — fitted di-Higgs mass in GeV (primary discriminant)
- `kinFit_chi2` — fit chi-squared

Note: The b-jet energy resolutions passed to the fitter are `b1_ptRes * E(b1)`, converting fractional pT resolution from JER tables to an absolute energy resolution. The tau 4-vectors use their visible decay products (not the full tau momentum including neutrinos).

`kinFit_m` is histogrammed with variable binning from 250 to 3000 GeV and used as the final discriminant in the cut-based analysis categories.

#### Key files

- `HHKinFit2/interface/HHKinFitMasterHeavyHiggs.h` — primary C++ API
- `HHKinFit2/src/HHKinFitMasterHeavyHiggs.cpp` — fit logic, b-jet resolution table
- `HHKinFit2/src/HHKinFit.cpp` — core iterative minimiser
- `include/KinFitInterface.h` — analysis wrapper providing `kin_fit::FitProducer::Fit()`
- `include/KinFitNamespace.h` — lightweight `kin_fit::FitResults` struct (for study scripts)
- `AnaProd/LegacyVariables.py` — RDataFrame integration

#### Inputs and outputs

- **Inputs:** Two tau 4-vectors (visible), two b-jet 4-vectors, MET 4-vector, 2×2 MET covariance, b-jet energy resolutions
- **Outputs:** `kinFit_m`, `kinFit_chi2`, `kinFit_convergence` branches in AnaTuple

**Note on SVfit:** The `include/SVfitAnaInterface.h` header wraps `TauAnalysis/ClassicSVfit` for di-tau mass reconstruction, producing `SVfit_m`, `SVfit_pt`, `SVfit_eta`, `SVfit_phi`, and `SVfit_mt`. These are also defined via `AnaProd/LegacyVariables.py` and used as DNN input features.

---

### StatInference and Inference

#### What they do

The statistical inference step takes the merged histogram ROOT files from the Analysis step and produces limits and significance estimates. It is split into two components:

- **`StatInference/`** (analysis-specific): Creates CMS-format datacards from the analysis histograms using CombineHarvester. Also contains the Bayesian binning optimisation framework.
- **`inference/` (dhi)**: The CMS HH combination inference framework. Takes datacards as input and produces workspaces, limits, likelihood scans, postfit plots, pulls/impacts, and goodness-of-fit results using CMS Combine.

#### StatInference: Datacard production

The `DatacardMaker` class in `StatInference/dc_make/maker.py` wraps CombineHarvester:

1. Reads the analysis config (`StatInference/config/x_hh_bbtautau_run2.yaml`) specifying eras, channels, categories, signal mass hypotheses, backgrounds, and all systematic uncertainties.
2. Opens per-era shape ROOT files (histograms organised as `{channel}/{category}/{process_name}`).
3. For each (era, channel, category) combination, adds observations, signal processes (one per mass hypothesis), and background processes.
4. Applies rebinning from `StatInference/config/hhbbtautau_binning.json`.
5. Resolves negative bins (zeroes bins consistent with zero within N-sigma, redistributes content to neighbours).
6. Adds systematic uncertainties: `lnN` (log-normal rate uncertainties), `shape` (shape variations from Up/Down histograms), `auto` (automatically resolves to lnN or shape based on a chi-squared flat-fit test).
7. Writes one `.txt` datacard and one `.root` shape file per signal mass point.

```bash
cmsEnv python3 StatInference/dc_make/create_datacards.py \
  --input /path/to/all_histograms_Hadded.root \
  --output /path/to/datacards/ \
  --config StatInference/config/x_hh_bbtautau_run2.yaml
```

#### StatInference: Binning optimisation

The `StatInference/bin_opt/` directory contains a Bayesian optimisation framework that finds the histogram bin boundaries that minimise the expected 95% CL upper limit. It uses the `bayesian-optimization` Python library with a server-worker architecture:

- The server (`optimize_channel.py`) runs a Bayesian optimisation loop, proposing bin edge candidates and tracking the expected limit as the objective.
- Workers (`rebinAndRunLimitsWorker.py`) receive bin edge proposals, rebin the histograms, create temporary datacards, and run `law run UpperLimits` via the `dhi` framework.
- Workers can be submitted to HTCondor for parallelisation.

The resulting optimised bin edges are stored in `StatInference/config/hhbbtautau_binning.json`.

#### Inference (dhi): Limit computation

The dhi framework provides LAW tasks for the full statistical analysis pipeline. Key tasks and their commands:

```bash
# Source the dhi environment first:
source inference/setup.sh

# Run asymptotic limits for resonant search:
law run UpperLimits --version dev --datacards 'datacards/*.txt'

# Plot resonant limits:
law run PlotResonantLimits --version dev --datacards 'datacards/*.txt' --xsec fb --y-log

# Likelihood scan over kappa_lambda:
law run LikelihoodScan --version dev --scan-parameters kl

# Pulls and impacts:
law run PullsAndImpacts --version dev
```

The HH physics model (`inference/dhi/models/hh_model.py`) implements the parameterisation of HH cross-sections as functions of κ_λ, κ_t, C_2V, C_V, c_2 for both ggF and VBF production at 13 and 13.6 TeV.

#### Data flow

```
all_histograms_Hadded.root
  → StatInference/dc_make/create_datacards.py (DatacardMaker + CombineHarvester)
  → datacard_{mass}.txt + shapes_{mass}.root
  → inference/dhi/tasks/combine.py (CombineDatacards + CreateWorkspace)
  → workspace.root (RooFit workspace with HH model)
  → inference/dhi/tasks/limits.py (UpperLimits → AsymptoticLimits via combine)
  → limits.npz
  → inference/dhi/tasks/limits.py (PlotResonantLimits)
  → limits_resonant.pdf
```

#### Key files

| File | Role |
|---|---|
| `StatInference/dc_make/create_datacards.py` | CLI entry point for datacard creation |
| `StatInference/dc_make/maker.py` | Core `DatacardMaker` class |
| `StatInference/dc_make/uncertainty.py` | Uncertainty types: LnN, Shape, Auto |
| `StatInference/config/x_hh_bbtautau_run2.yaml` | Full analysis config for Run 2 resonant search |
| `StatInference/config/hhbbtautau_binning.json` | Pre-optimised bin edges per channel/category |
| `StatInference/bin_opt/optimize_channel.py` | Bayesian binning optimisation server |
| `StatInference/law/tasks.py` | LAW tasks: `CreateDatacardsTask`, `ResonantLimitsTask` |
| `inference/dhi/config.py` | HH branching ratios, luminosities, POI definitions |
| `inference/dhi/tasks/combine.py` | `CombineDatacards`, `CreateWorkspace` tasks |
| `inference/dhi/tasks/limits.py` | `UpperLimits`, `PlotResonantLimits` tasks |
| `inference/dhi/models/hh_model.py` | HH physics model (GGF and VBF parameterisations) |

#### Inputs and outputs

- **Inputs:** `all_histograms_Hadded.root` from the Analysis step
- **Outputs:** Text datacards, ROOT shape files, RooFit workspaces, limit `.npz` files, PDF plots

---

## Configuration System

### How configurations are loaded

FLAF loads configurations by merging multiple YAML files in priority order. Later entries override earlier ones for scalar values; list values are replaced, not extended. The search path is:

1. `FLAF/config/global.yaml` (framework defaults)
2. `FLAF/config/{era}/global.yaml` (framework era overrides)
3. `config/global.yaml` (analysis-level master config)
4. `config/{era}/global.yaml` (analysis era overrides)
5. `config/user_custom.yaml` (personal overrides, not committed)

### Key configuration files

**`config/global.yaml`** — the master analysis config (639 lines). Controls:
- Module paths: `anaTupleDef: AnaProd.anaTupleDef`, `histTupleDef: Analysis.histTupleDef`
- Remote cache endpoints and bundle definitions
- `signal_types`: which signal categories are active (currently only `HHnonRes`; resonant signals require uncommenting entries)
- `deepTauWPs`: VSe=VVLoose, VSmu=Tight, VSjet=Medium
- `corrections`: per-stage assignment of which corrections to apply
- `channelSelection`: which channels to process
- `category_definition`: baseline, inclusive, boosted, res2b, res1b
- `payload_producers.ggF_DNN`: DNN configuration (5 folds, 35 input features, 3 output classes)
- `variables`: list of variables to histogram
- Per-period trigger pT thresholds (`singleMu_th`, `singleEle_th`, `singleTau_th`)
- Mass window limits for the bb and ττ visible mass cuts
- JES uncertainty lists per era

**`config/Run3_{era}/datasets.yaml`** — dataset entries with DAS paths, cross-sections, and signal parameters (mass, spin). Signal datasets include Radion (spin-0) and BulkGraviton (spin-2) at masses 250–5000 GeV, and non-resonant ggF/VBF HH EFT coupling points.

**`config/Run3_{era}/processes.yaml`** — maps process group names (DY, TT, ST, ggH, HH, etc.) to datasets. Supports `is_meta_process: true` for automatic expansion of signal grids.

**`config/Run3_{era}/triggers.yaml`** — trigger paths per channel with per-leg offline cuts, online filter bit requirements, and trigger SF JSON keys.

**`config/Run3_{era}/weights.yaml`** — normalisation and shape uncertainty definitions (weight names and Up/Down branches).

**`config/phys_models.yaml`** — defines `BaseModel` (physics backgrounds + signals + data) and `TestModel` (CI testing). Controls which processes are treated as signal versus background versus data in the statistical analysis.

**`config/plot/histograms.yaml`** — per-variable binning. Bins are channel- and category-specific (e.g. `tau1_pt` has different bin edges for eTau/muTau/tauTau and for each category).

**`config/user_custom.yaml`** — personal overrides. Must be created manually (not committed to git). Essential entries: filesystem paths for outputs, list of variables to plot, flags for uncertainty computation.

### Adding a new era

To add support for a new data-taking period:

1. Create `config/Run3_{era}/` with `global.yaml`, `triggers.yaml`, `weights.yaml`, `datasets.yaml`, `processes.yaml`.
2. Add the era's trigger pT thresholds to `config/global.yaml` under `singleMu_th`, `singleEle_th`, `singleTau_th`.
3. Add the era to `uncs_to_exclude` if JES uncertainties need filtering.
4. Add correction period mappings in `Corrections/CorrectionsCore.py`.
5. Add a plot YAML in `config/plot/Run3_{era}.yaml`.

### Adding a new variable to histograms

1. Add the column definition to `Analysis/hh_bbtautau.py` (in `createInvMass()` or a new method in `DataFrameBuilderForHistograms`).
2. Add binning to `config/plot/histograms.yaml`.
3. Add the variable name to the `variables` list in `config/user_custom.yaml` (or `config/global.yaml`).
4. If the variable is computationally expensive, register it as a payload producer under `payload_producers` in `config/global.yaml`; it will then be produced by `AnalysisCacheTask` and pulled in automatically when requested (no `need_cache` flag needed).

---

## Interconnection Map

The following describes how all components communicate with each other.

```
NanoAOD (DAS/EOS)
        |
        | [InputFileTask: FLAF/AnaProd/tasks.py]
        |
        v
   File Lists (JSON)
        |
        | [AnaTupleFileTask → AnaTupleFileListBuilderTask → AnaTupleFileListTask
        |  → AnaTupleMergeTask: FLAF/AnaProd/tasks.py
        |  AnaTupleFileTask calls anaTupleProducer.py which calls
        |  AnaProd/anaTupleDef.py and AnaProd/baseline.py for physics selections;
        |  AnaTupleMergeTask merges per-file anaTuples and computes the
        |  MC-normalization denominator (no more separate AnaCacheTask)]
        |
        | Uses: Corrections/ (via Corrections.initializeGlobal)
        |       HHbtag/ (via HHBtagWrapper in anaTupleDef.py)
        |       HHKinFit2/ + ClassicSVfit (via include/KinFitInterface.h,
        |                                   include/SVfitAnaInterface.h)
        |       FLAF/include/*.h (AnalysisTools.h, HHCore.h,
        |                          BaselineRecoSelection.h)
        |       config/Run3_{era}/datasets.yaml, triggers.yaml, global.yaml
        v
   AnaTuple ROOT files
   (merged per-dataset, per-systematic-variation delta branches)
        |
        | [AnalysisCacheTask (+ AnalysisCacheAggregationTask): FLAF/Analysis/tasks.py
        |  calls Analysis/DNN_application.py (DNNProducer)
        |  which calls Analysis/interface.py (NNInterface)]
        |
        | Uses: config/nn_models/model_fold{0-4}_moe/ (TF SavedModels)
        |       b1/b2_HHbtag from AnaTuples as DNN input features
        v
   Analysis-cache ROOT files (friend trees)
   (DNN score branches: ggF_DNN_HH, ggF_DNN_TT, ggF_DNN_DY)
        |
        | [HistTupleProducerTask → HistFromNtupleProducerTask → HistMergerTask
        |  → HistPlotTask, all in FLAF/Analysis/tasks.py
        |  calls Analysis/histTupleDef.py → Analysis/hh_bbtautau.py]
        |
        | Uses: Corrections/ (normalisation weights: b-tag, tau ID, mu/ele ID, trigger)
        |       Analysis/GetCrossWeights.py (trigger SF computation)
        |       Analysis/QCD_estimation.py (ABCD method, applied post-merge)
        |       config/global.yaml (category defs, region defs, variables)
        |       config/plot/histograms.yaml (binning)
        v
   Merged histograms under fs_HistTuple
   (per-process, per-channel/region/category, per-variable histograms,
    one per systematic variation)  +  final plots under fs_plots
        |
        | [CreateDatacardsTask: StatInference/law/tasks.py
        |  calls StatInference/dc_make/create_datacards.py (DatacardMaker)]
        |
        | Uses: CombineHarvester Python bindings
        |       StatInference/config/x_hh_bbtautau_run2.yaml
        |       StatInference/config/hhbbtautau_binning.json
        v
   Datacards (.txt) + Shape files (.root)
   (one per signal mass hypothesis, per channel/category combination)
        |
        | [dhi: inference/dhi/tasks/combine.py (CombineDatacards, CreateWorkspace)
        |       inference/dhi/tasks/limits.py (UpperLimits, PlotResonantLimits)]
        |
        | Uses: CMS Combine (text2workspace.py, combine -M AsymptoticLimits)
        |       inference/dhi/models/hh_model.py (GGF/VBF cross-section parameterisation)
        v
   Limits, likelihood scans, postfit plots
```

### Component dependency summary

| Component | Depends on | Provides to |
|---|---|---|
| `config/` | (none — source of truth) | All components |
| `FLAF/include/*.h` | (compiled at runtime) | AnaProd, Corrections, Analysis |
| `Corrections/` | `config/`, cvmfs JSON files | AnaProd (energy scales), Analysis (weights) |
| `HHbtag/` | CMSSW TF wrapper | AnaProd (b-jet ranking for HbbCand) |
| `HHKinFit2/` | (standalone, ROOT) | AnaProd via `include/KinFitInterface.h` |
| `AnaProd/` | FLAF, Corrections, HHbtag, HHKinFit2, config | AnaTuples |
| `Analysis/interface.py` | TensorFlow, config/nn_models/ | DNN_application.py, analysis-cache trees |
| `Analysis/hh_bbtautau.py` | FLAF, Corrections, AnaTuples | Histogram ROOT files |
| `Analysis/QCD_estimation.py` | Histogram ROOT files | QCD background estimate |
| `StatInference/dc_make/` | Histogram ROOT files, CombineHarvester | Datacards |
| `StatInference/bin_opt/` | Datacards, dhi inference | Optimised binning JSON |
| `inference/dhi/` | Datacards, CMS Combine | Limits, workspaces, plots |

---

## Known Issues and Caveats

This section documents known bugs and design limitations that a new collaborator should be aware of.

### Corrections

- **Tau UncSource enum collision** (`Corrections/tau.h`): `stat1_dm1` and `stat2_dm0` both have integer value 7 in the C++ enum. They are indistinguishable internally, meaning the wrong uncertainty source may be evaluated for one of these two.
- **2024/2025 tau trigger SFs use 2023 as placeholder**: `tau_filename_dict` in `triggersRun3.py` maps 2024 and 2025 to `"2023_postBPix"`. Treat 2024/2025 tau trigger SF results with caution.
- **FatJet corrections only cover 2022-2023**: `fatjet.py` has no entries for 2024 or 2025; enabling fatjet corrections for those eras will raise a `KeyError`.
- **Singleton initialization pattern**: Each provider uses a class-level `initialized = False` flag. Re-initializing with different configuration in the same Python session will silently skip the second initialization.

### Configuration

- **`Run3_2024` missing from `global.yaml` trigger thresholds**: `singleMu_th`, `singleEle_th`, `singleTau_th` enumerate periods up to `Run3_2023BPix` but omit `Run3_2024`. This will cause a key error when processing 2024 data.
- **No plot YAML for `Run3_2024`**: `config/plot/` has no file for 2024.
- **Resonant signals not active**: `signal_types` in `global.yaml` only lists `HHnonRes`. The Radion and BulkGraviton entries are commented out and must be uncommented to process resonant signals.
- **`ditaujet` trigger filter bit bug in Run3_2022EE**: Line 54 of `Run3_2022EE/triggers.yaml` uses a single `&` (bitwise AND) instead of `&&` (logical AND) for a filter bit check. This is a real logic error that could silently pass wrong events.

### Analysis

- **Trigger weights partially disabled**: In `Analysis/hh_bbtautau.py::GetWeight()`, the line that adds trigger SFs to the weight expression is commented out. Trigger SFs are instead applied conditionally via `GetCrossWeights.py` when `"trigger"` appears in the corrections config. Verify your config explicitly includes trigger corrections.
- **Logic bug in `defineTriggersCentralWeights`**: `if "muMu" or "eMu" in channels:` always evaluates to `True` (due to Python's short-circuit evaluation of non-empty string). Should be `if "muMu" in channels or "eMu" in channels:`.
- **`mutau_tau_decayMode` copy-paste bug**: In `GetCrossWeights.py`, both branches of a ternary check `tau1_isMatched_mutau_tauLeg` instead of the second branch checking `tau2_isMatched_mutau_tauLeg`. The tau2 decay mode for muTau can never be returned.
- **DNN ignores mass and spin inputs**: In `Analysis/interface.py`, the `mass`, `spin`, and `era` inputs to `NNInterface.predict()` are accepted but the lines that include them in the input tensor are commented out. The model is effectively mass/spin/era agnostic.

### AnaProd

- **`LegacyVariables.py` broken import**: `from .Utilities import *` is a relative import that fails because `AnaProd/` has no `__init__.py`. The file is effectively dead code in the current pipeline.
- **`addLegacyVariables.py` is unused**: No file in the repository calls `applyLegacyVariables`. The kinematic fitter and SVfit variables are therefore not currently being computed in the standard pipeline.
- **HHbtag version hardcoded**: `HHBtagWrapper.Initialize(path, 3)` is hardcoded to version 3. Switching versions requires a code edit.
- **Two CMSSW base env vars**: `anaTupleDef.py::Initialize()` uses `$FLAF_CMSSW_BASE` for the shared library path and `$CMSSW_BASE` for the HHbtag model path. If these differ in the runtime environment, the library and models will be loaded from different CMSSW installations.

### StatInference

- **Hardcoded AFS path**: `StatInference/config/x_hh_bbtautau_run2.yaml` contains `hist_bins: /afs/cern.ch/work/v/vdamante/...` pointing to another user's AFS area. Update this path before use.
- **Era name mismatch in QCD_norm uncertainty**: Uncertainty entries using `eras: Run2_2018` will never match the era list which uses `Run_2018` (with underscore after "Run"), causing those uncertainties to be silently skipped.
- **No limit results in `inference/data/`**: The directory contains only a `.installed` marker. The full pipeline has not yet been run through to the inference stage in this installation, or outputs live on EOS.

### HHKinFit2

- **Silent exception swallowing**: All exceptions from the kinematic fitter are caught with an empty `catch(std::exception&) {}` block, returning a default `FitResults` with `convergence = INT_MIN`. Systematic fit failures produce no diagnostic output.
- **Tau mass override**: The constructor forcibly sets visible tau masses to 1.77682 GeV (tau pole mass), overwriting the actual visible hadronic system mass. This is a known approximation but affects events with multi-prong hadronic tau decays.
- **`kinFit_result` struct column**: `GetKinFit` returns the raw `kin_fit::FitResults` struct as a branch. This non-primitive type can cause issues with ROOT TTree serialization if not handled carefully.