# CR v7 评测发布包

可独立迁移的 CR v7 评测包。QA 与 HOI Check **均使用 Google 官方 Gemini 2.5 Pro API**（`google-genai` SDK）。包内不含 API Key，运行前需自行配置。

这是一个 **lite 代码发布包**：包含标注 JSON 与评测代码；原图、编辑帧、模型权重、checkpoint、运行输出与本地密钥配置均不随仓库提交。详见 [RELEASE_NOTES.md](RELEASE_NOTES.md)。

**路径示例：** `/path/to/cr_eval_release`

English version: [README.md](README.md)

---

## 目录结构

```
cr_eval_release/
├── run_qa_hoi.sh              # ★ 主入口：QA + HOI 两阶段
├── run_eval.sh                # 完整流程：QA + HOI + 计分
├── README.md                  # 英文说明
├── README_zh.md               # 中文说明（本文件）
│
├── env/
│   ├── workspace.conf         # 工作区路径（自动检测，一般不需改）
│   ├── local.conf.example     # 本机配置模板 → 复制为 local.conf
│   ├── setup_check.sh         # 快捷检查
│   ├── verify_scripts.sh      # 完整检查：文件、权重、语法、密钥扫描
│   ├── requirements-dino.txt
│   ├── requirements-sam2.txt
│   ├── requirements-hoi-google.txt
│   └── requirements/README.md # 三环境安装说明
│
├── data_v7/CR/                # ★ 标注 JSON；原图需本地自行放入
│   ├── collected_annotations_bboxes_v7_L1L2_questions_scoring_final.json  (499)
│   ├── collected_annotations_bboxes_v7_L3_questions_scoring_final.json    (136)
│   ├── data_v7_L12/           # 需自行提供：L1L2 原图 (499)
│   ├── data_v7_L3/            # 需自行提供：L3 原图 (136 keys, 143 files)
│   └── <model>_frames/        # ⚠ 需自备：各模型编辑图
│       ├── L1L2/
│       └── L3/
│
├── evaluation/                # 评测脚本
├── sam2/                      # SAM2 tracking 代码；checkpoint 需本地加入
├── third_party/GroundingDINO/ # Object detection 代码；weights 需本地加入
└── eval_runs/                 # 输出目录（QA / HOI / 得分）
```

---

## 一、快速开始（5 步）

### 1. 复制发布包

```bash
cp -r cr_eval_release /your/target/path
cd /your/target/path/cr_eval_release
```

### 2. 配置本机环境

```bash
cp env/local.conf.example env/local.conf
# 编辑 local.conf，填写三个 Python 路径和 GEMINI_API_KEY
```

`local.conf` 最少需要：

