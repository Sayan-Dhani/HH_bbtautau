# HH->bb$\tau$$\tau$ analysis steps

**Commands below assume that AnaTuples have already been produced. If not, please produce them following the instruction in the analysis section.**

Remember that:

- `ERA` variable is set. E.g.
    ```sh
    ERA=Run3_2022
    ```
    Alternatively you can add `ERA=Run3_2022; ...` in front of each command.
    Run2 possible eras are: `Run3_2022`,`Run3_2022EE`,`Run3_2023` and `Run3_2023BPix`
    <br/>
- when expliciting `VERSION_NAME` variable, its name contains explicitly the deepTau version: `VERSION_NAME= vXX_deepTauYY_ZZZ`, where:
    - XX is the anaTuple version (if not the first production it can be useful to have `v1,v2,..`),
    - YY is the deepTau version (`2p1` or `2p5`)
    - ZZZ are other eventual addition (e.g. if only tauTau channel `_onlyTauTau` or if `Zmumu` ntuples `_Zmumu`..)
    <br/>
- `--workflow` can be `htcondor` or `local`. It is recommended to develop and test locally and then switch to `htcondor` for production. In examples below `--workflow local` is used for illustration purposes.<br/> <br/>
- when running on `htcondor` it is recommended to add `--transfer-logs` to the command to transfer logs to local.<br/> <br/>
- `--customisations` argument is used to pass custom parameters to the task in form param1=value1,param2=value2,...
    **IMPORTANT for HHbbTauTau analysis:** if running using deepTau 2p5 add `--customisations deepTauVersion=2p5`<br/> <br/>
- if you want to run only on few files, you can specify list of branches to run using `--branches` argument. E.g. `--branches 2,7:10,17`.<br/> <br/>
- to get status, use `--print-stauts N,K` where N is depth for task dependencies, K is depths for file dependencies. E.g. `--print-status 3,1`.<br/> <br/>
- to remove task output use `--remove-output N,a,y`, where N is depth for task dependencies. E.g. `--remove-output 0,a,y`.<br/> <br/>
- it is highly recommended to limitate the maximum number of parallel jobs running adding `--parallel-jobs M` where M is the number of the parallel jobs (e.g. M=100)

## Analysis cache (heavy payload observables)

Computationally heavy per-event observables (e.g. the ggF DNN score) are not
recomputed during histogramming. They are produced once per anaTuple by
`AnalysisCacheTask` and stored as friend ("cache") trees. Each heavy producer is
configured under the `payload_producers` key in `config/global.yaml`; the
producer to run is selected with `--producer-to-run` (e.g. `ggF_DNN`):

```sh
law run AnalysisCacheTask --period ${ERA} --version ${VERSION_NAME} --producer-to-run ggF_DNN
```

Per-process aggregation of the caches (needed by producers that require the whole
process) is handled by `AnalysisCacheAggregationTask`, selected with
`--producer-to-aggregate`:

```sh
law run AnalysisCacheAggregationTask --period ${ERA} --version ${VERSION_NAME} --producer-to-aggregate ggF_DNN
```

**Note:** You usually do not need to run these by hand. Any requested variable
that is produced by a payload producer is resolved automatically, so the
histogram tasks below pull in the required `AnalysisCacheTask` on demand. This
replaces the old `AnaCacheTupleTask` / `DataCacheMergeTask` steps and the
`need_cache` flag.


### Histograms Production

This has to be run after `AnaTupleMergeTask`. The `AnalysisCacheTask` step above
is only needed (and is then triggered automatically) if a variable to plot is
produced by a payload producer.

These tasks produce histograms with observables that need to be specified inside
the `config/global.yaml` file or `user_custom.yaml`, specifically inside the
`variables` list.

The tasks to run are the following (each depends on the previous one):

1. `HistTupleProducerTask`: for each merged anaTuple a flat "HistTuple" is created,
   carrying all variables, weights, channels, regions and categories needed for
   histogramming.
    ```sh
    law run HistTupleProducerTask --period $ERA --version ${VERSION_NAME}
    ```
1. `HistFromNtupleProducerTask`: for each HistTuple the per-file histograms of the
   configured variables are filled (one per channel/region/category and
   systematic variation).
    ```sh
    law run HistFromNtupleProducerTask --period $ERA --version ${VERSION_NAME}
    ```
1. `HistMergerTask`: the per-file/per-sample histograms are merged into one
   histogram per variable (for each norm/shape uncertainty + the central scenario).
    ```sh
    law run HistMergerTask --period $ERA --version ${VERSION_NAME}
    ```
1. `HistPlotTask`: produces the final plots for each variable and
   channel/category/region.
    ```sh
    law run HistPlotTask --period $ERA --version ${VERSION_NAME}
    ```

Alternatively, after `HistMergerTask`, you can produce stack plots with the
helper script (edit the paths/variable names inside it first):
```sh
cd Analysis
python3 make_stackplots.py
```

## How to run HHbtag training skim ntuple production
```sh
python Studies/HHBTag/CreateTrainingSkim.py --inFile $CENTRAL_STORAGE/prod_v1/nanoAOD/2018/GluGluToBulkGravitonToHHTo2B2Tau_M-350.root --outFile output/skim.root --mass 350 --sample GluGluToBulkGraviton --year 2018 >& EventInfo.txt
python Common/SaveHisto.txt --inFile $CENTRAL_STORAGE/prod_v1/nanoAOD/2018/GluGluToBulkGravitonToHHTo2B2Tau_M-350.root --outFile output/skim.root
```
