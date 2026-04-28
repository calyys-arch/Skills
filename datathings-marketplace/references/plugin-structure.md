# Claude Code Marketplace Plugin Structure

Based on the datathings/marketplace pattern (https://github.com/datathings/marketplace).

## Full Directory Structure

```
marketplace-root/
├── .claude-plugin/
│   └── marketplace.json        # Lists all plugins for /plugin marketplace add
├── marketplace.json            # Skills listing (alternative format)
├── plugins/
│   └── <plugin-name>/
│       ├── .claude-plugin/
│       │   └── plugin.json     # Plugin metadata (used by Claude Code)
│       ├── .gitignore
│       └── skills/
│           └── <plugin-name>/
│               ├── SKILL.md    # Main skill — REQUIRED
│               └── references/ # Optional deep-dive docs
│                   ├── api-*.md
│                   └── workflows.md
├── skills/                     # Pre-packaged .skill files (zip archives)
│   └── <plugin-name>.skill
├── README.md
├── LICENSE
├── bump-version.sh             # Bump all plugin versions at once
└── package.sh                  # Package skills into .skill zip files
```

## `.claude-plugin/marketplace.json` — Marketplace Registration

This file registers the marketplace so Claude Code can discover all plugins:

```json
{
  "name": "marketplace-name",
  "license": "Apache-2.0",
  "owner": {
    "name": "Author Name",
    "email": "author@example.com"
  },
  "plugins": [
    {
      "name": "plugin-one",
      "source": "./plugins/plugin-one",
      "description": "One-line description for /plugin list",
      "version": "1.0.0"
    },
    {
      "name": "plugin-two",
      "source": "./plugins/plugin-two",
      "description": "One-line description",
      "version": "1.0.0"
    }
  ]
}
```

## `plugins/<name>/.claude-plugin/plugin.json` — Plugin Metadata

```json
{
  "name": "plugin-name",
  "version": "1.0.0",
  "description": "Detailed description shown when plugin is installed",
  "author": {
    "name": "Author Name",
    "email": "author@example.com"
  },
  "license": "MIT",
  "repository": "https://github.com/your/marketplace",
  "keywords": ["keyword1", "keyword2", "keyword3"],
  "skills": "./skills/"
}
```

- `"skills": "./skills/"` — points to the skills directory relative to the plugin root
- `keywords` — help users discover the plugin

## `SKILL.md` Structure

The agent reads `SKILL.md` to understand how to use the technology.

### YAML Frontmatter (Required)

```yaml
---
name: plugin-name              # max 64 chars, lowercase + hyphens
description: "WHAT it does and WHEN to use it. Include technology names,
  file extensions, import patterns, or error keywords as triggers."
---
```

### Recommended Body Sections

```markdown
# Plugin Title (Version)

## Key Concepts
- Bullet summary of the 3-5 most important concepts
- Focus on what's non-obvious

## Quick Start
[Minimal working code example — copy-paste ready]

## API Reference
| Function/Endpoint | Description |
|---|---|
| `fn_name(args)` | What it does |

## Common Patterns
[Patterns for the top 3-5 use cases]

## Detailed References
- [Full API](references/api.md)
- [Workflows](references/workflows.md)
```

### SKILL.md Best Practices

- Keep under 500 lines — defer detail to `references/`
- Use tables for API references (scannable)
- Include a copy-paste Quick Start
- Write trigger keywords into the frontmatter description
- Reference files are one level deep: `references/foo.md` (not nested)

## `references/` Files

Break large API documentation into focused files:

| Filename | Contents |
|----------|----------|
| `api-core.md` | Core/main API functions |
| `api-<subsystem>.md` | Subsystem-specific APIs |
| `workflows.md` | End-to-end workflow examples |
| `configuration.md` | Config options and flags |

### Reference File Pattern

```markdown
# Subsystem API Reference

## Function Group A

### `function_name(param: Type): ReturnType`
Description of what it does.

**Parameters:**
- `param` — what it means

**Example:**
\`\`\`c
result = function_name(value);
\`\`\`

---
```

## Installing a Local Marketplace (Development)

```bash
# Point Claude Code at a local directory
/plugin marketplace add /path/to/your/marketplace

# Then install plugins from it
/plugin install my-plugin@local-marketplace-name
```

## Versioning

The `bump-version.sh` pattern:

```bash
./bump-version.sh           # Show all current versions
./bump-version.sh 2.0.0     # Bump all plugins to 2.0.0
```

## Packaging `.skill` Files

`.skill` files are zip archives for standalone distribution:

```bash
./package.sh           # Interactive selection
./package.sh -a        # Package all
./package.sh greycat   # Package specific plugin
```

Structure inside the zip:
```
greycat.skill (zip)
├── SKILL.md
└── references/
    └── *.md
```

## Real-World Plugin Examples (datathings)

| Plugin | Skills Path | References Count |
|--------|-------------|-----------------|
| greycat | plugins/greycat/skills/greycat/ | ~15 reference files |
| ollama | plugins/ollama/skills/ollama/ | 5 reference files |
| cuda | plugins/cuda/skills/cuda/ | 8 reference files |
| blas_lapack | plugins/blas_lapack/skills/blas_lapack/ | 10 reference files |
