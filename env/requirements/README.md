# Environment Setup Guide / 环境安装指南

Full QA + HOI evaluation requires **three separate Conda environments**.  
完整 QA + HOI 评测需要 **三个独立的 Conda 环境**。

Each pipeline stage is launched with a different Python interpreter configured in `env/local.conf`.  
流水线各阶段通过 `env/local.conf` 里不同的 Python 路径启动。

---

## Overview / 总览

| Env | `local.conf` | Pipeline stage | Requirements |
|-----|--------------|----------------|--------------|
| `cr-dino` | `DINO_ENV_PY` | HOI Step 1 — GroundingDINO | `env/requirements-dino.txt` + `pip install -e third_party/GroundingDINO` |
| `cr-sam2` | `SS_ENV_PY` | HOI Step 2 — SAM2 tracking | `env/requirements-sam2.txt` |
| `cr-gemini` | `GOOGLE_ENV_PY` | QA + HOI Step 3 — Gemini API | `env/requirements-hoi-google.txt` |

### When you need all three / 何时需要三个都装

| Run mode | Envs needed |
|----------|-------------|
| Full `run_qa_hoi.sh` | all three / 三个都要 |
| QA only (`SKIP_HOI=1`) | `cr-gemini` only |
| HOI check only (`HOI_CHECK_ONLY=1`) | `cr-gemini` only |
| Full HOI, skip QA (`SKIP_QA=1`) | all three / 三个都要 |

---

## 0. Configure `local.conf`

```bash
cp env/local.conf.example env/local.conf
```

---

## 1. `cr-dino` — GroundingDINO

**Bundled in this release / 发布包内已包含：**

- Source: `third_party/GroundingDINO/`
- Weights: `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth`
- Config: `third_party/GroundingDINO/groundingdino/config/GroundingDINO_SwinT_OGC.py`
- Called by: `evaluation/inference_on_multi_image_eval_optimized.py`

```bash
conda create -n cr-dino python=3.10 -y
conda activate cr-dino
pip install -r env/requirements-dino.txt
pip install -r third_party/GroundingDINO/requirements.txt
cd third_party/GroundingDINO && pip install -e .
```

Add to `local.conf`:
```bash
export DINO_ENV_PY="$CONDA_PREFIX/bin/python"
```

Verify:
```bash
python -c "import torch; from groundingdino.models import build_model; print('OK, cuda:', torch.cuda.is_available())"
```

> `pip install -e .` compiles CUDA extensions. Requires CUDA toolkit matching your PyTorch build.  
> `pip install -e .` 会编译 CUDA 扩展，需要与本机 PyTorch 匹配的 CUDA toolkit。

---

## 2. `cr-sam2` — SAM2 tracking

**Bundled in this release / 发布包内已包含：**

- Source: `sam2/`
- Weights: `sam2/checkpoints/sam2.1_hiera_large.pt`
- Called by: `sam2/run_sam2_tracking_for_eval.py`

```bash
conda create -n cr-sam2 python=3.10 -y
conda activate cr-sam2
pip install -r env/requirements-sam2.txt
```

Add to `local.conf`:
```bash
export SS_ENV_PY="$CONDA_PREFIX/bin/python"
```

If checkpoint is missing / 若权重缺失:
```bash
bash sam2/checkpoints/download_ckpts.sh
```

---

## 3. `cr-gemini` — Gemini QA + HOI Check

**Called by / 调用脚本：**

- `evaluation/run_question_answering.py` (QA)
- `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` (HOI)

```bash
conda create -n cr-gemini python=3.11 -y
conda activate cr-gemini
pip install -r env/requirements-hoi-google.txt
```

Add to `local.conf`:
```bash
export GOOGLE_ENV_PY="$CONDA_PREFIX/bin/python"
export GEMINI_API_KEY="your-key-here"
```

Optional proxy / 可选代理:
```bash
export CR_DEFAULT_PROXY_URL="http://127.0.0.1:7890"
```

Verify:
```bash
python -c "from google import genai; print('google-genai OK')"
```

---

## 4. Final check / 整体检查

```bash
bash env/setup_check.sh
DRY_RUN=1 bash run_qa_hoi.sh
```

---

## Requirements files / 依赖文件

| File | Env | Main packages |
|------|-----|---------------|
| `env/requirements-dino.txt` | cr-dino | torch, torchvision, transformers (<5), opencv |
| `env/requirements-sam2.txt` | cr-sam2 | torch, hydra, opencv |
| `env/requirements-hoi-google.txt` | cr-gemini | google-genai, pillow, opencv, tqdm |
