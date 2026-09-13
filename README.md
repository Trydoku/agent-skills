# TRYDOKU Agent Skills

Official agent skills for generating documents with [TRYDOKU](https://www.trydoku.com) from AI coding tools.

The [`trydoku-generation`](skills/trydoku-generation/SKILL.md) skill guides an agent through preparing a Word (`.docx`) template, mapping data, submitting a generation batch, checking its status, and downloading the results. It supports existing TRYDOKU templates and local `.docx` files encoded as base64. CSV and Excel data must be converted to JSON before submission to the REST API.

This repository provides agent instructions, not an API client or a local document renderer. Generation runs on TRYDOKU and uses your account's credits.

## Install

Clone the repository, then copy the skill into the directory used by your agent. The shell examples below use Bash or Zsh on macOS or Linux; on other platforms, copy the folder to the equivalent location.

```bash
git clone https://github.com/Trydoku/agent-skills.git
cd agent-skills
```

Choose a project installation to share the skill with a project, or a personal installation to make it available across your projects.

| Agent | Project skills directory | Personal skills directory | Official instructions |
| --- | --- | --- | --- |
| Claude Code | `.claude/skills/` | `~/.claude/skills/` | [Claude Code skills](https://code.claude.com/docs/en/skills) |
| Cursor | `.cursor/skills/` | `~/.cursor/skills/` | [Cursor skills](https://cursor.com/docs/skills) |
| Codex | `.agents/skills/` | `~/.agents/skills/` | [Codex skills](https://learn.chatgpt.com/docs/build-skills) |
| Antigravity IDE | `.agents/skills/` | `~/.gemini/config/skills/` | [Antigravity skills](https://antigravity.google/docs/skills) |

### Project installation

Run this from the cloned repository. Replace `/path/to/your-project` with the project where you want to use the skill, and select the skills directory from the table. For example, for Codex or Antigravity IDE:

```bash
mkdir -p /path/to/your-project/.agents/skills
cp -R skills/trydoku-generation /path/to/your-project/.agents/skills/
```

For Claude Code, use `.claude/skills`; for Cursor, use `.cursor/skills`. Cursor also supports `.agents/skills`, which can be shared with Codex and Antigravity IDE.

### Personal installation

For example, to install for Codex across your projects:

```bash
mkdir -p ~/.agents/skills
cp -R skills/trydoku-generation ~/.agents/skills/
```

Use the corresponding personal directory from the table for another agent. Copy the entire `trydoku-generation` folder; `SKILL.md` must sit directly inside it. Install one copy per agent and scope to avoid competing versions.

For an agent without native skill support, ask it to read `skills/trydoku-generation/SKILL.md` from this repository before working with the TRYDOKU API. Automatic discovery depends on that agent's capabilities.

## Configure authentication

Create a token under **Settings → API Keys** in TRYDOKU, then make it available as `TRYDOKU_API_TOKEN` in the environment that runs your agent's commands. See the [API authentication guide](https://www.trydoku.com/docs/api#authentication).

Use your environment's secret manager or protected environment settings. A `.env` file works only if your tooling explicitly loads it; placing a token in the file does not export it to an agent automatically. Keep tokens out of version control, prompts, and logs. Environment variables are not encrypted storage.

Check availability without printing the token:

```bash
if [ -n "${TRYDOKU_API_TOKEN:-}" ]; then
  echo "TRYDOKU_API_TOKEN is available."
else
  echo "Set TRYDOKU_API_TOKEN in the agent's execution environment." >&2
fi
```

A token is needed for API requests. Inspecting local templates and preparing data can be done without one.

## Use the skill

After installation, ask your agent to use the skill by name. For example:

> Use trydoku-generation to inspect `invoice-template.docx` and `customers.csv`. Check the placeholders and prepare the JSON payload without submitting a batch.

To generate documents:

> Use trydoku-generation to generate documents from `invoice-template.docx` and `customers.csv`, download the ZIP into `output/`, and report any failed rows.

If the agent does not find the skill, verify the installation path and reload its session. You can also point it directly to the installed `SKILL.md`.

## Documentation and maintenance

The installation guidance and API details were checked against the published documentation on **2026-09-13**. Consult the current references when updating integrations:

- [TRYDOKU API overview](https://www.trydoku.com/docs/api)
- [Interactive API reference](https://www.trydoku.com/docs/api/reference) and [OpenAPI JSON](https://www.trydoku.com/docs/api/api.json)
- [Template syntax and data preparation](https://www.trydoku.com/docs)
- [Word Template Variable Parser](https://www.trydoku.com/tools/word-template-variable-parser)
- [Pricing and credits](https://www.trydoku.com/pricing)
- [Security and data handling](https://www.trydoku.com/security)

When updating the skill, verify endpoint fields, request limits, response handling, and template syntax against these sources. Keep installation instructions aligned with each agent's official documentation. Local validation of instructions and examples does not verify authenticated API behavior.

## License

[MIT](LICENSE) © 2026 TRYDOKU.
