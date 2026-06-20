# Taming I2V models for Image HOI Editing: A Cognitive Benchmark and Agentic Self-Correcting Framework

Official code release for **HOI-Edit** and **SCPE**.

Current image editing methods excels at static attributes but fails at complex Human-Object Interactions (HOI), a critical challenge unaddressed by existing benchmarks that conflate HOI with static attributes, relying on global metrics incapable of simultaneously assessing dynamic interaction validity and entangled human-object pair preservation. Thus, we first introduce **HOI-Edit**, a comprehensive benchmark with three progressive cognitive levels, which features an automated metric **HOI-Eval** that first reliably evaluates instance-level interaction by letting VLM Q&A after thinking with images containing grounded Human-Object pair. Considering the task's essence of remodeling dynamic relationships, we benchmark Image-to-Video (I2V) models, finding them inherently suited for dynamic editing due to their temporal generation capabilities. Crucially, beyond superior performance, this capability provides a "replay of the failure process", offering unique diagnosability into why errors occur. We thus propose **SCPE (Self-Correcting Process Editing)**, a novel, agentic self-correcting framework that constrains the generation of I2V models through iteratively refined prompts, enabling the generated videos to more accurately present the target HOI. Extracted frames from these videos are the final editing results. On HOI-Edit, SCPE achieves performance competitive with state-of-the-art (SOTA) editing models like Nano Banana on interaction.

> This is a lightweight release. API keys, original images, reference videos, generated videos, edited frames, model weights, checkpoints, and runtime outputs are not included.

Chinese version: [README_zh.md](README_zh.md)

## News

- **Code release**: SCPE generation code and HOI-Eval evaluation code are available.
- **Lite benchmark files**: HOI-Edit annotation JSON files are included.
- **Local assets required**: images, videos, Wan2.2 weights, GroundingDINO weights, and SAM2 checkpoints should be prepared locally.

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
│   ├── scripts/               # ACE, Wan2.2 generation, QA2 frame selection
│   ├── data/                  # Playbook seeds and prompt templates
│   ├── run_minimal.sh         # SCPE one-command runner
│   ├── env.example            # SCPE local config template
│   └── README.md              # Detailed SCPE documentation
├── run_eval.sh                # HOI-Eval: QA + HOI + final scoring
├── run_qa_hoi.sh              # HOI-Eval: QA + HOI only
├── data_v7/CR/                # HOI-Edit annotation JSON files
├── evaluation/                # QA, HOI, preprocessing, scoring scripts
├── sam2/                      # SAM2 tracking code
├── third_party/GroundingDINO/ # GroundingDINO code
└── env/                       # HOI-Eval config template and requirements
```

## SCPE: Environment Setup

SCPE stands for **Self-Correcting Process Editing** and lives in [scpe/](scpe). It uses Gemini for ACE / Playbook learning and QA2. It can use either DashScope Wan2.2 I2V or a local Wan2.2 I2V A14B checkout for video generation.

```bash
cd scpe

# Create Python environment and install dependencies
./setup_env.sh

# Create local config. Do not commit env.local.
cp env.example env.local
```

Edit `scpe/env.local`:

```bash
export GEMINI_API_KEY="your-gemini-api-key"

# Required only for DashScope backend
export DASHSCOPE_API_KEY="your-dashscope-api-key"

# cn or en prompt / Playbook templates
export ACE_LANG="cn"

# Dataset root: annotations, source images, and epoch0 reference videos
export DATA_ROOT="/path/to/CameraReady"

# Required only for local Wan2.2 backend
export WAN22_REPO="/path/to/Wan2.2"
export WAN22_CKPT_DIR="/path/to/Wan2.2-I2V-A14B"
```

`DATA_ROOT` should follow this layout:

```text
DATA_ROOT/
├── collected_annotations_bboxes_v7_L1L2_questions.json
├── collected_annotations_bboxes_v7_L3_questions.json
├── data_v7_L12/
├── data_v7_L3/
├── epoch_0_L1L2/
└── epoch_0_L3/
```

## SCPE: Run

Minimal end-to-end run:

```bash
cd scpe
source env.local

