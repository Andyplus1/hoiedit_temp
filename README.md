# CR v7 Evaluation Release

Portable CR v7 evaluation package. Both **QA** and **HOI Check** use the **Google Gemini 2.5 Pro API** via the official `google-genai` SDK. No API keys are bundled — configure your own before running.

This is a **lite code release**: annotation JSON files and evaluation code are included, while original images, edited frames, model weights, checkpoints, runtime outputs, and local credentials are intentionally excluded. See [RELEASE_NOTES.md](RELEASE_NOTES.md).

**Example path:** `/path/to/cr_eval_release`

Also available in Chinese: [README_zh.md](README_zh.md)

---

## Directory Layout

```
cr_eval_release/
├── run_qa_hoi.sh              # Main entry: QA + HOI
├── run_eval.sh                # Full pipeline: QA + HOI + scoring
├── README.md                  # English (this file)
├── README_zh.md               # Chinese
│
├── env/
│   ├── workspace.conf         # Workspace paths (auto-detected)
│   ├── local.conf.example     # Machine-specific config template
│   ├── setup_check.sh         # Quick sanity check
│   ├── verify_scripts.sh      # Full check: files, weights, syntax, secrets
│   ├── requirements-dino.txt
│   ├── requirements-sam2.txt
│   ├── requirements-hoi-google.txt
│   └── requirements/README.md
│
├── data_v7/CR/                # Annotation JSON files; add images locally
│   ├── collected_annotations_bboxes_v7_L1L2_questions_scoring_final.json  (499)
│   ├── collected_annotations_bboxes_v7_L3_questions_scoring_final.json    (136)
│   ├── data_v7_L12/           # You provide: L1L2 originals (499)
│   ├── data_v7_L3/            # You provide: L3 originals (136 keys, 143 files)
│   └── <model>_frames/        # You provide: edited frames per model
│       ├── L1L2/
│       └── L3/
│
├── evaluation/                # Evaluation scripts
├── sam2/                      # SAM2 tracking code; add checkpoint locally
├── third_party/GroundingDINO/ # Object detection code; add weights locally
└── eval_runs/                 # Outputs (QA / HOI / scores)
```

---

## Quick Start

### 1. Copy the release

```bash
cp -r cr_eval_release /your/target/path
cd /your/target/path/cr_eval_release
```

### 2. Configure the machine

```bash
cp env/local.conf.example env/local.conf
# Edit local.conf: three Python paths + GEMINI_API_KEY
```

Minimum `local.conf`:

```bash
export DINO_ENV_PY="/path/to/conda/envs/cr-dino/bin/python"
export SS_ENV_PY="/path/to/conda/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="/path/to/conda/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

### 3. Install Conda environments (three required for full pipeline)

**Yes — you need three separate Conda environments** for a full QA + HOI run. The shell scripts call a different Python interpreter at each stage via `env/local.conf`:

| Env | `local.conf` variable | Used in pipeline | Requirements |
|-----|----------------------|------------------|--------------|
| `cr-dino` | `DINO_ENV_PY` | HOI Step 1 — GroundingDINO detection | `env/requirements-dino.txt` + compile `third_party/GroundingDINO` |
| `cr-sam2` | `SS_ENV_PY` | HOI Step 2 — SAM2 tracking | `env/requirements-sam2.txt` |
| `cr-gemini` | `GOOGLE_ENV_PY` | Phase 1 QA + HOI Step 3 Gemini API | `env/requirements-hoi-google.txt` |

Why three envs instead of one? PyTorch/CUDA builds, GroundingDINO compiled extensions, SAM2, and `google-genai` have overlapping but incompatible dependency pins. Keeping them separate avoids install conflicts and matches how `run_qa_hoi.sh` invokes each step.

**Partial runs need fewer envs:**

| What you run | Environments needed |
|--------------|---------------------|
| Full `run_qa_hoi.sh` (QA + HOI) | all three |
| QA only (`SKIP_HOI=1`) | `cr-gemini` only |
| HOI only, DINO/SAM2 already done (`HOI_CHECK_ONLY=1`) | `cr-gemini` only |
| HOI only, full preprocessing (`SKIP_QA=1`) | all three |

#### 3a. `cr-dino` (GroundingDINO)

Code: `third_party/GroundingDINO/`  
Weights (not bundled in lite release): `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth`  
Inference script: `evaluation/inference_on_multi_image_eval_optimized.py`

```bash
conda create -n cr-dino python=3.10 -y
conda activate cr-dino
pip install -r env/requirements-dino.txt
pip install -r third_party/GroundingDINO/requirements.txt
cd third_party/GroundingDINO && pip install -e .
python -c "import torch; print('cuda:', torch.cuda.is_available())"
```

#### 3b. `cr-sam2` (SAM2)

Code: `sam2/`  
Weights (not bundled in lite release): `sam2/checkpoints/sam2.1_hiera_large.pt`  
Tracking script: `sam2/run_sam2_tracking_for_eval.py`

```bash
conda create -n cr-sam2 python=3.10 -y
conda activate cr-sam2
pip install -r env/requirements-sam2.txt
# If checkpoint missing: bash sam2/checkpoints/download_ckpts.sh
```

#### 3c. `cr-gemini` (Gemini QA + HOI Check)

Scripts: `evaluation/run_question_answering.py`, `evaluation/gemini3_*_google_newsim.py`

```bash
conda create -n cr-gemini python=3.11 -y
conda activate cr-gemini
pip install -r env/requirements-hoi-google.txt
python -c "from google import genai; print('google-genai OK')"
```

#### 3d. Wire paths in `local.conf`

After creating the envs, point `local.conf` to each interpreter (replace with your conda paths):

```bash
export DINO_ENV_PY="$HOME/miniconda3/envs/cr-dino/bin/python"
export SS_ENV_PY="$HOME/miniconda3/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="$HOME/miniconda3/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

