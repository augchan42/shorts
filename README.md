# shorts

A workflow for creating vertical social media shorts from long-form video content.

These skills were generated in Claude Code, but follow the open `SKILL.md` standard and are compatible with all major agentic CLI tools.

Used to generate Shorts for [hongkongaipodcast.com](https://hongkongaipodcast.com).

## Compatible Agents

The `SKILL.md` format (YAML frontmatter + markdown) pioneered by Claude Code/Anthropic is now an open standard supported by 27+ agents:

| Agent | Project Path | Agent | Project Path |
|---|---|---|---|
| **Claude Code** | `.claude/skills/` | **Goose** | `.goose/skills/` |
| **OpenCode** | `.claude/skills/`, `.agents/skills/` | **Roo Code** | `.roo/skills/` |
| **Cursor** | `.cursor/skills/` | **Windsurf** | `.windsurf/skills/` |
| **Codex CLI** | `.codex/skills/` | **Cline** | `.cline/skills/` |
| **Gemini CLI** | `.gemini/skills/` | **Aider** | `.aider/skills/` |
| **GitHub Copilot** | `.github/skills/` | **Continue** | — |
| **Kiro CLI** | `.kiro/skills/` | **Devin** | — |
| **Trae** | `.trae/skills/` | **Bolt** | — |
| **Amazon Q** | `.amazonq/skills/` | **Tabby** | `.tabby/skills/` |

Each agent uses its own skill directory. The open `SKILL.md` format works across all of them, but placement determines compatibility:

| Path | Compatibility |
|---|---|
| `.claude/skills/` | Claude Code, OpenCode |
| `.agents/skills/` | OpenCode, Cline, Aider, and most other tools |

For maximum portability across all agents, use `.agents/skills/`.

Features:
- Vertical 9:16 crop from widesource
- Smart speaker detection and framing
- Burned-in subtitles
- YouTube upload automation