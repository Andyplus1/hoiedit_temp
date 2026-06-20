# CR v7 Evaluation Release Notes

This repository is a lightweight open-source release assembled from the portable `cr_eval_release_v7_lite` package.

## Included

- Evaluation entry scripts: `run_qa_hoi.sh` and `run_eval.sh`
- CR v7 annotation JSON files under `data_v7/CR/`
- QA, HOI, resizing, scoring, and path utility scripts under `evaluation/`
- SAM2 source code needed by the evaluation wrapper
- GroundingDINO source code needed by the evaluation wrapper
- Environment requirement files and local configuration template under `env/`

## Not Included

Large or user-specific artifacts are intentionally not committed:

- Original CR images: `data_v7/CR/data_v7_L12/`, `data_v7/CR/data_v7_L3/`
- Edited frames: `data_v7/CR/<model>_frames/`
- GroundingDINO weights: `third_party/GroundingDINO/weights/`
- SAM2 checkpoints: `sam2/checkpoints/`
- Runtime outputs under `eval_runs/`
- Local credentials and paths in `env/local.conf`

Use `env/local.conf.example` as the template for local paths and API keys.

## Source Integration Note

The release was prepared from the available lite tarball:

```text
cr_eval_release_v7_lite.tar.gz
```

The additional requested source path:

```text
/network_space/server127_2/shared/gjy/cr_eval_release
```

was not mounted or readable in the current environment at packaging time. The packaged scripts already use `env/workspace.conf` to resolve paths relative to the repository root, so no algorithmic code changes were made.
