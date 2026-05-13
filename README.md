# Qwen2.5-7B 酒店管理 LoRA 微调项目（LLaMA-Factory）

本项目基于 **Qwen2.5-7B-Instruct**，使用 **LLaMA-Factory** 进行 **LoRA SFT 微调**，目标是让模型在“住宿 / 餐饮 / 酒店服务管理”相关对话场景下具备更好的回答风格与业务表达能力。

仓库包含：

- 微调语料（来自魔搭下载）：[industry_instruction_semantic_cluster_dedup_住宿_餐饮_酒店_valid_val.jsonl](file:///e:/乱七八糟/骆炜权_Qwem2.5-7b_酒店管理/industry_instruction_semantic_cluster_dedup_住宿_餐饮_酒店_valid_val.jsonl)
- 训练流程 Notebook：[Qwen2_5_7B_Instruct_酒店管理 (2).ipynb](file:///e:/乱七八糟/骆炜权_Qwem2.5-7b_酒店管理/Qwen2_5_7B_Instruct_酒店管理%20(2).ipynb)
- LoRA 训练产物目录：[my_hotel_model_lora/](file:///e:/乱七八糟/骆炜权_Qwem2.5-7b_酒店管理/my_hotel_model_lora)

注意：本仓库不包含 Qwen2.5-7B-Instruct 的完整基座权重（体积过大），Notebook 中使用 ModelScope 下载模型。

## 目录结构

```text
.
├─ README.md
├─ Qwen2_5_7B_Instruct_酒店管理 (2).ipynb
├─ industry_instruction_semantic_cluster_dedup_住宿_餐饮_酒店_valid_val.jsonl
└─ my_hotel_model_lora/
   ├─ adapter_config.json
   ├─ adapter_model.safetensors
   ├─ chat_template.jinja
   ├─ tokenizer.json
   ├─ tokenizer_config.json
   ├─ trainer_state.json
   ├─ training_args.bin
   ├─ training_loss.png
   └─ checkpoint-96/...
```

## 数据集说明

语料文件为 JSONL（每行一个 JSON 对象），关键字段包含：

- `conversations`: 对话数组，`from` 取值通常为 `human` / `gpt`，`value` 为文本内容
- `lang`: 语言标注（示例中包含 `zh` / `en`）
- 其他字段如 `deita_score`、`rw_score`、`id`、`length` 等为元信息

示例（节选）：

```json
{
  "lang": "zh",
  "conversations": [
    {"from": "human", "value": "在你的摄影作品中，你如何通过照片展示住宿场所的环保与可持续性实践？"},
    {"from": "gpt", "value": "在我的摄影作品中，我通过照片展示住宿场所的环保与可持续性实践的几种方式是：..."}
  ]
}
```

## 环境准备（参考 Notebook）

Notebook 中使用的依赖安装示例：

```bash
pip install -U bitsandbytes transformers peft accelerate datasets trl
```

克隆 LLaMA-Factory：

```bash
git clone --depth 1 https://github.com/hiyouga/LLaMA-Factory.git
```

## 在 LLaMA-Factory 中准备数据（参考 Notebook）

Notebook 的做法是把本仓库的 JSONL 转换为 LLaMA-Factory 常用的 Alpaca 格式 JSON，并注册到 `dataset_info.json`：

- 输入：`industry_instruction_semantic_cluster_dedup_住宿_餐饮_酒店_valid_val.jsonl`
- 输出：`LLaMA-Factory/data/hotel_data.json`
- 注册：`LLaMA-Factory/data/dataset_info.json` 增加条目
  - `hotel_zh_data`: `{ "file_name": "hotel_data.json" }`

## LoRA SFT 训练命令（参考 Notebook）

训练命令（Notebook 原样）：

```bash
llamafactory-cli train \
  --stage sft \
  --do_train True \
  --model_name_or_path ./Qwen2.5-7B-Instruct \
  --finetuning_type lora \
  --template qwen \
  --dataset_dir ./LLaMA-Factory/data \
  --dataset hotel_zh_data \
  --output_dir ./LLaMA-Factory/saves/qwen2.5_7b/lora/hotel_sft \
  --overwrite_cache True \
  --overwrite_output_dir True \
  --cutoff_len 1024 \
  --preprocessing_num_workers 1 \
  --per_device_train_batch_size 1 \
  --gradient_accumulation_steps 16 \
  --lr_scheduler_type cosine \
  --logging_steps 10 \
  --warmup_steps 20 \
  --save_steps 100 \
  --learning_rate 5e-5 \
  --num_train_epochs 3.0 \
  --max_samples 1000 \
  --quantization_bit 4 \
  --lora_target all \
  --fp16 True \
  --plot_loss True
```

## 推理 / WebChat（参考 Notebook）

WebChat 示例（Notebook 原样）：

```bash
llamafactory-cli webchat \
  --model_name_or_path ./Qwen2.5-7B-Instruct \
  --adapter_name_or_path ./LLaMA-Factory/saves/qwen2.5_7b/lora/hotel_sft \
  --template qwen \
  --finetuning_type lora \
  --quantization_bit 4 \
  --system "你是一位拥有 10 年经验的五星级酒店大堂经理，回答需体现极高的服务意识，语气温和专业。"
```

## 训练产物说明

本仓库的 [my_hotel_model_lora/](file:///e:/乱七八糟/骆炜权_Qwem2.5-7b_酒店管理/my_hotel_model_lora) 为 LoRA 训练导出的结果目录，通常包含：

- `adapter_model.safetensors`：LoRA 权重
- `adapter_config.json`：LoRA 配置
- `chat_template.jinja`、`tokenizer*.json`：对话模板与分词器配置（训练时导出）
- `checkpoint-*`：训练过程中的 checkpoint（包含优化器等状态文件）

