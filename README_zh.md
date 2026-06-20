# HOI-Edit / CR v7 代码发布

这是 HOI editing 论文代码的临时开源版本，包含两部分：

- **SCPE generation pipeline**：ACE/Playbook prompt enhancement、Wan2.2 图生视频、QA2 选帧。
- **CR v7 evaluation pipeline**：CR 标注、Gemini QA、HOI Check、最终 I / S / O / IQA 计分。

> 本仓库是轻量代码发布。API key、原图、编辑帧/视频、模型权重、checkpoint、运行输出均不包含。

English version: [README.md](README.md)

## News

- **Code release**：提供 SCPE pipeline 与 CR v7 evaluation code。
- **Lite assets**：包含标注 JSON；大规模图片、视频、权重、checkpoint 需本地自行准备。

## Overview

```text
短 HOI instruction + 原图
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

## 论文模块与代码对应

| 论文部分 | 功能 | 代码 / 文件 |
|---|---|---|
| SCPE / ACE prompt enhancement | 从失败案例学习 Playbook，并把短 HOI 指令增强为详细 I2V prompt | `scpe/scripts/ace_i2v_official3.py`, `scpe/data/*playbook*`, `scpe/data/ace_prompts_*.json` |
| Wan2.2 I2V generation | 使用 DashScope 或本地 Wan2.2 A14B 生成视频 | `scpe/scripts/wan22_generate_from_enhanced_prompts.py`, `scpe/scripts/wan22_local_i2v_a14b_generate.py` |
| QA2 frame selection | 从生成视频中选择最适合下游图像评测的帧 | `scpe/scripts/ace_v2f_qa2.py`, `scpe/data/qa2_prompts_*.json` |
| CR benchmark annotations | 保存 instruction、HOI tag、生成 QA 问题和计分字段 | `data_v7/CR/*_scoring_final.json` |
| Edited image inputs | 待评测模型输出或 QA2 选出的帧 | `data_v7/CR/<model>_frames/{L1L2,L3}/` 或 `FRAMES_DIR=/path/to/frames` |
| Phase 1: QA evaluation | 对 `question_v6` 做 Gemini VQA | `evaluation/run_qa_gemini_question_v6.sh`, `evaluation/run_question_answering.py` |
| Phase 2: HOI check | 评估交互是否完成，以及主体/客体是否保持 | `evaluation/run_full_eval_v7_google.sh`, `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` |
| HOI preprocessing | resize、person/object detection、SAM2 tracking | `evaluation/resize_edited_images_to_original.py`, `evaluation/inference_on_multi_image_eval_optimized.py`, `sam2/run_sam2_tracking_for_eval.py` |
| Final metric table | 合并 QA + HOI，计算 I / S / O / IQA | `evaluation/compute_scoring_final_scores.py` |
| 一键评测入口 | 串联 QA、HOI、计分 | `run_eval.sh` |

## 目录结构

```text
.
├── scpe/                      # SCPE generation pipeline
├── run_eval.sh                # CR evaluation: QA + HOI + final scoring
├── run_qa_hoi.sh              # CR evaluation: QA + HOI only
├── data_v7/CR/                # CR 标注 JSON
├── evaluation/                # QA、HOI、预处理、计分脚本
├── sam2/                      # SAM2 tracking code
├── third_party/GroundingDINO/ # GroundingDINO code
├── env/                       # evaluation 配置模板和依赖
└── eval_runs/                 # 评测输出
```

## SCPE 快速开始

完整说明见 [scpe/README.md](scpe/README.md)。最小用法：

```bash
cd scpe
cp env.example env.local
# 编辑 env.local: GEMINI_API_KEY，可选 DASHSCOPE_API_KEY、DATA_ROOT、WAN22 路径

LIMIT=2 ./run_minimal.sh all
```

常用 SCPE 命令：

```bash
./run_minimal.sh learn      # 从 epoch0 参考视频学习 Playbook
./run_minimal.sh enhance    # 生成 enhanced prompts
./run_minimal.sh wan22      # 用 Wan2.2 生成视频
./run_minimal.sh qa2        # 从生成视频选帧/评估
WAN22_BACKEND=local ./run_minimal.sh wan22
```

## CR Evaluation 快速开始

创建本地评测配置：

```bash
cp env/local.conf.example env/local.conf
```

填写：

```bash
export DINO_ENV_PY="/path/to/conda/envs/cr-dino/bin/python"
export SS_ENV_PY="/path/to/conda/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="/path/to/conda/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

完整 CR 评测：

```bash
MODELS=<model_name> GPU_ID=0 bash run_eval.sh
```

如果编辑帧在仓库外：

```bash
FRAMES_DIR=/path/to/frames MODELS=<model_name> bash run_eval.sh
```

`FRAMES_DIR` 下应包含 `L1L2/` 和 `L3/`。

## 需要本地补充的资源

| 资源 | 放置路径 |
|---|---|
| L1/L2 原图 | `data_v7/CR/data_v7_L12/` |
| L3 原图 | `data_v7/CR/data_v7_L3/` |
| CR 评测编辑帧 | `data_v7/CR/<model>_frames/L1L2/`, `data_v7/CR/<model>_frames/L3/` |
| GroundingDINO weight | `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth` |
| SAM2 checkpoint | `sam2/checkpoints/sam2.1_hiera_large.pt` |
| SCPE 参考视频 / 原图 | 由 `scpe/env.local` 中 `DATA_ROOT` 配置 |
| 本地 Wan2.2 仓库和权重 | 由 `WAN22_REPO` 和 `WAN22_CKPT_DIR` 配置 |

## 输出

| 输出 | 路径 |
|---|---|
| SCPE enhanced prompts | `scpe/output/enhanced_prompts.json` |
| SCPE 生成视频 | `scpe/output/wan22_videos/` 或 `scpe/output/wan22_local_a14b/` |
| QA2 选帧 | `scpe/output/qa2_frames/` |
| CR QA 结果 | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| CR HOI 结果 | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |
| 最终分数表 | `eval_runs/scoring_final_scores_4dp.json` |

## 指标

`evaluation/compute_scoring_final_scores.py` 合并 HOI 和 QA 输出：

| 指标 | 来源 |
|---|---|
| `I` | HOI interaction confidence, `max_yes_confidence` |
| `S` | Subject preservation, `subject_similarity` |
| `O` | Object preservation, `object_similarity` |
| `QA` | 生成问题的 Gemini answer confidence |
| `IQA` | L1 / initial-position L2 取 `I`；其他取 `min(I, QA)` |

## 说明

- `env/workspace.conf` 会按仓库根目录自动解析评测路径。
- `env/local.conf` 和 `scpe/env.local` 被 git ignore，因为它们包含本机路径和 API key。
- release 范围和第三方代码说明见 [RELEASE_NOTES.md](RELEASE_NOTES.md) 与 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
