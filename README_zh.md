# Taming I2V models for Image HOI Editing: A Cognitive Benchmark and Agentic Self-Correcting Framework

这是 **HOI-Edit** 和 **SCPE** 的官方代码发布。

Current image editing methods excels at static attributes but fails at complex Human-Object Interactions (HOI), a critical challenge unaddressed by existing benchmarks that conflate HOI with static attributes, relying on global metrics incapable of simultaneously assessing dynamic interaction validity and entangled human-object pair preservation. Thus, we first introduce **HOI-Edit**, a comprehensive benchmark with three progressive cognitive levels, which features an automated metric **HOI-Eval** that first reliably evaluates instance-level interaction by letting VLM Q&A after thinking with images containing grounded Human-Object pair. Considering the task's essence of remodeling dynamic relationships, we benchmark Image-to-Video (I2V) models, finding them inherently suited for dynamic editing due to their temporal generation capabilities. Crucially, beyond superior performance, this capability provides a "replay of the failure process", offering unique diagnosability into why errors occur. We thus propose **SCPE (Self-Correcting Process Editing)**, a novel, agentic self-correcting framework that constrains the generation of I2V models through iteratively refined prompts, enabling the generated videos to more accurately present the target HOI. Extracted frames from these videos are the final editing results. On HOI-Edit, SCPE achieves performance competitive with state-of-the-art (SOTA) editing models like Nano Banana on interaction.

> 这是轻量代码发布。API key、原图、参考视频、生成视频、编辑帧、模型权重、checkpoint 和运行输出均不包含。

English version: [README.md](README.md)

## News

- **Code release**：提供 SCPE generation code 和 HOI-Eval evaluation code。
- **Lite benchmark files**：包含 HOI-Edit annotation JSON。
- **Local assets required**：图片、视频、Wan2.2 权重、GroundingDINO 权重、SAM2 checkpoint 需本地准备。

## HOI-Edit and SCPE Overview

```text
原图 + 短 HOI instruction
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

## 目录结构

```text
.
├── scpe/                      # SCPE generation pipeline
│   ├── scripts/               # ACE、Wan2.2 generation、QA2 frame selection
│   ├── data/                  # Playbook seeds 和 prompt templates
│   ├── run_minimal.sh         # SCPE 一键运行入口
│   ├── env.example            # SCPE 本地配置模板
│   └── README.md              # SCPE 详细说明
├── run_eval.sh                # HOI-Eval: QA + HOI + final scoring
├── run_qa_hoi.sh              # HOI-Eval: QA + HOI only
├── data_v7/CR/                # HOI-Edit annotation JSON
├── evaluation/                # QA、HOI、预处理、计分脚本
├── sam2/                      # SAM2 tracking code
├── third_party/GroundingDINO/ # GroundingDINO code
└── env/                       # HOI-Eval 配置模板和依赖
```

## SCPE 环境配置

SCPE 全称为 **Self-Correcting Process Editing**，代码位于 [scpe/](scpe)。它使用 Gemini 完成 ACE / Playbook 学习与 QA2，可通过 DashScope Wan2.2 I2V 或本地 Wan2.2 I2V A14B 生成视频。

```bash
cd scpe

# 创建 Python 环境并安装依赖
./setup_env.sh

# 创建本地配置。不要提交 env.local。
cp env.example env.local
```

编辑 `scpe/env.local`：

```bash
export GEMINI_API_KEY="your-gemini-api-key"

# 仅 DashScope backend 需要
export DASHSCOPE_API_KEY="your-dashscope-api-key"

# cn 或 en，控制 prompt / Playbook 模板语言
export ACE_LANG="cn"

# 数据根目录：标注、原图、epoch0 参考视频
export DATA_ROOT="/path/to/CameraReady"

# 仅本地 Wan2.2 backend 需要
export WAN22_REPO="/path/to/Wan2.2"
export WAN22_CKPT_DIR="/path/to/Wan2.2-I2V-A14B"
```

`DATA_ROOT` 应为：

```text
DATA_ROOT/
├── collected_annotations_bboxes_v7_L1L2_questions.json
├── collected_annotations_bboxes_v7_L3_questions.json
├── data_v7_L12/
├── data_v7_L3/
├── epoch_0_L1L2/
└── epoch_0_L3/
```

## SCPE 运行

最小端到端运行：

```bash
cd scpe
source env.local

