---
layout: post
title: "vLLM 部署 Qwen3.5-2B 简记"
date: 2026-04-04
categories: ai llm
tags: vllm qwen gpu inference
---

## 1. 当前结论

-   `Qwen/Qwen3.5-2B` 已可通过 `vLLM` 正常启动并提供 OpenAI-compatible API。
    
-   机器有 2 张 GPU，每张 `48GB` 显存。
    
-   双卡启动成功后，可在 `nvidia-smi` 中看到：
    
    -   `VLLM::Worker_TP0`
        
    -   `VLLM::Worker_TP1`
        
-   这表示启用了 `tensor parallel`，模型已切分到两张卡上运行。
    

## 2. 常用 `nvidia-smi` 字段

| 字段  | 含义  | 如何理解 |
| --- | --- | --- |
| `Memory-Usage` | 当前显存占用 / 总显存 | 比如 `45349MiB / 49152MiB` 表示这张卡已占约 45.3GB |
| `GPU-Util` | GPU 计算忙碌度 | 类似 CPU usage，但更像“采样周期内 GPU 有多忙” |
| `Compute M.` | Compute Mode | 一般是 `Default`，表示可正常用于 CUDA 计算 |
| `Processes` | 当前 GPU 上的进程 | 可直接看到是哪个进程在占用显存 |

## 3. `GPU-Util` 是什么

-   `GPU-Util` 类似 CPU 使用率，但不是同一个概念。
    
-   它表示 GPU 计算核心在最近一个采样窗口内的忙碌比例。
    
-   例子：
    
    -   `4%`：基本空闲，通常只是服务刚启动完或没有实际请求。
        
    -   `50%+`：正在执行推理，GPU 有实质负载。
        

结合本次观察：

-   服务空闲时，`GPU-Util` 会很低。
    
-   发起一次较长请求后，双卡都升到 `50%+`，说明推理正在运行。
    

## 4. 显存占用在哪里控制

主要参数：

```bash
--gpu-memory-utilization
```

含义：

-   控制 `vLLM` 最多使用每张可见 GPU 的多少比例显存。
    
-   例如单卡 48GB，若设置：
    

```bash
--gpu-memory-utilization 0.90
```

则 `vLLM` 会尽量将每张卡使用到约：

```text
48GB * 0.90 ≈ 43.2GB
```

实际使用量会略有浮动。

常见值：

| 参数  | 大致单卡上限 |
| --- | --- |
| `0.50` | 约 24GB |
| `0.60` | 约 29GB |
| `0.70` | 约 34GB |
| `0.90` | 约 43GB |

此外显存还受这些参数影响：

-   `--max-model-len`
    
-   `--tensor-parallel-size`
    
-   是否启用多模态相关组件
    

## 5. 指定 GPU 的方法

用环境变量：

```bash
CUDA_VISIBLE_DEVICES=0
CUDA_VISIBLE_DEVICES=1
CUDA_VISIBLE_DEVICES=0,1
```

这表示让当前进程只看到指定的物理 GPU。

查看当前 shell 是否设置了这个变量：

```bash
echo $CUDA_VISIBLE_DEVICES
```

只对一条命令临时生效的写法：

```bash
CUDA_VISIBLE_DEVICES=0 vllm serve ...
```

对当前 shell 后续命令都生效的写法：

```bash
export CUDA_VISIBLE_DEVICES=0
```

取消：

```bash
unset CUDA_VISIBLE_DEVICES
```

## 6. 启动命令示例

### 单卡启动，使用 GPU 0

```bash
CUDA_VISIBLE_DEVICES=0 vllm serve /data/models/Qwen3.5-2B \
  --host 0.0.0.0 \
  --port 8000 \
  --served-model-name Qwen3.5-2B \
  --tensor-parallel-size 1 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90 \
  --enable-prefix-caching
```

### 单卡启动，使用 GPU 1

```bash
CUDA_VISIBLE_DEVICES=1 vllm serve /data/models/Qwen3.5-2B \
  --host 0.0.0.0 \
  --port 8000 \
  --served-model-name Qwen3.5-2B \
  --tensor-parallel-size 1 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90
```

### 双卡启动

```bash
CUDA_VISIBLE_DEVICES=0,1 vllm serve /data/models/Qwen3.5-2B \
  --host 0.0.0.0 \
  --port 8000 \
  --served-model-name Qwen3.5-2B \
  --tensor-parallel-size 2 \
  --max-model-len 8192 \
  --gpu-memory-utilization 0.90
```

说明：

-   `CUDA_VISIBLE_DEVICES=0,1`：让进程看到两张卡。
    
-   `--tensor-parallel-size 2`：把模型切到两张卡上并行运行。
    

如果只写了：

```bash
CUDA_VISIBLE_DEVICES=0,1
```

但仍然使用：

```bash
--tensor-parallel-size 1
```

通常不会自动双卡切分。

## 7. 如何判断服务是否真的起来了

先看端口：

```bash
ss -lntp | grep 8000
```

看进程：

```bash
ps -ef | grep -E "vllm|api_server|EngineCore|Worker_TP" | grep -v grep
```

看显卡占用：

```bash
nvidia-smi
```

接口探活：

```bash
curl http://127.0.0.1:8000/v1/models
```

## 8. API 测试命令

### 查看模型列表

```bash
curl http://127.0.0.1:8000/v1/models
```

### 发送对话请求

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer EMPTY" \
  -d '{
    "model": "Qwen3.5-2B",
    "messages": [
      {"role": "user", "content": "单词 'strawberry' 中有几个字母 'r'？"}
    ],
    "max_tokens": 128,
    "temperature": 0.7
  }'
```

### 本次已验证的长文本测试

```bash
curl http://127.0.0.1:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer EMPTY" \
  -d '{
    "model": "Qwen3.5-2B",
    "messages": [
      {"role": "user", "content": "写2000字论文 主题是AI革命 人类失业"}
    ],
    "max_tokens": 8000,
    "temperature": 0.7
  }'
```

## 9. 本次现象说明

结合当前 `nvidia-smi`：

| GPU | 显存占用 | 利用率 | 进程  |
| --- | --- | --- | --- |
| `GPU 0` | `45349MiB / 49152MiB` | `56%` | `VLLM::Worker_TP0` |
| `GPU 1` | `45349MiB / 49152MiB` | `52%` | `VLLM::Worker_TP1` |

这说明：

-   服务当前是双卡运行。
    
-   两张卡都在参与推理。
    
-   当前请求执行时，GPU 负载已明显上升，不再是空闲状态。
    

## 10. 排障要点

-   如果 `vllm` 命令不存在，先确认当前 virtualenv 里是否真的安装了 `vllm`。
    
-   如果报 `transformers does not recognize this architecture`，通常是 `transformers` 太旧。
    
-   如果报 `architecture not supported for now`，通常是 `vLLM` 版本太旧，需要更新到更高版本。
    
-   如果端口没监听但 GPU 有显存占用，通常还在启动、编译或 warmup 阶段。
