# CR v7 Evaluation Code

Lite release for the CR v7 evaluation used in the paper. The code is organized by the paper evaluation pipeline: **CR data**, **QA evaluation**, **HOI check**, and **final scoring**.

No API keys are included. Original images, edited frames, model weights, checkpoints, and runtime outputs are also excluded from this lite release.

Chinese version: [README_zh.md](README_zh.md)

## Paper-To-Code Map

| Paper component | What it does | Code / files |
|---|---|---|
| CR benchmark annotations | Stores image-level instructions, HOI tags, generated QA questions, and scoring metadata | `data_v7/CR/*_scoring_final.json` |
| Edited image inputs | Model outputs to be evaluated | `data_v7/CR/<model>_frames/{L1L2,L3}/` or `FRAMES_DIR=/path/to/frames` |
| Phase 1: QA evaluation | Runs Gemini VQA on generated questions (`question_v6`) | `evaluation/run_qa_gemini_question_v6.sh`, `evaluation/run_question_answering.py` |
| Phase 2: HOI check | Measures interaction success and subject/object preservation | `evaluation/run_full_eval_v7_google.sh`, `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` |
| HOI preprocessing | Resize, person/object detection, and tracking before HOI check | `evaluation/resize_edited_images_to_original.py`, `evaluation/inference_on_multi_image_eval_optimized.py`, `sam2/run_sam2_tracking_for_eval.py` |
| Final metric table | Merges QA + HOI outputs and computes I / S / O / IQA scores | `evaluation/compute_scoring_final_scores.py` |
| One-command reproduction | Runs QA, HOI, and scoring in sequence | `run_eval.sh` |

## Repository Layout

```text
cr_eval_release/
├── run_eval.sh                # QA + HOI + final scoring
├── run_qa_hoi.sh              # QA + HOI only
├── data_v7/CR/                # CR annotation JSON files
├── evaluation/                # QA, HOI, preprocessing, scoring scripts
├── sam2/                      # SAM2 tracking code
├── third_party/GroundingDINO/ # GroundingDINO code
├── env/                       # local config template and requirements
└── eval_runs/                 # generated outputs
```

## Setup

Create a local config:

```bash
cp env/local.conf.example env/local.conf
```

Fill in:

```bash
export DINO_ENV_PY="/path/to/conda/envs/cr-dino/bin/python"
export SS_ENV_PY="/path/to/conda/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="/path/to/conda/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

Full runs use three Python environments:

| Env | Used for | Requirements |
|---|---|---|
| `cr-dino` | GroundingDINO detection | `env/requirements-dino.txt` |
| `cr-sam2` | SAM2 tracking | `env/requirements-sam2.txt` |
| `cr-gemini` | Gemini QA + HOI check + scoring | `env/requirements-hoi-google.txt` |

More dependency notes are in [env/requirements/README.md](env/requirements/README.md).

## Required Local Assets

This repository intentionally does not include large or private assets:

| Asset | Expected path |
|---|---|
| L1/L2 original images | `data_v7/CR/data_v7_L12/` |
| L3 original images | `data_v7/CR/data_v7_L3/` |
| Edited frames | `data_v7/CR/<model>_frames/L1L2/`, `data_v7/CR/<model>_frames/L3/` |
| GroundingDINO weight | `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth` |
| SAM2 checkpoint | `sam2/checkpoints/sam2.1_hiera_large.pt` |

## Run

Run the full pipeline:

```bash
MODELS=<model_name> GPU_ID=0 bash run_eval.sh
```

Run QA + HOI without final scoring:

```bash
MODELS=<model_name> GPU_ID=0 bash run_qa_hoi.sh
```

Common partial runs:

```bash
SKIP_HOI=1 MODELS=<model_name> bash run_qa_hoi.sh
SKIP_QA=1 MODELS=<model_name> GPU_ID=0 bash run_qa_hoi.sh
SCORES_ONLY=1 SCORE_MODEL=<model_name> bash run_eval.sh
```

If edited frames are outside the repository:

```bash
FRAMES_DIR=/path/to/frames MODELS=<model_name> bash run_eval.sh
```

`FRAMES_DIR` should contain `L1L2/` and `L3/`.

## Outputs

| Output | Path |
|---|---|
| QA results | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| HOI results | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |
| Final score table | `eval_runs/scoring_final_scores_4dp.json` |

## Metrics

`compute_scoring_final_scores.py` merges HOI and QA outputs:

| Metric | Source |
|---|---|
| `I` | HOI interaction confidence, `max_yes_confidence` |
| `S` | Subject preservation, `subject_similarity` |
| `O` | Object preservation, `object_similarity` |
| `QA` | Gemini answer confidence for generated questions |
| `IQA` | `I` for L1 / initial-position L2; otherwise `min(I, QA)` |

## Notes

- `env/workspace.conf` resolves paths relative to the repository root, so the code can be moved without editing hardcoded paths.
- `env/local.conf` is ignored by git because it contains machine-specific paths and API keys.
- See [RELEASE_NOTES.md](RELEASE_NOTES.md) and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for release scope and third-party code notes.