```bash
export DINO_ENV_PY="/path/to/conda/envs/cr-dino/bin/python"
export SS_ENV_PY="/path/to/conda/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="/path/to/conda/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

### 3. 安装 Conda 环境（完整流程需要三个）

**是的，完整 QA + HOI 评测需要配置三个独立的 Conda 环境。** 流水线在不同阶段会调用不同的 Python 解释器，路径写在 `env/local.conf` 里：

| 环境 | `local.conf` 变量 | 用于哪一步 | 依赖文件 |
|------|-------------------|-----------|----------|
| `cr-dino` | `DINO_ENV_PY` | HOI Step 1 — GroundingDINO 检测 | `env/requirements-dino.txt` + 编译 `third_party/GroundingDINO` |
| `cr-sam2` | `SS_ENV_PY` | HOI Step 2 — SAM2 跟踪 | `env/requirements-sam2.txt` |
| `cr-gemini` | `GOOGLE_ENV_PY` | Phase 1 QA + HOI Step 3 Gemini API | `env/requirements-hoi-google.txt` |

为什么要三个环境而不是一个？PyTorch/CUDA 版本、GroundingDINO 的 CUDA 编译扩展、SAM2、`google-genai` 的依赖互相冲突，拆成三个 env 最稳，也和 `run_qa_hoi.sh` 的实际调用方式一致。

**只跑部分阶段时，可以少配：**

| 运行方式 | 需要的环境 |
|----------|-----------|
| 完整 `run_qa_hoi.sh`（QA + HOI） | 三个都要 |
| 只跑 QA（`SKIP_HOI=1`） | 只需 `cr-gemini` |
| 只补 HOI Check，DINO/SAM2 已有（`HOI_CHECK_ONLY=1`） | 只需 `cr-gemini` |
| 只跑 HOI 全流程（`SKIP_QA=1`） | 三个都要 |

#### 3a. `cr-dino`（GroundingDINO 检测）

代码：`third_party/GroundingDINO/`  
权重（lite release 不包含）：`third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth`  
调用脚本：`evaluation/inference_on_multi_image_eval_optimized.py`

```bash
conda create -n cr-dino python=3.10 -y
conda activate cr-dino
pip install -r env/requirements-dino.txt
pip install -r third_party/GroundingDINO/requirements.txt
cd third_party/GroundingDINO && pip install -e .
python -c "import torch; print('cuda:', torch.cuda.is_available())"
```

#### 3b. `cr-sam2`（SAM2 跟踪）

代码：`sam2/`  
权重（lite release 不包含）：`sam2/checkpoints/sam2.1_hiera_large.pt`  
调用脚本：`sam2/run_sam2_tracking_for_eval.py`

```bash
conda create -n cr-sam2 python=3.10 -y
conda activate cr-sam2
pip install -r env/requirements-sam2.txt
# 若权重缺失：bash sam2/checkpoints/download_ckpts.sh
```

#### 3c. `cr-gemini`（Gemini QA + HOI Check）

调用脚本：`evaluation/run_question_answering.py`、`evaluation/gemini3_*_google_newsim.py`

```bash
conda create -n cr-gemini python=3.11 -y
conda activate cr-gemini
pip install -r env/requirements-hoi-google.txt
python -c "from google import genai; print('google-genai OK')"
```

#### 3d. 在 `local.conf` 里绑定三个解释器

三个环境装好后，把路径写进 `local.conf`（改成你机器上的 conda 路径）：

```bash
export DINO_ENV_PY="$HOME/miniconda3/envs/cr-dino/bin/python"
export SS_ENV_PY="$HOME/miniconda3/envs/cr-sam2/bin/python"
export GOOGLE_ENV_PY="$HOME/miniconda3/envs/cr-gemini/bin/python"
export GEMINI_API_KEY="your-gemini-api-key"
export GPU_ID="0"
```

更完整的说明见 [env/requirements/README.md](env/requirements/README.md)

### 4. 放入模型编辑图

自定一个 **名称**（用你实验里已有的 checkpoint /  run 名即可），编辑图默认放在：

```
data_v7/CR/<名称>_frames/L1L2/
data_v7/CR/<名称>_frames/L3/
```

运行：`MODELS=<名称> bash run_qa_hoi.sh`

若目录不在默认位置，设置 `FRAMES_DIR=/你的/frames根目录`（其下需有 `L1L2/`、`L3/`）。详见 **「需配置的路径」**。

### 5. 检查并运行

```bash
bash env/setup_check.sh          # 检查文件、权重、语法
DRY_RUN=1 MODELS=<名称> bash run_qa_hoi.sh     # 预览命令
MODELS=<名称> GPU_ID=0 bash run_qa_hoi.sh      # 正式运行
```

---

## 需配置的路径

**同一字符串**贯穿输入与输出（下文用 `<名称>` 占位，请换成你自己的名字）：

| 配置项 | 位置 |
|--------|------|
| 编辑图（默认） | `data_v7/CR/<名称>_frames/L1L2/` 与 `.../L3/` |
| 编辑图（覆盖） | `FRAMES_DIR=/你的/frames根目录`（其下含 `L1L2/`、`L3/`） |
| 运行 QA + HOI | `MODELS=<名称>`（多个用逗号分隔） |
| QA 输出 | `eval_runs/qa_results_v6_{L1L2,L3}_<名称>.json` |
| HOI 输出 | `eval_runs/<名称>_{L1L2,L3}_full/.../results_<名称>_*_google_full.json` |
| 计分 | 与上面相同： `--model <名称>` 或 `SCORE_MODEL=<名称>` |

```bash
MODELS=<名称> GPU_ID=0 bash run_qa_hoi.sh
```

**目录布局不同？**

- 软链接：`ln -s /你的/frames data_v7/CR/<名称>_frames`
- 或：`FRAMES_DIR=/你的/frames`
- 或手动指定 HOI 路径：

```bash
export EVAL_V7_IMAGE_DIR=/你的/frames/L3
export EVAL_V7_JSON=data_v7/CR/collected_annotations_bboxes_v7_L3_questions_scoring_final.json
export EVAL_V7_ORIG_DIR=data_v7/CR/data_v7_L3
bash evaluation/run_full_eval_v7_google.sh --model <名称> --datasets V7 --gpu-id 0
```

单独跑 QA 时可传：`--model-edited-dir <名称>=/你的路径`

---

## 二、主入口 `run_qa_hoi.sh`

按顺序执行 QA 与 HOI 两阶段：

| 阶段 | 脚本 | 模型/API | 输出 |
|------|------|----------|------|
| Phase 1 QA | `run_qa_gemini_question_v6.sh` → `run_question_answering.py` | Gemini 2.5 Pro | `eval_runs/qa_results_v6_{L1L2,L3}_<model>.json` |
| Phase 2 HOI | `run_full_eval_v7_google.sh` → resize/DINO/SAM2/Gemini | Gemini 2.5 Pro | `eval_runs/<model>_{L1L2,L3}_full/.../results_*_google_full.json` |

### 基本用法

```bash
MODELS=<名称> GPU_ID=0 bash run_qa_hoi.sh
SPLITS=L3 MODELS=<名称> bash run_qa_hoi.sh
SKIP_HOI=1 MODELS=<名称> bash run_qa_hoi.sh
SKIP_QA=1 MODELS=<名称> GPU_ID=0 bash run_qa_hoi.sh
SKIP_QA=1 HOI_CHECK_ONLY=1 MODELS=<名称> GPU_ID=0 bash run_qa_hoi.sh
DRY_RUN=1 MODELS=<名称> bash run_qa_hoi.sh
```

### 环境变量参考

| 变量 | 默认 | 说明 |
|------|------|------|
| `GEMINI_API_KEY` | （必填） | Google Gemini API Key，QA 与 HOI 共用 |
| `MODELS` | （必填） | 逗号分隔的名称，与输出路径中的名字一致（见 **需配置的路径**） |
| `FRAMES_DIR` | 空 | 可选：编辑图根目录（含 `L1L2/`、`L3/`）；默认 `data_v7/CR/<名称>_frames` |
| `SPLITS` | `L1L2,L3` | 数据集 split |
| `GPU_ID` | `0` | CUDA 设备 |
| `SKIP_QA` | `0` | `1` 跳过 QA |
| `SKIP_HOI` | `0` | `1` 跳过 HOI |
| `HOI_CHECK_ONLY` | `0` | `1` 跳过 resize/DINO/SAM2，直接 HOI |
| `SKIP_CONVERT` | `1` | `0` 开启编辑图格式规范化 |
| `SKIP_RESIZE` | `0` | `1` 跳过 resize |
| `SKIP_DINO` | `0` | `1` 跳过 GroundingDINO |
| `SKIP_SAM_TRACK` | `0` | `1` 跳过 SAM2 跟踪 |
| `MAX_PARALLEL_QA` | `2` | QA 并行模型数 |
| `CR_DEFAULT_PROXY_URL` | 空 | 可选 HTTP 代理（访问 Google API） |

---

## 三、完整流程 `run_eval.sh`

在 `run_qa_hoi.sh` 基础上增加 **Phase 3 计分**：

```bash
bash run_eval.sh                                    # QA + HOI + 计分
SCORES_ONLY=1 bash run_eval.sh                      # 只计分
SKIP_SCORES=1 bash run_eval.sh                      # 不计分
LEGACY_SCORES_ROOT=/path/to/old bash run_eval.sh    # 合并历史 HOI/QA 结果
MODELS=<名称> bash run_eval.sh                      # QA+HOI 完成后计分
SCORE_MODEL=<名称> SCORES_ONLY=1 bash run_eval.sh   # 只计分
```

输出：`eval_runs/scoring_final_scores_4dp.json`

---

## 三（续）、最终得分怎么算

由 `evaluation/compute_scoring_final_scores.py` 实现：读取 **HOI 结果 + QA 结果**，按 5models 表的方式汇总。

### 怎么跑

```bash
# QA + HOI 跑完后（或通过 run_eval.sh）：
python evaluation/compute_scoring_final_scores.py \
  --workspace . \
  --model <名称> \
  --decimals 4

