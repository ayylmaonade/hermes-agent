# Wiki Schema

This document defines the conventions and taxonomy for the LLM Wiki.

## Directory Structure

- `entities/` - Specific organizations, companies, labs, people, etc.
  - `companies/` - AI companies and organizations
  - `labs/` - AI research labs
  - `models/` - AI models and model families
  - `open-source/` - Open-source projects and libraries
  - `people/` - Notable people in AI
  - `projects/` - AI projects and initiatives
- `concepts/` - Abstract concepts and ideas
- `index.md` - Main catalog of all wiki pages

## Tags

Use the following tag taxonomy:

### Entity Types
- `#company` - AI companies (OpenAI, Anthropic, Meta, etc.)
- `#lab` - AI research labs (DeepMind, etc.)
- `#model` - AI models (LLaMA, Qwen, etc.)
- `#open-source` - Open-source projects
- `#person` - Notable individuals
- `#project` - AI projects
- `#concept` - Abstract concepts

### Topics
- `#llm` - Large Language Models
- `#ai-safety` - AI Safety and alignment
- `#agents` - AI agents and autonomous systems
- `#cli` - Command-line tools
- `#foundation-model` - Foundation models

## Wikilink Format

Use double brackets for internal wiki links: `[[pagename]]` or `[[path/to/page]]`

## Sources/References Format

```markdown
## Sources/References

- [Description](URL) - Brief note about what this source is
```

## Page Naming Conventions

- Use lowercase with hyphens (e.g., `openai.md`, `anthropic.md`)
- Keep names concise and descriptive
- For model versions, include version in name (e.g., `llama-3.md`)
