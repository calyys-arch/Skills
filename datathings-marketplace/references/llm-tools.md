# LLM Inference Tools Reference

Datathings marketplace provides 4 local LLM plugins: Ollama, vLLM, llama.cpp, and ggml.

---

## Ollama (v0.16.3+) — Easiest Local LLM

REST API on `http://localhost:11434`. Best for quick integration.

### Install & Start

```bash
# Install from https://ollama.com
ollama pull llama3.2         # Download model
ollama serve                 # Start server (auto-starts on most systems)
ollama list                  # Show downloaded models
```

### Key Endpoints

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/generate` | Text completion (streaming) |
| `POST` | `/api/chat` | Chat with history (streaming) |
| `POST` | `/api/embed` | Generate embeddings |
| `GET` | `/api/tags` | List local models |
| `POST` | `/api/pull` | Download a model |
| `DELETE` | `/api/delete` | Remove a model |
| `GET` | `/api/ps` | List loaded models |

### Quick Examples

```bash
# Non-streaming chat
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "messages": [{"role": "user", "content": "Hello!"}],
  "stream": false
}'

# Structured JSON output
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "List 3 colors in JSON",
  "format": {"type": "array", "items": {"type": "string"}},
  "stream": false
}'

# Embeddings
curl http://localhost:11434/api/embed -d '{
  "model": "all-minilm",
  "input": ["text to embed", "another text"]
}'
```

### Python Example

```python
import requests

def chat(prompt: str, model: str = "llama3.2") -> str:
    resp = requests.post("http://localhost:11434/api/chat", json={
        "model": model,
        "messages": [{"role": "user", "content": prompt}],
        "stream": False
    })
    return resp.json()["message"]["content"]

print(chat("What is GreyCat?"))
```

### Key Options

| Option | Default | Description |
|--------|---------|-------------|
| `temperature` | 0.8 | Randomness (0=deterministic, 1=creative) |
| `num_ctx` | 2048 | Context window size |
| `num_predict` | -1 | Max tokens (-1=unlimited) |
| `seed` | -1 | Random seed for reproducibility |
| `stream` | true | Stream tokens as generated |
| `keep_alive` | "5m" | Keep model loaded duration |

---

## vLLM (v0.16.0) — High-Throughput Python

Best for production inference at scale. OpenAI-compatible API.

### Install

```bash
pip install vllm
```

### Offline Batch Inference

```python
from vllm import LLM, SamplingParams

llm = LLM(model="meta-llama/Llama-3.2-1B")
params = SamplingParams(temperature=0.8, max_tokens=256)

outputs = llm.generate(["Hello!", "What is 2+2?"], params)
for output in outputs:
    print(output.outputs[0].text)
```

### OpenAI-Compatible Server

```bash
# Start server
vllm serve meta-llama/Llama-3.2-1B --port 8000

# Use with OpenAI client
python -c "
from openai import OpenAI
client = OpenAI(base_url='http://localhost:8000/v1', api_key='token')
r = client.chat.completions.create(
    model='meta-llama/Llama-3.2-1B',
    messages=[{'role':'user','content':'Hello!'}]
)
print(r.choices[0].message.content)
"
```

### Structured Output

```python
from pydantic import BaseModel
from vllm import LLM, SamplingParams

class City(BaseModel):
    name: str
    population: int

llm = LLM(model="meta-llama/Llama-3.2-1B")
params = SamplingParams(
    temperature=0,
    guided_decoding={"json": City.model_json_schema()}
)
outputs = llm.generate(["Capital of France with population?"], params)
```

---

## llama.cpp — C API for LLM Inference

163 C functions. Best for embedding in native applications or GreyCat C extensions.

### Key Function Groups

| Group | Functions | Description |
|-------|-----------|-------------|
| Model | `llama_model_load_from_file` | Load GGUF model |
| Context | `llama_init_from_model` | Create inference context |
| Tokenize | `llama_tokenize` | Text → tokens |
| Decode | `llama_decode` | Run inference batch |
| Sampling | `llama_sampler_*` | Token sampling strategies |
| GGUF | `gguf_init_from_file` | Read model metadata |

### Minimal Inference Loop (C)

```c
#include "llama.h"

llama_model_params mparams = llama_model_default_params();
llama_model *model = llama_model_load_from_file("model.gguf", mparams);