More detail: [env/requirements/README.md](env/requirements/README.md)

### 4. Add edited frames

Pick any **name** for your model (e.g. the checkpoint or experiment id you already use). Put edited images at:

```
data_v7/CR/<name>_frames/L1L2/
data_v7/CR/<name>_frames/L3/
```

Run: `MODELS=<name> bash run_qa_hoi.sh`

If your frames live elsewhere, set `FRAMES_DIR=/path/to/frames` (must contain `L1L2/` and `L3/` subfolders). See **Paths you configure** below.

### 5. Verify and run

```bash
bash env/setup_check.sh
DRY_RUN=1 bash run_qa_hoi.sh
GPU_ID=0 bash run_qa_hoi.sh
```

---

## Paths you configure

You choose the **same name string** everywhere (examples below use `<name>` as a placeholder — replace it with yours).

| What | Where |
|------|--------|
| Edited frames (default) | `data_v7/CR/<name>_frames/L1L2/` and `.../L3/` |
| Edited frames (override) | `FRAMES_DIR=/your/frames/root` → must contain `L1L2/`, `L3/` |
| Run QA + HOI | `MODELS=<name>` (comma-separated for multiple) |
| QA output | `eval_runs/qa_results_v6_{L1L2,L3}_<name>.json` |
| HOI output | `eval_runs/<name>_{L1L2,L3}_full/.../results_<name>_*_google_full.json` |
| Scoring | Same `<name>` as above: `--model <name>` or `SCORE_MODEL=<name>` in `run_eval.sh` |

```bash
MODELS=<name> GPU_ID=0 bash run_qa_hoi.sh
# reads frames from data_v7/CR/<name>_frames/ unless FRAMES_DIR is set
```

**Different folder layout?**

- Symlink: `ln -s /your/frames data_v7/CR/<name>_frames`
- Or set `FRAMES_DIR=/your/frames` before running
- Or call HOI with explicit paths:

```bash
export EVAL_V7_IMAGE_DIR=/your/frames/L3
export EVAL_V7_JSON=data_v7/CR/collected_annotations_bboxes_v7_L3_questions_scoring_final.json
export EVAL_V7_ORIG_DIR=data_v7/CR/data_v7_L3
bash evaluation/run_full_eval_v7_google.sh --model <name> --datasets V7 --gpu-id 0
```

QA-only override when calling `run_question_answering.py` directly:  
`--model-edited-dir <name>=/your/path`

---

## Main Entry: `run_qa_hoi.sh`

Runs QA and HOI in sequence:

| Phase | Scripts | API | Output |
|-------|---------|-----|--------|
| 1 QA | `run_qa_gemini_question_v6.sh` → `run_question_answering.py` | Gemini 2.5 Pro | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| 2 HOI | `run_full_eval_v7_google.sh` → resize / DINO / SAM2 / Gemini | Gemini 2.5 Pro | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |

### Usage

```bash
MODELS=<name> GPU_ID=0 bash run_qa_hoi.sh
SPLITS=L3 MODELS=<name> bash run_qa_hoi.sh
SKIP_HOI=1 MODELS=<name> bash run_qa_hoi.sh
SKIP_QA=1 MODELS=<name> GPU_ID=0 bash run_qa_hoi.sh
SKIP_QA=1 HOI_CHECK_ONLY=1 MODELS=<name> GPU_ID=0 bash run_qa_hoi.sh
DRY_RUN=1 MODELS=<name> bash run_qa_hoi.sh
```

