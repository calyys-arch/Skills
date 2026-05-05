---
name: vllm
description: "vLLM high-throughput Python LLM inference reference. Use when working with vLLM, OpenAI-compatible model serving, offline batch inference, LoRA adapters, multimodal inputs, structured outputs, paged attention, or production LLM serving performance."
---

# vLLM Reference

Use this skill when the user is building or debugging applications with vLLM, including local or server-based large language model inference, OpenAI-compatible APIs, batch generation, LoRA adapters, structured outputs, and high-throughput serving.

## Core Concepts

| Concept | Purpose |
|--------|---------|
| `LLM` | Offline inference entry point for Python batch generation |
| `SamplingParams` | Controls temperature, top-p, max tokens, stop tokens, and related decoding behavior |
| `vllm serve` | Starts an OpenAI-compatible HTTP server |
| PagedAttention | vLLM memory management mechanism for efficient KV cache usage |
| LoRA | Adapter-based fine-tuning support at inference time |
| Structured outputs | JSON, regex, or grammar-constrained generation |

## Quick Start: Offline Inference

```python
from vllm import LLM, SamplingParams

prompts = [
    "Explain paged attention in one paragraph.",
    "Write a short SQL query that selects active users.",
]

sampling_params = SamplingParams(
    temperature=0.7,
    top_p=0.95,
    max_tokens=256,
)

llm = LLM(model="Qwen/Qwen2.5-7B-Instruct")
outputs = llm.generate(prompts, sampling_params)

for output in outputs:
    print(output.prompt)
    print(output.outputs[0].text)
```

## OpenAI-Compatible Server

```bash
vllm serve Qwen/Qwen2.5-7B-Instruct --host 0.0.0.0 --port 8000
```

Then call it using an OpenAI-compatible client:

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY",
)

response = client.chat.completions.create(
    model="Qwen/Qwen2.5-7B-Instruct",
    messages=[{"role": "user", "content": "Summarize vLLM in one sentence."}],
)

print(response.choices[0].message.content)
```

## Common Serving Options

| Option | Use |
|--------|-----|
| `--tensor-parallel-size` | Split model execution across multiple GPUs |
| `--gpu-memory-utilization` | Control how much GPU memory vLLM can reserve |
| `--max-model-len` | Limit context length for memory control |
| `--enable-lora` | Enable LoRA adapter serving |
| `--served-model-name` | Expose a custom model name through the API |
| `--dtype` | Select model precision, such as `auto`, `float16`, or `bfloat16` |

## Structured Output Guidance

Use structured outputs when downstream systems require machine-parseable responses. Prefer schema-constrained JSON for APIs and regex or grammar constraints for compact formats.

```python
from vllm import LLM, SamplingParams

params = SamplingParams(
    temperature=0,
    max_tokens=128,
)

llm = LLM(model="Qwen/Qwen2.5-7B-Instruct")
outputs = llm.generate(
    ["Return a JSON object with keys name and purpose for vLLM."],
    params,
)
```

## Troubleshooting Checklist

- If model loading fails, verify model name, local path, Hugging Face access, and available GPU memory.
- If throughput is low, check batch size, context length, tensor parallelism, and GPU utilization.
- If requests are rejected, confirm the client model name matches the served model name.
- If output is malformed, lower temperature and use structured output constraints where available.
- If memory usage is unstable, reduce `--max-model-len` or `--gpu-memory-utilization`.

## When to Use Another Tool

- Use `ollama` for simple local model management and desktop-friendly REST inference.
- Use `llamacpp` for GGUF models, CPU-friendly inference, and C/C++ embedding.
- Use `ggml` for low-level tensor graph and quantization work.
