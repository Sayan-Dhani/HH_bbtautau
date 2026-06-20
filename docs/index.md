# FLAF

FLAF - Flexible LAW-based Analysis Framework.
Task workflow managed is done via [LAW](https://github.com/riga/law) (Luigi Analysis Framework).

## How to install
1. Setup ssh keys:
    - On GitHub [settings/keys](https://github.com/settings/keys)
    - On CERN GitLab [profile/keys](https://gitlab.cern.ch/-/profile/keys)

1. Clone the repository:
  ```sh
  git clone --recursive git@github.com:cms-flaf/HH_bbtautau.git HH_bbtautau
  ```

1. Create a user customisation file `config/user_custom.yaml`. It should contain all user-specific modifications that you don't want to be committed to the central repository. Below is example of minimal content of the file (replace `USER_NAME` and `ANA_FOLDER` with your values):
    ```yaml
    fs_default:
        - 'T3_CH_CERNBOX:/store/user/USER_NAME/ANA_FOLDER/'
    fs_anaTuple:
        - 'T3_CH_CERNBOX:/store/user/USER_NAME/ANA_FOLDER/'
    fs_anaCacheTuple:
        - 'T3_CH_CERNBOX:/store/user/USER_NAME/ANA_FOLDER/'
    fs_HistTuple:
        - 'T3_CH_CERNBOX:/store/user/USER_NAME/ANA_FOLDER/'
    fs_plots:
        - 'T3_CH_CERNBOX:/store/user/USER_NAME/ANA_FOLDER/plots/'
    fs_json:
        - 'T3_CH_CERNBOX:/store/user/USER_NAME/ANA_FOLDER/jsonFiles/'
    analysis_config_area: config
    compute_unc_variations: true
    compute_unc_histograms: true
    store_noncentral: true
    variables:
    - b1_pt
    - bb_m_vis
    ```
    Notes:

    - Any `fs_*` location that you do not define falls back to `fs_default`, so
      at minimum you only need `fs_default`. The locations actually used by the
      current workflow are `fs_anaTuple`, `fs_anaCacheTuple`, `fs_HistTuple` and
      `fs_plots` (the old `fs_anaCache` / `fs_histograms` keys are no longer
      used).
    - The variables to histogram are listed under the `variables` key. Variables
      that are produced by a payload producer (configured under
      `payload_producers` in `config/global.yaml`, e.g. the ggF DNN score) are
      computed by `AnalysisCacheTask` and pulled in automatically when requested
      — there is no longer a `need_cache` flag.

## How to load environment
1. Following command activates the framework environment:
    ```sh
    source env.sh
    ```

1. For the new installation or after you implement new law tasks, you need to update the law index:
    ```sh
    law index --verbose
    ```

1. Initialize voms proxy:
    ```sh
    voms-proxy-init -voms cms -rfc -valid 192:00
    ```