### Environment variables

| Variable | Default | Description |
|----------|---------|-------------|
| `GEMINI_API_KEY` | *(required)* | Google Gemini API key (QA + HOI) |
| `MODELS` | *(required)* | Comma-separated names; same string used in output paths (see **Paths you configure**) |
| `FRAMES_DIR` | empty | Optional root for edited frames (contains `L1L2/`, `L3/`); default `data_v7/CR/<name>_frames` |
| `SPLITS` | `L1L2,L3` | Dataset splits |
| `GPU_ID` | `0` | CUDA device |
| `SKIP_QA` | `0` | `1` to skip QA |
| `SKIP_HOI` | `0` | `1` to skip HOI |
| `HOI_CHECK_ONLY` | `0` | `1` to skip resize/DINO/SAM2 |
| `SKIP_CONVERT` | `1` | `0` to normalize edited frame names |
| `SKIP_RESIZE` | `0` | `1` to skip resize |
| `SKIP_DINO` | `0` | `1` to skip GroundingDINO |
| `SKIP_SAM_TRACK` | `0` | `1` to skip SAM2 |
| `MAX_PARALLEL_QA` | `2` | Parallel QA jobs |
| `CR_DEFAULT_PROXY_URL` | empty | Optional HTTP proxy for Google API |

---

## Full Pipeline: `run_eval.sh`

Adds **Phase 3 scoring** on top of `run_qa_hoi.sh`:

```bash
bash run_eval.sh                                 # QA + HOI + scoring
SCORES_ONLY=1 bash run_eval.sh                   # scoring only
SKIP_SCORES=1 bash run_eval.sh                   # skip scoring
LEGACY_SCORES_ROOT=/path/to/old bash run_eval.sh  # merge historical HOI/QA results
MODELS=<name> bash run_eval.sh                   # score the same name after QA+HOI
SCORE_MODEL=<name> SCORES_ONLY=1 bash run_eval.sh  # score only
```

Output: `eval_runs/scoring_final_scores_4dp.json`

---

## Phase 3: Final Scoring

Implemented by `evaluation/compute_scoring_final_scores.py`. It reads **HOI results + QA results** and aggregates them into the 5-models table metrics.

### How to run

```bash
# After QA + HOI (via run_eval.sh, or manually):
python evaluation/compute_scoring_final_scores.py \
  --workspace . \
  --model <name> \
  --decimals 4

# Score only, merge historical results from another repo:
LEGACY_SCORES_ROOT=/path/to/gjy SCORES_ONLY=1 bash run_eval.sh
```

`--model` must be the **same name** you passed to `MODELS=` when running QA/HOI. Comma-separated values score multiple names in one run.

### Inputs

Replace `<name>` with your chosen string:

| Source | Path (under `eval_runs/`) |
|--------|---------------------------|
| HOI (L1L2) | `<name>_L1L2_full/.../results_<name>_L1L2_google_full.json` |
| HOI (L3) | `<name>_L3_full/.../results_<name>_L3_google_full.json` |
| QA (L1L2) | `qa_results_v6_L1L2_<name>.json` |
| QA (L3) | `qa_results_v6_L3_<name>.json` |
| Annotations | `data_v7/CR/*_scoring_final.json` |

HOI failures (`has_error`, `processing_error`, etc.) are excluded from that sample.

### Per-sample metrics (from HOI + QA)

| Metric | Source | Meaning |
|--------|--------|---------|
| **I** | HOI `max_yes_confidence` | Interaction / edit quality (Gemini HOI Check) |
| **S** | HOI `subject_similarity` | Subject preservation |
| **O** | HOI `object_similarity` | Object preservation |
| **QA** | Phase 1 VQA | Yes → `confidence`; No → `0` (matched by `question_v6` + `tags_v6`) |
| **IQA** | Derived | See rule below |

**IQA rule:**

- **L1** tags or **`L2-...-pos_initial`**: `IQA = I`
- **Other L2 / L3** tags: `IQA = min(I, QA)` (both must be available)

### Aggregation (final table rows)

Annotations are split into two pools (same as the 5-models table):

- **L1/L2 block** → only `L1L2_scoring_final.json` (per-tag rows + `L1_average`, `L2_average`)
- **L3 block** → only `L3_scoring_final.json` (per-tag rows + `L3_average`)