# 调试：每个 split 跑 2 条
LIMIT=2 ./run_minimal.sh all
```

分阶段运行：

```bash
./run_minimal.sh learn      # 从 epoch0 参考视频学习 / 更新 Playbook
./run_minimal.sh enhance    # 由原图 + instruction 生成 enhanced prompts
./run_minimal.sh wan22      # 用 Wan2.2 生成视频
./run_minimal.sh qa2        # 从生成视频中选帧
```

使用本地 Wan2.2 A14B：

```bash
WAN22_BACKEND=local ./run_minimal.sh wan22
```

常用续跑 / 调试：

```bash
SKIP_LEARN=1 ./run_minimal.sh enhance
SKIP_QA2=1 ./run_minimal.sh all
FORCE_REGEN=1 ./run_minimal.sh wan22
ACE_LANG=en ./run_minimal.sh all
```

SCPE 输出：

```text
scpe/output/
├── playbook.json
├── playbook_normalized.json
├── enhanced_prompts.json
├── wan22_videos/             # DashScope backend
├── wan22_local_a14b/         # local Wan2.2 backend
└── qa2_frames/
```

## HOI-Eval 环境配置

创建本地配置：

```bash
cp env/local.conf.example env/local.conf
```

编辑 `env/local.conf`：

```bash
export DINO_ENV_PY="/path/to/conda/envs/cr-dino/bin/python"
export SS_ENV_PY="/path/to/conda/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="/path/to/conda/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

完整评测使用三个环境：

| 环境 | 用途 | 依赖 |
|---|---|---|
| `cr-dino` | GroundingDINO detection | `env/requirements-dino.txt` |
| `cr-sam2` | SAM2 tracking | `env/requirements-sam2.txt` |
| `cr-gemini` | Gemini QA + HOI check + scoring | `env/requirements-hoi-google.txt` |

依赖说明见 [env/requirements/README.md](env/requirements/README.md)。

## HOI-Eval 运行

把编辑帧放在：

```text
data_v7/CR/<model_name>_frames/L1L2/
data_v7/CR/<model_name>_frames/L3/
```

然后运行：

```bash
MODELS=<model_name> GPU_ID=0 bash run_eval.sh
```

如果编辑帧在仓库外：

```bash
FRAMES_DIR=/path/to/frames MODELS=<model_name> bash run_eval.sh
```

`FRAMES_DIR` 下应包含 `L1L2/` 和 `L3/`。

部分流程：

```bash
SKIP_HOI=1 MODELS=<model_name> bash run_qa_hoi.sh
SKIP_QA=1 MODELS=<model_name> GPU_ID=0 bash run_qa_hoi.sh
SCORES_ONLY=1 SCORE_MODEL=<model_name> bash run_eval.sh
```

## 论文模块与代码对应

| 论文部分 | 代码 / 文件 |
|---|---|
| SCPE / ACE prompt enhancement | `scpe/scripts/ace_i2v_official3.py`, `scpe/data/*playbook*`, `scpe/data/ace_prompts_*.json` |
| Wan2.2 I2V generation | `scpe/scripts/wan22_generate_from_enhanced_prompts.py`, `scpe/scripts/wan22_local_i2v_a14b_generate.py` |
| QA2 frame selection | `scpe/scripts/ace_v2f_qa2.py`, `scpe/data/qa2_prompts_*.json` |
| HOI-Edit annotations | `data_v7/CR/*_scoring_final.json` |
| Gemini QA evaluation | `evaluation/run_qa_gemini_question_v6.sh`, `evaluation/run_question_answering.py` |
| HOI-Eval interaction check | `evaluation/run_full_eval_v7_google.sh`, `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` |
| HOI preprocessing | `evaluation/resize_edited_images_to_original.py`, `evaluation/inference_on_multi_image_eval_optimized.py`, `sam2/run_sam2_tracking_for_eval.py` |
| Final scoring | `evaluation/compute_scoring_final_scores.py` |

## 需要本地补充的资源

| 资源 | 放置路径 |
|---|---|
| L1/L2 原图 | `data_v7/CR/data_v7_L12/` |
| L3 原图 | `data_v7/CR/data_v7_L3/` |
| HOI-Eval 评测编辑帧 | `data_v7/CR/<model>_frames/L1L2/`, `data_v7/CR/<model>_frames/L3/` |
| GroundingDINO weight | `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth` |
| SAM2 checkpoint | `sam2/checkpoints/sam2.1_hiera_large.pt` |
| SCPE 参考视频 / 原图 | 由 `scpe/env.local` 的 `DATA_ROOT` 配置 |
| 本地 Wan2.2 仓库和权重 | 由 `WAN22_REPO` 和 `WAN22_CKPT_DIR` 配置 |

## 评测输出与指标

| 输出 | 路径 |
|---|---|
| HOI-Eval QA 结果 | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| HOI-Eval HOI 结果 | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |
| 最终分数表 | `eval_runs/scoring_final_scores_4dp.json` |

`evaluation/compute_scoring_final_scores.py` 合并 HOI 和 QA 输出：

| 指标 | 来源 |
|---|---|
| `I` | HOI interaction confidence, `max_yes_confidence` |
| `S` | Subject preservation, `subject_similarity` |
| `O` | Object preservation, `object_similarity` |
| `QA` | 生成问题的 Gemini answer confidence |
| `IQA` | L1 / initial-position L2 取 `I`；其他取 `min(I, QA)` |

## 说明

- `env/local.conf` 和 `scpe/env.local` 被 git ignore，因为它们包含本地路径和 API key。
- `scpe/output/`、`scpe/logs/`、`eval_runs/` 是运行输出，默认忽略。
- release 范围和第三方代码说明见 [RELEASE_NOTES.md](RELEASE_NOTES.md) 与 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
