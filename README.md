# Taming I2V models for Image HOI Editing: A Cognitive Benchmark and Agentic Self-Correcting Framework

Official code release for **HOI-Edit**, **HOI-Eval**, and **SCPE**.

Current image editing methods excels at static attributes but fails at complex Human-Object Interactions (HOI), a critical challenge unaddressed by existing benchmarks that conflate HOI with static attributes, relying on global metrics incapable of simultaneously assessing dynamic interaction validity and entangled human-object pair preservation. Thus, we first introduce **HOI-Edit**, a comprehensive benchmark with three progressive cognitive levels, which features an automated metric **HOI-Eval** that first reliably evaluates instance-level interaction by letting VLM Q&A after thinking with images containing grounded Human-Object pair. Considering the task's essence of remodeling dynamic relationships, we benchmark Image-to-Video (I2V) models, finding them inherently suited for dynamic editing due to their temporal generation capabilities. Crucially, beyond superior performance, this capability provides a "replay of the failure process", offering unique diagnosability into why errors occur. We thus propose **SCPE (Self-Correcting Process Editing)**, a novel, agentic self-correcting framework that constrains the generation of I2V models through iteratively refined prompts, enabling the generated videos to more accurately present the target HOI. Extracted frames from these videos are the final editing results. On HOI-Edit, SCPE achieves performance competitive with state-of-the-art (SOTA) editing models like Nano Banana on interaction.

> This GitHub repository contains code and annotation JSON files. Large assets are hosted separately. See [ASSETS.md](ASSETS.md).

Chinese version: [README_zh.md](README_zh.md)

## News

- **Code release**: SCPE generation code and HOI-Eval evaluation code are available.
- **Benchmark assets**: original images and model checkpoints are packaged separately for Baidu Netdisk release.
- **Lite GitHub repo**: API keys, original images, reference videos, generated videos, edited frames, model weights, checkpoints, and runtime outputs are not committed.

## HOI-Edit and SCPE Overview

```text
Source image + short HOI instruction
        │
        ▼
SCPE (Self-Correcting Process Editing)
ACE / Playbook prompt enhancement
        │
        ▼
Wan2.2 image-to-video generation
        │
        ▼
QA2 frame selection
        │
        ▼
HOI-Eval on HOI-Edit
        ├─ Gemini QA
        ├─ HOI check
        └─ I / S / O / IQA scoring
```

## Repository Layout

```text
.
├── scpe/                      # SCPE generation pipeline
├── run_eval.sh                # HOI-Eval: QA + HOI + final scoring
├── run_qa_hoi.sh              # HOI-Eval: QA + HOI only
├── data_v7/CR/                # HOI-Edit annotation JSON files
├── evaluation/                # QA, HOI, preprocessing, scoring scripts
├── sam2/                      # SAM2 tracking code; checkpoint downloaded separately
├── third_party/GroundingDINO/ # GroundingDINO code; weights downloaded separately
└── env/                       # HOI-Eval config template and requirements
```

## Download Assets

The full asset package corresponding to this code release is described in [ASSETS.md](ASSETS.md).

After downloading and extracting the asset package, place files at:

```text
data_v7/CR/data_v7_L12/
data_v7/CR/data_v7_L3/
third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth
sam2/checkpoints/sam2.1_hiera_large.pt
```

## SCPE: Environment Setup

SCPE stands for **Self-Correcting Process Editing** and lives in [scpe/](scpe). It uses Gemini for ACE / Playbook learning and QA2. It can use either DashScope Wan2.2 I2V or a local Wan2.2 I2V A14B checkout for video generation.

```bash
cd scpe
./setup_env.sh
cp env.example env.local
```

Edit `scpe/env.local`:

```bash
export GEMINI_API_KEY="your-gemini-api-key"
export DASHSCOPE_API_KEY="your-dashscope-api-key"  # DashScope backend only
export ACE_LANG="cn"
export DATA_ROOT="/path/to/CameraReady"
export WAN22_REPO="/path/to/Wan2.2"
export WAN22_CKPT_DIR="/path/to/Wan2.2-I2V-A14B"
```

`DATA_ROOT` should contain annotation JSON, source images, and epoch0 reference videos.

## SCPE: Run

```bash
cd scpe
source env.local

LIMIT=2 ./run_minimal.sh all
./run_minimal.sh learn
./run_minimal.sh enhance
./run_minimal.sh wan22
./run_minimal.sh qa2
WAN22_BACKEND=local ./run_minimal.sh wan22
```

## HOI-Eval: Environment Setup

```bash
cp env/local.conf.example env/local.conf
```

Edit `env/local.conf`:

```bash
export DINO_ENV_PY="/path/to/conda/envs/cr-dino/bin/python"
export SS_ENV_PY="/path/to/conda/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="/path/to/conda/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

Dependency notes are in [env/requirements/README.md](env/requirements/README.md).

## HOI-Eval: Run

Place edited frames at:

```text
data_v7/CR/<model_name>_frames/L1L2/
data_v7/CR/<model_name>_frames/L3/
```

Then run:

```bash
MODELS=<model_name> GPU_ID=0 bash run_eval.sh
```

If edited frames are outside the repository:

```bash
FRAMES_DIR=/path/to/frames MODELS=<model_name> bash run_eval.sh
```

Partial runs:

```bash
SKIP_HOI=1 MODELS=<model_name> bash run_qa_hoi.sh
SKIP_QA=1 MODELS=<model_name> GPU_ID=0 bash run_qa_hoi.sh
SCORES_ONLY=1 SCORE_MODEL=<model_name> bash run_eval.sh
```

## Paper-To-Code Map

| Paper component | Code / files |
|---|---|
| SCPE / ACE prompt enhancement | `scpe/scripts/ace_i2v_official3.py`, `scpe/data/*playbook*`, `scpe/data/ace_prompts_*.json` |
| Wan2.2 I2V generation | `scpe/scripts/wan22_generate_from_enhanced_prompts.py`, `scpe/scripts/wan22_local_i2v_a14b_generate.py` |
| QA2 frame selection | `scpe/scripts/ace_v2f_qa2.py`, `scpe/data/qa2_prompts_*.json` |
| HOI-Edit annotations | `data_v7/CR/*_scoring_final.json` |
| Gemini QA evaluation | `evaluation/run_qa_gemini_question_v6.sh`, `evaluation/run_question_answering.py` |
| HOI-Eval interaction check | `evaluation/run_full_eval_v7_google.sh`, `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` |
| Final scoring | `evaluation/compute_scoring_final_scores.py` |

## Metrics

| Metric | Source |
|---|---|
| `I` | HOI interaction confidence, `max_yes_confidence` |
| `S` | Subject preservation, `subject_similarity` |
| `O` | Object preservation, `object_similarity` |
| `QA` | Gemini answer confidence for generated questions |
| `IQA` | `I` for L1 / initial-position L2; otherwise `min(I, QA)` |
