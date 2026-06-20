# HOI-Edit / CR v7 Code Release

This repository is a temporary open-source release for the HOI editing paper code. It contains two parts:

- **SCPE generation pipeline**: ACE/Playbook prompt enhancement, Wan2.2 image-to-video generation, and QA2 frame selection.
- **CR v7 evaluation pipeline**: CR annotations, Gemini QA, HOI check, and final I / S / O / IQA scoring.

> The repository is organized as a lightweight code release. API keys, original images, edited frames/videos, model weights, checkpoints, and runtime outputs are not included.

Chinese version: [README_zh.md](README_zh.md)

## News

- **Code release**: SCPE pipeline and CR v7 evaluation code are provided.
- **Lite assets**: annotation JSON files are included; large images, videos, weights, and checkpoints should be prepared locally.

## Overview

```text
Short HOI instruction + source image
        │
        ▼
SCPE / ACE prompt enhancement
        │  scpe/
        ▼
Wan2.2 image-to-video generation
        │
        ▼
QA2 frame selection / edited frame extraction
        │
        ▼
CR v7 evaluation
        ├─ Phase 1: Gemini QA
        ├─ Phase 2: HOI check
        └─ Phase 3: final I / S / O / IQA scoring
```

## Paper-To-Code Map

| Paper component | What it does | Code / files |
|---|---|---|
| SCPE / ACE prompt enhancement | Learns a Playbook from failure cases and generates detailed I2V prompts from short HOI instructions | `scpe/scripts/ace_i2v_official3.py`, `scpe/data/*playbook*`, `scpe/data/ace_prompts_*.json` |
| Wan2.2 I2V generation | Converts enhanced prompts and source images into videos using DashScope or local Wan2.2 A14B | `scpe/scripts/wan22_generate_from_enhanced_prompts.py`, `scpe/scripts/wan22_local_i2v_a14b_generate.py` |
| QA2 frame selection | Selects the best frame from generated videos for downstream image evaluation | `scpe/scripts/ace_v2f_qa2.py`, `scpe/data/qa2_prompts_*.json` |
| CR benchmark annotations | Stores image-level instructions, HOI tags, generated QA questions, and scoring metadata | `data_v7/CR/*_scoring_final.json` |
| Edited image inputs | Model outputs or QA2-selected frames to be evaluated | `data_v7/CR/<model>_frames/{L1L2,L3}/` or `FRAMES_DIR=/path/to/frames` |
| Phase 1: QA evaluation | Runs Gemini VQA on generated questions (`question_v6`) | `evaluation/run_qa_gemini_question_v6.sh`, `evaluation/run_question_answering.py` |
| Phase 2: HOI check | Measures interaction success and subject/object preservation | `evaluation/run_full_eval_v7_google.sh`, `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` |
| HOI preprocessing | Resize, person/object detection, and tracking before HOI check | `evaluation/resize_edited_images_to_original.py`, `evaluation/inference_on_multi_image_eval_optimized.py`, `sam2/run_sam2_tracking_for_eval.py` |
| Final metric table | Merges QA + HOI outputs and computes I / S / O / IQA scores | `evaluation/compute_scoring_final_scores.py` |
| One-command evaluation | Runs QA, HOI, and scoring in sequence | `run_eval.sh` |

## Repository Layout

```text
.
├── scpe/                      # SCPE generation pipeline
├── run_eval.sh                # CR evaluation: QA + HOI + final scoring
├── run_qa_hoi.sh              # CR evaluation: QA + HOI only
├── data_v7/CR/                # CR annotation JSON files
├── evaluation/                # QA, HOI, preprocessing, scoring scripts
├── sam2/                      # SAM2 tracking code
├── third_party/GroundingDINO/ # GroundingDINO code
├── env/                       # evaluation config template and requirements
└── eval_runs/                 # generated evaluation outputs
```

## SCPE Quick Start

See [scpe/README.md](scpe/README.md) for the full SCPE pipeline. Minimal usage:

```bash
cd scpe
cp env.example env.local
# edit env.local: GEMINI_API_KEY, optional DASHSCOPE_API_KEY, DATA_ROOT, WAN22 paths

LIMIT=2 ./run_minimal.sh all
```

Useful SCPE commands:

```bash
./run_minimal.sh learn      # learn Playbook from epoch0 reference videos
./run_minimal.sh enhance    # generate enhanced prompts
./run_minimal.sh wan22      # generate videos with Wan2.2
./run_minimal.sh qa2        # select/evaluate frames from generated videos
WAN22_BACKEND=local ./run_minimal.sh wan22
```

## CR Evaluation Quick Start

Create local evaluation config:

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

Run the full CR evaluation:

```bash
MODELS=<model_name> GPU_ID=0 bash run_eval.sh
```

If edited frames are outside the repository:

```bash
FRAMES_DIR=/path/to/frames MODELS=<model_name> bash run_eval.sh
```

`FRAMES_DIR` should contain `L1L2/` and `L3/`.

## Required Local Assets

| Asset | Expected path |
|---|---|
| L1/L2 original images | `data_v7/CR/data_v7_L12/` |
| L3 original images | `data_v7/CR/data_v7_L3/` |
| Edited frames for CR evaluation | `data_v7/CR/<model>_frames/L1L2/`, `data_v7/CR/<model>_frames/L3/` |
| GroundingDINO weight | `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth` |
| SAM2 checkpoint | `sam2/checkpoints/sam2.1_hiera_large.pt` |
| SCPE reference videos / source images | configured by `scpe/env.local` via `DATA_ROOT` |
| Local Wan2.2 repository and weights | configured by `WAN22_REPO` and `WAN22_CKPT_DIR` |

## Outputs

| Output | Path |
|---|---|
| SCPE enhanced prompts | `scpe/output/enhanced_prompts.json` |
| SCPE generated videos | `scpe/output/wan22_videos/` or `scpe/output/wan22_local_a14b/` |
| QA2 selected frames | `scpe/output/qa2_frames/` |
| CR QA results | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| CR HOI results | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |
| Final score table | `eval_runs/scoring_final_scores_4dp.json` |

## Metrics

`evaluation/compute_scoring_final_scores.py` merges HOI and QA outputs:

| Metric | Source |
|---|---|
| `I` | HOI interaction confidence, `max_yes_confidence` |
| `S` | Subject preservation, `subject_similarity` |
| `O` | Object preservation, `object_similarity` |
| `QA` | Gemini answer confidence for generated questions |
| `IQA` | `I` for L1 / initial-position L2; otherwise `min(I, QA)` |

## Notes

- `env/workspace.conf` resolves evaluation paths relative to the repository root.
- `env/local.conf` and `scpe/env.local` are ignored by git because they contain machine-specific paths and API keys.
- See [RELEASE_NOTES.md](RELEASE_NOTES.md) and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for release scope and third-party code notes.
