---
name: datathings-marketplace
description: >-
  Knowledge base for the datathings/marketplace Claude Code plugin ecosystem — GreyCat graph database,
  local LLM inference (Ollama, llama.cpp, vLLM, ggml), GPU computing (CUDA, OpenCL, ROCm),
  numerical math (BLAS/LAPACK), and power grid analysis (pandapower, power-grid-model).
  Also guides creating new Claude Code marketplace plugins following the datathings structure.
  Use when user mentions GreyCat, GCL, greycat.io, datathings, Kopr, or any of the supported
  libraries, or when creating Claude Code plugins/skills for a marketplace.
---

# Datathings Marketplace — Claude Code Plugins & Skills

The datathings/marketplace provides Claude Code plugins for:
- **GreyCat** — temporal graph database + programming language (GCL)
- **Agentic AI** — llama.cpp, ggml, vLLM, Ollama (local LLM inference)
- **GPU Computing** — NVIDIA CUDA, OpenCL, AMD ROCm
- **Math** — BLAS/LAPACK (numerical linear algebra)
- **Power Grid** — pandapower, power-grid-model

## Quick Setup (Claude Code)

```bash
# 1. Install GreyCat (Windows PowerShell)
iwr https://get.greycat.io/install_dev.ps1 -useb | iex

# 2. Add marketplace to Claude Code
/plugin marketplace add datathings/marketplace

# 3. Install desired plugin
/plugin install greycat@datathings
/plugin install ollama@datathings
/plugin install cuda@datathings
```

## Available Plugins

| Plugin | Category | Description |
|--------|----------|-------------|
| `greycat` | GreyCat Tech | Full-stack GCL development, LSP, graph persistence |
| `greycat-c` | GreyCat Tech | GreyCat C API for native extensions |
| `llamacpp` | Agentic AI | llama.cpp C API (163 functions), local LLM inference |
| `ggml` | Agentic AI | ggml tensor library (560+ functions), GGUF I/O |
| `vllm` | Agentic AI | vLLM Python, high-throughput inference, OpenAI-compat |
| `ollama` | Agentic AI | Ollama REST API on localhost:11434 |
| `blas_lapack` | Math & GPU | CBLAS + LAPACKE (1284 functions) |
| `cuda` | Math & GPU | NVIDIA CUDA Runtime, cuBLAS, cuFFT, cuSPARSE, Thrust |
| `opencl` | Math & GPU | Khronos OpenCL C/C++ cross-platform GPU |
| `rocm` | Math & GPU | AMD ROCm, HIP kernels, rocBLAS/rocFFT |
| `pandapower` | Power Grid | AC/DC power flow, OPF, IEC 60909 short-circuit |
| `powergridmodel` | Power Grid | High-performance distribution grid analysis |

## Plugin Configuration (`~/.claude/settings.json`)

```json
{
  "extraKnownMarketplaces": {
    "datathings": {
      "source": { "source": "github", "repo": "datathings/marketplace" }
    }
  },
  "enabledPlugins": {
    "greycat@datathings": true,
    "ollama@datathings": true
  }
}
```

## GreyCat Quick Reference

GreyCat is a database AND programming language. Files use `.gcl` extension.

```gcl
// Define a node type (persistent by default)
@node
type City {
  name: String;
  population: int;
  country: Country;
}

// nodeIndex = indexed collection
var cities: nodeIndex<City>;

// Expose as HTTP API
@expose
fn getCities(): Array<City> {
  return cities.all();
}

// Expose as MCP endpoint
@mcp
fn searchCity(name: String): City? {
  return cities.findBy("name", name);
}
```

### GreyCat Project Structure

```
project/
├── project.gcl       # Entry point (main fn)
├── model/            # Node type definitions
├── api/              # @expose functions
└── data/             # CSV/seed data
```

### Start GreyCat Server

```bash
greycat serve          # Start on port 8080
greycat run            # Run script once
greycat-lang --version # Verify LSP installation
```

## Creating a New Marketplace Plugin

Follow the datathings plugin structure. See [plugin-structure.md](references/plugin-structure.md) for full details.

### Directory Layout

```
plugins/<name>/
├── .claude-plugin/
│   └── plugin.json        # Plugin metadata
└── skills/<name>/
    ├── SKILL.md            # Main skill (required)
    └── references/         # Optional reference docs
        ├── api-*.md
        └── workflows.md
```

### `plugin.json` Template

```json
{
  "name": "my-plugin",
  "version": "1.0.0",
  "description": "Short description for Claude Code plugin list",
  "author": { "name": "Your Name", "email": "you@example.com" },
  "license": "MIT",
  "repository": "https://github.com/your/marketplace",
  "keywords": ["tag1", "tag2"],
  "skills": "./skills/"
}
```

### `SKILL.md` Template

```markdown
---
name: my-plugin
description: "What this skill does and WHEN to use it (include trigger keywords)"
---

# My Plugin Title

## Key Concepts
[2-4 essential concepts the agent needs to understand]

## Quick Start
[Minimal working example]

## API Reference
[Table of main functions/endpoints]

## Detailed References
- [Full API docs](references/api.md)
- [Workflow examples](references/workflows.md)
```

### Register in `marketplace.json`

```json
{
  "skills": [
    {
      "name": "my-plugin",
      "path": "./plugins/my-plugin/skills/my-plugin",
      "version": "1.0.0",
      "description": "Brief description"
    }
  ]
}
```

## Management Commands

```bash
/plugin list                          # List installed plugins
/plugin update greycat@datathings     # Update plugin
/plugin uninstall greycat@datathings  # Remove plugin
/plugin marketplace list              # List registered marketplaces
/plugin marketplace remove datathings # Remove marketplace
```

## Detailed References

- [Plugin structure & templates](references/plugin-structure.md)
- [GreyCat GCL language guide](references/greycat-quickstart.md)
- [LLM inference tools (Ollama, vLLM, llama.cpp, ggml)](references/llm-tools.md)

## Key Links

- GreyCat docs: https://doc.greycat.io
- GreyCat install: https://get.greycat.io
- Marketplace repo: https://github.com/datathings/marketplace
- Claude Code docs: https://code.claude.com/docs/en/setup
