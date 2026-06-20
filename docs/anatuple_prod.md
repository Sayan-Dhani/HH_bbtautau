# Common analysis steps

Remarks:

- commands bellow assume that `ERA` variable is set. E.g.
    ```sh
    ERA=Run3_2022
    ```
    Alternatively you can add `ERA=Run3_2022; ...` in front of each command.

- `version` argument alows to produce different versions of the same task. In the command below `--version dev` is used for illustration purposes. You can replace it with your version naming.
- `--workflow` can be `htcondor` or `local`. It is recommended to develop and test locally and then switch to `htcondor` for production. In examples below `--workflow local` is used for illustration purposes.
- when running on `htcondor` it is recommended to add `--transfer-logs` to the command to transfer logs to local.
- `--customisations` argument is used to pass custom parameters to the task in form param1=value1,param2=value2,...
    IMPORTANT for HHbbTauTau analysis: if running using deepTau 2p5 add `--customisations deepTauVersion=2p5`
- if you want to run only on few files, you can specify list of branches to run using `--branches` argument. E.g. `--branches 2,7:10,17`.
- to get status, use `--print-stauts N,K` where N is depth for task dependencies, K is depths for file dependencies. E.g. `--print-status 3,1`.
- to remove task output use `--remove-output N,a`, where N is depth for task dependencies. E.g. `--remove-output 0,a`.
- it is highly recommended to limitate the maximum number of parallel jobs running adding `--parallel-jobs M` where M is the number of the parallel jobs (e.g. M=100)

The anaTuple production proceeds through the following chain of tasks. Each task
depends on the previous one, so running a downstream task automatically triggers
the upstream ones:

`InputFileTask` → `AnaTupleFileTask` → `AnaTupleFileListBuilderTask` → `AnaTupleFileListTask` → `AnaTupleMergeTask`

> **Note:** the old `AnaCacheTask` step has been **removed**. The denominator
> used for MC normalization is now computed inside `AnaTupleMergeTask` (see the
> last step below).

## Create input file list

```sh
law run InputFileTask --period ${ERA} --version dev
```

Lists the input nanoAOD files of each dataset and writes one JSON file list per dataset.

## Create per-file anaTuples

For each input nanoAOD file an anaTuple is produced, together with a JSON report
holding its metadata (denominator contributions, number of events, validity, ...).

```sh
law run AnaTupleFileTask --period ${ERA} --version dev
```

## Build the merge plan

Defines the list of final (merged) anaTuples and merges the individual per-file
reports into a per-dataset plan.

```sh
law run AnaTupleFileListBuilderTask --period ${ERA} --version dev
```

`AnaTupleFileListTask` then copies the resulting file list locally (it is a
lightweight local task that is pulled in automatically by the merge step):

```sh
law run AnaTupleFileListTask --period ${ERA} --version dev
```

## Merge anaTuples

Merges the per-file anaTuples according to the plan **and computes the
denominator used for MC normalization** (this is the step that replaces the old
`AnaCacheTask`).

```sh
law run AnaTupleMergeTask --period ${ERA} --version dev
```

- note: It's very important to first run `InputFileTask`; the other task
  dependencies are then resolved automatically (e.g. running `AnaTupleMergeTask`
  will first run `AnaTupleFileTask`, `AnaTupleFileListBuilderTask` and
  `AnaTupleFileListTask` as needed). If you do not run `InputFileTask`, running
  other tasks will raise an error.