llama_context_params cparams = llama_context_default_params();
cparams.n_ctx = 2048;
llama_context *ctx = llama_init_from_model(model, cparams);

// Tokenize prompt
const char *prompt = "Hello";
int32_t tokens[256];
int n = llama_tokenize(model, prompt, strlen(prompt), tokens, 256, true, false);

// Decode
llama_batch batch = llama_batch_get_one(tokens, n);
llama_decode(ctx, batch);

// Sample next token
llama_sampler *sampler = llama_sampler_chain_init(llama_sampler_chain_default_params());
llama_sampler_chain_add(sampler, llama_sampler_init_temp(0.8f));
llama_sampler_chain_add(sampler, llama_sampler_init_dist(42));
llama_token next = llama_sampler_sample(sampler, ctx, -1);

// Cleanup
llama_sampler_free(sampler);
llama_free(ctx);
llama_model_free(model);
```

### Sampling Strategies

| Sampler | Function | Description |
|---------|----------|-------------|
| Temperature | `llama_sampler_init_temp(temp)` | Scale logits |
| Top-K | `llama_sampler_init_top_k(k)` | Keep top K tokens |
| Top-P | `llama_sampler_init_top_p(p, min)` | Nucleus sampling |
| Greedy | `llama_sampler_init_greedy()` | Always pick highest prob |
| XTC | `llama_sampler_init_xtc(p, t, min, seed)` | Exclude top choices |
| DRY | `llama_sampler_init_dry(...)` | Repetition avoidance |

---

## ggml — Tensor Computation Library

560+ C functions. The engine under llama.cpp. For custom ML operations.

### Core Concepts

```c
#include "ggml.h"

// Create computation context
struct ggml_init_params params = {
    .mem_size   = 128 * 1024 * 1024,  // 128MB
    .mem_buffer = NULL,
    .no_alloc   = false,
};
struct ggml_context *ctx = ggml_init(params);

// Create tensors
struct ggml_tensor *a = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 4, 4);
struct ggml_tensor *b = ggml_new_tensor_2d(ctx, GGML_TYPE_F32, 4, 4);

// Build computation graph (lazy — not executed yet)
struct ggml_tensor *c = ggml_mul_mat(ctx, a, b);  // matrix multiply
struct ggml_tensor *d = ggml_relu(ctx, c);         // activation

// Execute
struct ggml_cgraph *graph = ggml_new_graph(ctx);
ggml_build_forward_expand(graph, d);
ggml_graph_compute_with_ctx(ctx, graph, 4 /*threads*/);

// Read result
float *data = (float *)d->data;

ggml_free(ctx);
```

### Quantization Types

| Format | Bits/weight | Speed | Quality |
|--------|-------------|-------|---------|
| F32 | 32 | Slow | Best |
| F16 | 16 | Fast | Great |
| Q8_0 | 8 | Fast | Very good |
| Q4_K_M | ~4.5 | Fastest | Good |
| Q4_0 | 4 | Fastest | OK |

### GGUF Model I/O

```c
#include "gguf.h"

// Read GGUF metadata
struct gguf_init_params params = { .no_alloc = true, .ctx = NULL };
struct gguf_context *gguf = gguf_init_from_file("model.gguf", params);

// Read key-value metadata
int key_id = gguf_find_key(gguf, "general.architecture");
const char *arch = gguf_get_val_str(gguf, key_id);

int n_tensors = gguf_get_n_tensors(gguf);
for (int i = 0; i < n_tensors; i++) {
    printf("Tensor: %s\n", gguf_get_tensor_name(gguf, i));
}

gguf_free(gguf);
```

### Multi-Backend Support

ggml supports: CPU, CUDA, Metal (Apple), Vulkan, OpenCL, ROCm

```c
// Check available backends
ggml_backend_t backend;
if (ggml_cuda_available()) {
    backend = ggml_backend_cuda_init(0);  // GPU 0
} else {
    backend = ggml_backend_cpu_init();
}
```

---

## Choosing the Right Tool

| Use Case | Recommended Tool |
|----------|-----------------|
| Quick prototype, simple REST | **Ollama** |
| Production Python inference at scale | **vLLM** |
| Embedding LLM in C/GreyCat native | **llama.cpp** |
| Custom ML ops, training, GGUF I/O | **ggml** |
| OpenAI API compatibility | **vLLM** or **Ollama** |
| GPU-accelerated custom inference | **ggml** + CUDA backend |