For each tag row, **I / S / O / IQA** = mean over all samples carrying that tag in `tags_v6`.  
`L*_average` rows pool all samples with L1 / L2 / L3 tags respectively (IQA averaged per sample–tag pair).

Example output rows (printed and saved to JSON):

```
L1-interaction-relation_occur          n=...  I=...  IQA=...
L2-location_understanding-specific_pos_end  ...
L1_average                             ...
L2_average                             ...
L3-non_rigid_change                    ...
L3_average                             ...
```

---

## HOI Pipeline (Phase 2)

For each `(model, split)`, `run_full_eval_v7_google.sh` runs:

```
Step 0   convert_images_for_eval.py        [optional; skipped by default]
Step 0b  resize_edited_images_to_original.py
Step 1   inference_on_multi_image_eval_optimized.py  (GroundingDINO)
Step 2   sam2/run_sam2_tracking_for_eval.py
Step 3   gemini3_*_google_newsim.py        (HOI Check: I, S, O)
```

Advanced: run HOI shell directly:

```bash
export EVAL_V7_SPLIT_TAG=L3
export EVAL_V7_IMAGE_DIR=data_v7/CR/<name>_frames/L3
export EVAL_V7_JSON=data_v7/CR/collected_annotations_bboxes_v7_L3_questions_scoring_final.json
export EVAL_V7_ORIG_DIR=data_v7/CR/data_v7_L3

bash evaluation/run_full_eval_v7_google.sh \
  --model <name> \
  --datasets V7 \
  --gpu-id 0 \
  --output-dir eval_runs
```

---

## QA (Phase 1)

Reads `question_v6` from the scoring JSON and runs Yes/No VQA on each **edited** frame.

Output: `eval_runs/qa_results_v6_L1L2_<model>.json`

---

## Environment Setup

### `env/workspace.conf`

Auto-sets `EVAL_WORKSPACE`, `DATA_V7_CR`, `EVAL_DIR`, `GROUNDING_DINO_ROOT`, `SAM2_ROOT`, `EVAL_RUNS_DIR`.

### `env/local.conf`

Copy from `local.conf.example`. Set Python paths, `GEMINI_API_KEY`, optional `GPU_ID` and `CR_DEFAULT_PROXY_URL`.

**Do not commit `local.conf` to version control.**

### Proxy

If Google API access requires a proxy:

```bash
export CR_DEFAULT_PROXY_URL="http://127.0.0.1:7890"
```

Applied via `evaluation/cr_proxy_defaults.sh`. DINO inference temporarily disables proxy and uses an HF mirror for tokenizer downloads.

### Three Conda environments

Full install steps are in **Quick Start §3** and [env/requirements/README.md](env/requirements/README.md).

- **`cr-dino`** → `DINO_ENV_PY` — GroundingDINO (`third_party/GroundingDINO/`)
- **`cr-sam2`** → `SS_ENV_PY` — SAM2 (`sam2/`)
- **`cr-gemini`** → `GOOGLE_ENV_PY` — Gemini QA + HOI Check

All three are required for a full QA + HOI run. QA-only or HOI-check-only modes need only `cr-gemini`.

---

## Outputs

```
eval_runs/
├── qa_results_v6_L1L2_<model>.json
├── qa_results_v6_L3_<model>.json
├── <model>_L1L2_full/.../results_*_google_full.json
├── <model>_L3_full/.../results_*_google_full.json
└── scoring_final_scores_4dp.json
```

Logs: `evaluation/logs/run_qa_hoi_YYYYMMDD_HHMMSS/`

---

## Verification

```bash
bash env/verify_scripts.sh
```

Checks: entry scripts, data, weights, Python syntax, and that no hardcoded API keys are present in scripts.

---

## Script Reference

| Script | Role |
|--------|------|
| `run_qa_hoi.sh` | Main entry: QA + HOI |
| `run_eval.sh` | QA + HOI + scoring |
| `run_qa_gemini_question_v6.sh` | QA batch orchestration |
| `run_question_answering.py` | Gemini QA implementation |
| `run_full_eval_v7_google.sh` | HOI pipeline orchestration |
| `gemini3_*_google_newsim.py` | Gemini HOI Check |
| `resize_edited_images_to_original.py` | Resize edited frames to original size |
| `convert_images_for_eval.py` | Normalize edited frame names/formats |
| `inference_on_multi_image_eval_optimized.py` | GroundingDINO batch detection |
| `gdino_transformers_compat.py` | transformers 5.x compatibility patch |
| `compute_scoring_final_scores.py` | 5-models table scoring |
| `sam2/run_sam2_tracking_for_eval.py` | SAM2 tracking |
