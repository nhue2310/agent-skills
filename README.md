# Agent Skills

A collection of agent and skill definitions for Claude, CodeX, and Gemini AI platforms.

## Overview

This repository provides standardized YAML definitions for AI agents and skills/tools that can be used across multiple AI platforms. Each definition follows a consistent schema that describes the agent's behavior, model parameters, and available tools.

## Repository Structure

```
agent-skills/
├── agents/              # Agent definitions by platform
│   ├── claude/          # Anthropic Claude agents
│   ├── codex/           # OpenAI CodeX agents
│   └── gemini/          # Google Gemini agents
├── skills/              # Skill/tool definitions
│   └── common/          # Skills usable across multiple platforms
└── schemas/             # JSON schemas for validation
    ├── agent-schema.json
    └── skill-schema.json
```

## Agents

### Claude (Anthropic)

| Agent | Description |
|-------|-------------|
| [code-assistant](agents/claude/code-assistant.yaml) | Coding assistant for code review, debugging, and generation |
| [research-assistant](agents/claude/research-assistant.yaml) | Research and information synthesis agent |

### CodeX (OpenAI)

| Agent | Description |
|-------|-------------|
| [code-generation](agents/codex/code-generation.yaml) | Produces complete, production-ready code from descriptions |
| [debugging-agent](agents/codex/debugging-agent.yaml) | Identifies and fixes bugs in existing code |

### Gemini (Google)

| Agent | Description |
|-------|-------------|
| [multimodal-assistant](agents/gemini/multimodal-assistant.yaml) | Processes and analyzes text, images, audio, and video |
| [data-analysis](agents/gemini/data-analysis.yaml) | Analyzes datasets and generates insights |

## Skills

Skills are reusable tool definitions that agents can invoke. They are defined independently of any specific agent and can be shared across platforms.

### Common Skills

| Skill | Category | Platforms |
|-------|----------|-----------|
| [web_search](skills/common/web-search.yaml) | information_retrieval | claude, codex, gemini |
| [read_file](skills/common/read-file.yaml) | file_operations | claude, codex, gemini |
| [write_file](skills/common/write-file.yaml) | file_operations | claude, codex, gemini |
| [code_interpreter](skills/common/code-interpreter.yaml) | code_execution | codex, gemini |
| [search_code](skills/common/search-code.yaml) | code_operations | claude, codex, gemini |
| [run_tests](skills/common/run-tests.yaml) | code_operations | codex, gemini |
| [summarize_document](skills/common/summarize-document.yaml) | information_processing | claude, gemini |
| [image_analysis](skills/common/image-analysis.yaml) | multimodal | claude, gemini |
| [audio_transcription](skills/common/audio-transcription.yaml) | multimodal | gemini |
| [video_analysis](skills/common/video-analysis.yaml) | multimodal | gemini |
| [data_visualization](skills/common/data-visualization.yaml) | data_analysis | codex, gemini |

## Definition Format

### Agent Definition

```yaml
name: my-agent             # Unique kebab-case identifier
description: ...           # What the agent does
model: claude-opus-4-5    # Specific model to use
platform: claude           # claude | codex | gemini

system_prompt: |
  Your agent's behavior and instructions...

parameters:
  max_tokens: 4096
  temperature: 0.2

tools:
  - web_search
  - read_file
```

### Skill Definition

```yaml
name: my_skill             # Unique snake_case identifier
description: ...           # What the skill does
category: file_operations  # Skill category

input_schema:
  type: object
  properties:
    param_name:
      type: string
      description: Parameter description
  required:
    - param_name

output_schema:
  type: object
  properties:
    result:
      type: string

compatible_platforms:
  - claude
  - codex
  - gemini
```

## Schemas

All definitions are validated against JSON schemas located in the `schemas/` directory:

- [`agent-schema.json`](schemas/agent-schema.json) — validates agent definition files
- [`skill-schema.json`](schemas/skill-schema.json) — validates skill definition files

## Contributing

1. Create a new YAML file in the appropriate `agents/<platform>/` or `skills/common/` directory
2. Follow the existing definition format and naming conventions
   - Agent files: `kebab-case.yaml`
   - Skill names: `snake_case`
3. Validate your definition against the relevant JSON schema
4. Submit a pull request with a clear description of the new agent or skill