# 只计分，并从其他目录合并历史 HOI/QA：
LEGACY_SCORES_ROOT=/path/to/gjy SCORES_ONLY=1 bash run_eval.sh
```

`--model` 必须与跑 QA/HOI 时 `MODELS=` 使用的 **同一字符串**。逗号分隔可一次计多个。

### 输入文件

将 `<名称>` 换成你的名字：

| 来源 | 路径（在 `eval_runs/` 下） |
|------|---------------------------|
| HOI（L1L2） | `<名称>_L1L2_full/.../results_<名称>_L1L2_google_full.json` |
| HOI（L3） | `<名称>_L3_full/.../results_<名称>_L3_google_full.json` |
| QA（L1L2） | `qa_results_v6_L1L2_<名称>.json` |
| QA（L3） | `qa_results_v6_L3_<名称>.json` |
| 标注 | `data_v7/CR/*_scoring_final.json` |

HOI 报错样本（`has_error`、`processing_error` 等）不参与该样本计分。

### 单样本指标（HOI + QA 合并）

| 指标 | 来源 | 含义 |
|------|------|------|
| **I** | HOI `max_yes_confidence` | 交互/编辑质量（Gemini HOI Check） |
| **S** | HOI `subject_similarity` | 主体保持 |
| **O** | HOI `object_similarity` | 客体保持 |
| **QA** | Phase 1 VQA | 答 Yes → 取 `confidence`；答 No → `0`（按 `question_v6` 与 `tags_v6` 对齐） |
| **IQA** | 派生 | 见下 |

**IQA 规则：**

- **L1** 类 tag，或 **`L2-...-pos_initial`**：`IQA = I`
- **其余 L2 / L3** tag：`IQA = min(I, QA)`（I 和 QA 都需有效）

### 汇总成表（最终各行）

标注分两个池（与 5models 表一致）：

- **L1/L2 块** → 只用 `L1L2_scoring_final.json`（各 tag 行 + `L1_average`、`L2_average`）
- **L3 块** → 只用 `L3_scoring_final.json`（各 tag 行 + `L3_average`）

每个 tag 行的 **I / S / O / IQA** = 该 tag 在 `tags_v6` 中出现过的所有样本上取平均。  
`L*_average` 行则对该难度下全部样本（按 sample–tag 对）再平均 IQA。

终端与 JSON 中示例：

```
L1-interaction-relation_occur          n=...  I=...  IQA=...
L2-location_understanding-specific_pos_end  ...
L1_average                             ...
L2_average                             ...
L3-non_rigid_change                    ...
L3_average                             ...
```

---

## 四、Phase 2 HOI 流水线

`run_full_eval_v7_google.sh` 对每个 `(model, split)` 执行：

```
Step 0   convert_images_for_eval.py        [可选，默认跳过]
Step 0b  resize_edited_images_to_original.py
Step 1   inference_on_multi_image_eval_optimized.py  (GroundingDINO)
Step 2   sam2/run_sam2_tracking_for_eval.py
Step 3   gemini3_*_google_newsim.py        (HOI Check: I, S, O)
```

单独调用 HOI shell（高级用法）：

```bash
export EVAL_V7_SPLIT_TAG=L3
export EVAL_V7_IMAGE_DIR=data_v7/CR/<名称>_frames/L3
export EVAL_V7_JSON=data_v7/CR/collected_annotations_bboxes_v7_L3_questions_scoring_final.json
export EVAL_V7_ORIG_DIR=data_v7/CR/data_v7_L3

bash evaluation/run_full_eval_v7_google.sh \
  --model <名称> \
  --datasets V7 \
  --gpu-id 0 \
  --output-dir eval_runs
```

---

## 五、Phase 1 QA

读取 scoring_final JSON 中的 `question_v6` 字段，对每张**编辑图**做 Yes/No 视觉问答。

输出：`eval_runs/qa_results_v6_L1L2_<model>.json`

---

## 六、环境配置

### `env/workspace.conf`

自动设置 `EVAL_WORKSPACE`、`DATA_V7_CR`、`EVAL_DIR`、`GROUNDING_DINO_ROOT`、`SAM2_ROOT`、`EVAL_RUNS_DIR`。

### `env/local.conf`（本机必配）

从 `local.conf.example` 复制，填写 Python 路径、`GEMINI_API_KEY`、可选 `GPU_ID` 和 `CR_DEFAULT_PROXY_URL`。

**请勿将 `local.conf` 提交到 Git。**

### 代理

访问 Google API 需要代理时：

```bash
export CR_DEFAULT_PROXY_URL="http://127.0.0.1:7890"
```

通过 `evaluation/cr_proxy_defaults.sh` 注入。DINO 推理时会临时禁用代理并使用 HF 镜像下载 tokenizer。

### 三个 Conda 环境

完整安装见 **「一、快速开始 → 3. 安装 Conda 环境」** 及 [env/requirements/README.md](env/requirements/README.md)。跑 QA+HOI 需三个环境；只跑 QA 或只补 HOI Check 时只需 Gemini 环境。

---

## 七、输出文件

```
eval_runs/
├── qa_results_v6_L1L2_<model>.json
├── qa_results_v6_L3_<model>.json
├── <model>_L1L2_full/.../results_*_google_full.json
├── <model>_L3_full/.../results_*_google_full.json
└── scoring_final_scores_4dp.json
```

日志：`evaluation/logs/run_qa_hoi_YYYYMMDD_HHMMSS/`

---

## 八、检查脚本

```bash
bash env/verify_scripts.sh
```

检查：入口脚本、数据 JSON、原图、模型权重、Python 语法、脚本中是否含硬编码 API Key。

---

## 九、脚本清单

| 脚本 | 职责 |
|------|------|
| `run_qa_hoi.sh` | **主入口**：串联 QA + HOI |
| `run_eval.sh` | QA + HOI + 计分 |
| `run_qa_gemini_question_v6.sh` | QA 批处理编排 |
| `run_question_answering.py` | Gemini QA 实现 |
| `run_full_eval_v7_google.sh` | HOI 流水线编排 |
| `gemini3_*_google_newsim.py` | Gemini HOI Check |
| `resize_edited_images_to_original.py` | 编辑图对齐原图尺寸 |
| `convert_images_for_eval.py` | 编辑图命名/格式规范化 |
| `inference_on_multi_image_eval_optimized.py` | GroundingDINO 批量检测 |
| `gdino_transformers_compat.py` | transformers 5.x 兼容补丁 |
| `compute_scoring_final_scores.py` | 5models 表计分 |
| `sam2/run_sam2_tracking_for_eval.py` | SAM2 跟踪 |