# Debug run: process 2 samples per split
LIMIT=2 ./run_minimal.sh all
```

Run stages separately:

```bash
./run_minimal.sh learn      # learn / update Playbook from epoch0 reference videos
./run_minimal.sh enhance    # generate enhanced prompts from source image + instruction
./run_minimal.sh wan22      # generate videos with Wan2.2
./run_minimal.sh qa2        # select frames from generated videos
```

Use local Wan2.2 A14B:

```bash
WAN22_BACKEND=local ./run_minimal.sh wan22
```

Useful resume/debug flags:

```bash
SKIP_LEARN=1 ./run_minimal.sh enhance
SKIP_QA2=1 ./run_minimal.sh all
FORCE_REGEN=1 ./run_minimal.sh wan22
ACE_LANG=en ./run_minimal.sh all
```

SCPE outputs:

```text
scpe/output/
├── playbook.json
├── playbook_normalized.json
├── enhanced_prompts.json
├── wan22_videos/             # DashScope backend
├── wan22_local_a14b/         # local Wan2.2 backend
└── qa2_frames/
```

## HOI-Eval: Environment Setup

Create local config:

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

The full evaluation uses three environments:

| Env | Used for | Requirements |
|---|---|---|
| `cr-dino` | GroundingDINO detection | `env/requirements-dino.txt` |
| `cr-sam2` | SAM2 tracking | `env/requirements-sam2.txt` |
| `cr-gemini` | Gemini QA + HOI check + scoring | `env/requirements-hoi-google.txt` |

More dependency notes are in [env/requirements/README.md](env/requirements/README.md).

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

`FRAMES_DIR` should contain `L1L2/` and `L3/`.

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
| HOI preprocessing | `evaluation/resize_edited_images_to_original.py`, `evaluation/inference_on_multi_image_eval_optimized.py`, `sam2/run_sam2_tracking_for_eval.py` |
| Final scoring | `evaluation/compute_scoring_final_scores.py` |

## Required Local Assets

| Asset | Expected path |
|---|---|
| L1/L2 original images | `data_v7/CR/data_v7_L12/` |
| L3 original images | `data_v7/CR/data_v7_L3/` |
| Edited frames for HOI-Eval | `data_v7/CR/<model>_frames/L1L2/`, `data_v7/CR/<model>_frames/L3/` |
| GroundingDINO weight | `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth` |
| SAM2 checkpoint | `sam2/checkpoints/sam2.1_hiera_large.pt` |
| SCPE reference videos / source images | configured by `scpe/env.local` via `DATA_ROOT` |
| Local Wan2.2 repository and weights | configured by `WAN22_REPO` and `WAN22_CKPT_DIR` |

## Evaluation Outputs and Metrics

| Output | Path |
|---|---|
| HOI-Eval QA results | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| HOI-Eval HOI results | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |
| Final score table | `eval_runs/scoring_final_scores_4dp.json` |

`evaluation/compute_scoring_final_scores.py` merges HOI and QA outputs:

| Metric | Source |
|---|---|
| `I` | HOI interaction confidence, `max_yes_confidence` |
| `S` | Subject preservation, `subject_similarity` |
| `O` | Object preservation, `object_similarity` |
| `QA` | Gemini answer confidence for generated questions |
| `IQA` | `I` for L1 / initial-position L2; otherwise `min(I, QA)` |

## Notes

- `env/local.conf` and `scpe/env.local` are ignored by git because they contain local paths and API keys.
- `scpe/output/`, `scpe/logs/`, and `eval_runs/` are ignored runtime outputs.
- See [RELEASE_NOTES.md](RELEASE_NOTES.md) and [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md) for release scope and third-party code notes.
