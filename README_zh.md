# CR v7 评测代码

这是论文中 CR v7 evaluation 的 lite 开源版本。仓库按论文评测流程组织：**CR 数据**、**QA 评测**、**HOI Check**、**最终计分**。

仓库不包含 API key。原图、编辑结果、模型权重、checkpoint、运行输出也不包含在 lite release 中。

English version: [README.md](README.md)

## 论文模块与代码对应

| 论文部分 | 功能 | 代码 / 文件 |
|---|---|---|
| CR benchmark annotations | 保存 instruction、HOI tag、生成的 QA 问题和计分字段 | `data_v7/CR/*_scoring_final.json` |
| Edited image inputs | 待评测模型的编辑结果 | `data_v7/CR/<model>_frames/{L1L2,L3}/` 或 `FRAMES_DIR=/path/to/frames` |
| Phase 1: QA evaluation | 对 `question_v6` 做 Gemini VQA | `evaluation/run_qa_gemini_question_v6.sh`, `evaluation/run_question_answering.py` |
| Phase 2: HOI check | 评估交互是否完成，以及主体/客体是否保持 | `evaluation/run_full_eval_v7_google.sh`, `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` |
| HOI preprocessing | resize、person/object detection、SAM2 tracking | `evaluation/resize_edited_images_to_original.py`, `evaluation/inference_on_multi_image_eval_optimized.py`, `sam2/run_sam2_tracking_for_eval.py` |
| Final metric table | 合并 QA + HOI，计算 I / S / O / IQA | `evaluation/compute_scoring_final_scores.py` |
| 一键复现入口 | 串联 QA、HOI、计分 | `run_eval.sh` |

## 目录结构

```text
cr_eval_release/
├── run_eval.sh                # QA + HOI + final scoring
├── run_qa_hoi.sh              # QA + HOI only
├── data_v7/CR/                # CR 标注 JSON
├── evaluation/                # QA、HOI、预处理、计分脚本
├── sam2/                      # SAM2 tracking code
├── third_party/GroundingDINO/ # GroundingDINO code
├── env/                       # 本地配置模板和依赖
└── eval_runs/                 # 运行输出
```

## 配置

创建本地配置：

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

完整流程使用三个 Python 环境：

| 环境 | 用途 | 依赖 |
|---|---|---|
| `cr-dino` | GroundingDINO 检测 | `env/requirements-dino.txt` |
| `cr-sam2` | SAM2 tracking | `env/requirements-sam2.txt` |
| `cr-gemini` | Gemini QA + HOI check + scoring | `env/requirements-hoi-google.txt` |

更详细依赖见 [env/requirements/README.md](env/requirements/README.md)。

## 需要本地补充的资源

这些大文件或私有资源没有提交到仓库：

| 资源 | 放置路径 |
|---|---|
| L1/L2 原图 | `data_v7/CR/data_v7_L12/` |
| L3 原图 | `data_v7/CR/data_v7_L3/` |
| 编辑结果 | `data_v7/CR/<model>_frames/L1L2/`, `data_v7/CR/<model>_frames/L3/` |
| GroundingDINO weight | `third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth` |
| SAM2 checkpoint | `sam2/checkpoints/sam2.1_hiera_large.pt` |

## 运行

完整流程：

```bash
MODELS=<model_name> GPU_ID=0 bash run_eval.sh
```

只跑 QA + HOI，不计分：

```bash
MODELS=<model_name> GPU_ID=0 bash run_qa_hoi.sh
```

常用部分流程：

```bash
SKIP_HOI=1 MODELS=<model_name> bash run_qa_hoi.sh
SKIP_QA=1 MODELS=<model_name> GPU_ID=0 bash run_qa_hoi.sh
SCORES_ONLY=1 SCORE_MODEL=<model_name> bash run_eval.sh
```

如果编辑结果在仓库外：

```bash
FRAMES_DIR=/path/to/frames MODELS=<model_name> bash run_eval.sh
```

`FRAMES_DIR` 下应包含 `L1L2/` 和 `L3/`。

## 输出

| 输出 | 路径 |
|---|---|
| QA 结果 | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| HOI 结果 | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |
| 最终分数表 | `eval_runs/scoring_final_scores_4dp.json` |

## 指标

`compute_scoring_final_scores.py` 合并 HOI 和 QA 输出：

| 指标 | 来源 |
|---|---|
| `I` | HOI interaction confidence, `max_yes_confidence` |
| `S` | Subject preservation, `subject_similarity` |
| `O` | Object preservation, `object_similarity` |
| `QA` | 生成问题的 Gemini answer confidence |
| `IQA` | L1 / initial-position L2 取 `I`；其他取 `min(I, QA)` |

## 说明

- `env/workspace.conf` 会按仓库根目录自动解析路径，移动代码后一般不用改脚本。
- `env/local.conf` 被 git ignore，因为它包含本机路径和 API key。
- release 范围和第三方代码说明见 [RELEASE_NOTES.md](RELEASE_NOTES.md) 与 [THIRD_PARTY_NOTICES.md](THIRD_PARTY_NOTICES.md)。
