# Taming I2V models for Image HOI Editing: A Cognitive Benchmark and Agentic Self-Correcting Framework

这是 **HOI-Edit**、**HOI-Eval** 和 **SCPE** 的官方代码发布。

Current image editing methods excels at static attributes but fails at complex Human-Object Interactions (HOI), a critical challenge unaddressed by existing benchmarks that conflate HOI with static attributes, relying on global metrics incapable of simultaneously assessing dynamic interaction validity and entangled human-object pair preservation. Thus, we first introduce **HOI-Edit**, a comprehensive benchmark with three progressive cognitive levels, which features an automated metric **HOI-Eval** that first reliably evaluates instance-level interaction by letting VLM Q&A after thinking with images containing grounded Human-Object pair. Considering the task's essence of remodeling dynamic relationships, we benchmark Image-to-Video (I2V) models, finding them inherently suited for dynamic editing due to their temporal generation capabilities. Crucially, beyond superior performance, this capability provides a "replay of the failure process", offering unique diagnosability into why errors occur. We thus propose **SCPE (Self-Correcting Process Editing)**, a novel, agentic self-correcting framework that constrains the generation of I2V models through iteratively refined prompts, enabling the generated videos to more accurately present the target HOI. Extracted frames from these videos are the final editing results. On HOI-Edit, SCPE achieves performance competitive with state-of-the-art (SOTA) editing models like Nano Banana on interaction.

> GitHub 仓库只放代码和标注 JSON。大文件资源单独下载，见 [ASSETS.md](ASSETS.md)。

English version: [README.md](README.md)

## News

- **Code release**：提供 SCPE generation code 和 HOI-Eval evaluation code。
- **Benchmark assets**：原图和模型 checkpoint 将通过百度云资源包发布。
- **Lite GitHub repo**：API key、原图、参考视频、生成视频、编辑帧、模型权重、checkpoint、运行输出均不提交。

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

## 下载资源

完整资源包说明见 [ASSETS.md](ASSETS.md)。

下载并解压资源后，应放置为：

```text
data_v7/CR/data_v7_L12/
data_v7/CR/data_v7_L3/
third_party/GroundingDINO/weights/groundingdino_swint_ogc.pth
sam2/checkpoints/sam2.1_hiera_large.pt
```

## SCPE 环境配置与运行

```bash
cd scpe
./setup_env.sh
cp env.example env.local
```

编辑 `scpe/env.local`：

```bash
export GEMINI_API_KEY="your-gemini-api-key"
export DASHSCOPE_API_KEY="your-dashscope-api-key"  # 仅 DashScope backend 需要
export ACE_LANG="cn"
export DATA_ROOT="/path/to/CameraReady"
export WAN22_REPO="/path/to/Wan2.2"
export WAN22_CKPT_DIR="/path/to/Wan2.2-I2V-A14B"
```

运行：

```bash
source env.local
LIMIT=2 ./run_minimal.sh all
./run_minimal.sh learn
./run_minimal.sh enhance
./run_minimal.sh wan22
./run_minimal.sh qa2
WAN22_BACKEND=local ./run_minimal.sh wan22
```

## HOI-Eval 环境配置与运行

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

把编辑帧放在：

```text
data_v7/CR/<model_name>_frames/L1L2/
data_v7/CR/<model_name>_frames/L3/
```

然后运行：

```bash
MODELS=<model_name> GPU_ID=0 bash run_eval.sh
```

## 论文模块与代码对应

| 论文部分 | 代码 / 文件 |
|---|---|
| SCPE / ACE prompt enhancement | `scpe/scripts/ace_i2v_official3.py`, `scpe/data/*playbook*`, `scpe/data/ace_prompts_*.json` |
| Wan2.2 I2V generation | `scpe/scripts/wan22_generate_from_enhanced_prompts.py`, `scpe/scripts/wan22_local_i2v_a14b_generate.py` |
| QA2 frame selection | `scpe/scripts/ace_v2f_qa2.py`, `scpe/data/qa2_prompts_*.json` |
| HOI-Edit annotations | `data_v7/CR/*_scoring_final.json` |
| HOI-Eval interaction check | `evaluation/run_full_eval_v7_google.sh`, `evaluation/gemini3_final_hoicheck_new_noquestion_track_google_newsim.py` |
| Final scoring | `evaluation/compute_scoring_final_scores.py` |
