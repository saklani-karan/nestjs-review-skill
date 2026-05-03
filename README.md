# Backend PR Review (Skill)

A 12-rule rubric for reviewing NestJS/TypeScript backend pull requests, packaged as an [Agent Skill](https://agentskills.io). Works with Claude Code, Cursor, GitHub Copilot, OpenClaw, Codex CLI, Gemini CLI, and any other compatible agent.

The skill scores code against twelve weighted rules covering transactional management, repository/service architecture, abstract service patterns, event emission, design-pattern usage, folder structure, error handling, logging, circular dependencies, and dependency-vs-composition. Each rule has a penalty for violations and a reward for correct implementation; the agent produces a final score and merge recommendation.

## What it covers

| # | Rule | Penalty | Reward |
|---|------|--------:|-------:|
| 1 | Transactional management | -1000 / occurrence | +10 |
| 2 | Repository definition | -100 / occurrence | +50 |
| 3 | Service abstraction (Abstract + Impl) | -100 | +100 |
| 4 | Event emission pattern | -50 | +100 |
| 5 | Decorator design pattern | -50 | +100 |
| 6 | Folder structure | -200 | +50 |
| 7 | Strategy design pattern | -50 | +50 |
| 8 | Adapter design pattern | -50 | +50 |
| 9 | Error handling (ErrorTypes casting) | -200 | +50 |
| 10 | Request-scoped logging | -200 | +50 |
| 11 | Circular dependencies | -500 | +50 |
| 12 | Dependency vs Composition | -50 | +50 |

Rules #1 and #11 are critical and block the merge regardless of net score. See [`SKILL.md`](skills/nestjs-pr-review/SKILL.md) for the full workflow and [`references/`](skills/nestjs-pr-review/references/) for per-rule deep dives with examples, detection patterns, and edge cases.

## Install

**GitHub repository:** [`saklani-karan/nestjs-review-skill`](https://github.com/saklani-karan/nestjs-review-skill). **Skill id** (the `name` field in [`SKILL.md`](skills/nestjs-pr-review/SKILL.md), used for install paths and `gh skill update`): **`nestjs-pr-review`**.

The packaged skill lives at **`skills/nestjs-pr-review/SKILL.md`** so [`gh skill install`](https://cli.github.com/manual/gh_skill_install) can discover it (same layout as other GitHub skills publishers).

The fastest path on any supported agent host is the GitHub CLI. If you don't have `gh` installed (or your version is older than 2.90.0), jump to [Installing the GitHub CLI](#installing-the-github-cli) first. Otherwise, pick the section for your agent below.

### Claude Code

```bash
# Install for personal use (available across all your projects)
gh skill install saklani-karan/nestjs-review-skill

# Or pin to a specific release
gh skill install saklani-karan/nestjs-review-skill@v1.0.1
```

Skills land in `~/.claude/skills/nestjs-pr-review/` (the skill name from `SKILL.md`, not the repo folder name). Live change detection means edits take effect within the current session — no restart needed.

For project-scoped install (commit it to your repo so teammates get it on clone):

```bash
gh skill install saklani-karan/nestjs-review-skill --scope project
```

### Cursor

```bash
gh skill install saklani-karan/nestjs-review-skill --agent cursor
```

This drops the skill into `.cursor/skills/nestjs-pr-review/` in the current project. Cursor is project-scoped only — there's no global skills directory, so each project that needs the skill must install it.

After install, run `Developer: Reload Window` from the command palette (or restart Cursor) so the skill is picked up.

### Other agent hosts

`gh skill install` auto-detects most major hosts: GitHub Copilot, OpenClaw, Codex CLI, Gemini CLI, and others. Pass `--agent <name>` to target a specific one if you have multiple installed.

```bash
gh skill install saklani-karan/nestjs-review-skill --agent <agent-name>
```

Run `gh skill install --help` for the full list of supported agents.

## Manual install (no GitHub CLI)

If you can't or don't want to install `gh`, clone the repository and copy the skill folder (`skills/nestjs-pr-review/`) into your agent’s skills directory. The repo root is not a skill by itself — only that subdirectory contains `SKILL.md` and `references/`.

**Claude Code (personal):**
```bash
git clone https://github.com/saklani-karan/nestjs-review-skill.git /tmp/nestjs-review-skill \
  && cp -R /tmp/nestjs-review-skill/skills/nestjs-pr-review ~/.claude/skills/ \
  && rm -rf /tmp/nestjs-review-skill
```

**Claude Code (project, commit-friendly):**
```bash
git clone https://github.com/saklani-karan/nestjs-review-skill.git /tmp/nestjs-review-skill \
  && cp -R /tmp/nestjs-review-skill/skills/nestjs-pr-review .claude/skills/ \
  && rm -rf /tmp/nestjs-review-skill
```

**Cursor:**
```bash
git clone https://github.com/saklani-karan/nestjs-review-skill.git /tmp/nestjs-review-skill \
  && cp -R /tmp/nestjs-review-skill/skills/nestjs-pr-review .cursor/skills/ \
  && rm -rf /tmp/nestjs-review-skill
```

The trade-off vs `gh skill install`: no provenance metadata, no `gh skill update` support. To update manually, `cd` into the skill folder and `git pull`.

## Verifying installation

Open your agent and ask something that should trigger the skill, e.g.:

> "Review this PR against my backend standards."

You should see the agent reference the rubric and produce a structured score with findings. If nothing happens:

- Confirm the skill is present at the expected path.
- For Cursor, reload the window.
- For Claude Code, ask "What skills are available?" — `nestjs-pr-review` should be listed.
- Make sure your prompt mentions PR review, code review, or audit — the skill description triggers on those phrases.

## Updating

```bash
# With gh — checks all installed skills for upstream changes
gh skill update nestjs-pr-review

# Update every gh-installed skill at once
gh skill update --all

# Manual install (no `gh` metadata): re-run the [clone + copy steps](#manual-install-no-github-cli), or if you use a full git clone of this repo, `git pull` then copy `skills/nestjs-pr-review/` again.
```

## Installing the GitHub CLI

`gh skill install` requires GitHub CLI **v2.90.0 or later**. The CLI is free and available on macOS, Linux, and Windows.

**macOS** (Homebrew):
```bash
brew install gh
```

**Linux — Debian/Ubuntu** (official package list, since distro repos often lag):
```bash
(type -p wget >/dev/null || sudo apt install wget -y) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && wget -qO- https://cli.github.com/packages/githubcli-archive-keyring.gpg \
     | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg > /dev/null \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" \
     | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
  && sudo apt update \
  && sudo apt install gh -y
```

**Linux — Fedora/RHEL/CentOS:**
```bash
sudo dnf install gh
```

**Linux — Arch:**
```bash
sudo pacman -S github-cli
```

**Windows** (winget):
```powershell
winget install --id GitHub.cli
```

**Windows** (Scoop):
```powershell
scoop install gh
```

**Any OS — manual download:** grab the latest installer from <https://github.com/cli/cli/releases>.

After install, verify the version and authenticate:

```bash
gh --version       # must be 2.90.0 or later for `gh skill`
gh auth login      # opens a browser; sign in to your GitHub account
```

If your distro ships an older `gh`, use the official package list above instead — Debian and Ubuntu repos in particular often lag behind. For the full install matrix and troubleshooting, see <https://cli.github.com>.

## How it works

This skill uses progressive disclosure: `skills/nestjs-pr-review/SKILL.md` is a compact dispatcher with the scoring rubric, rule index, and review workflow. Each rule lives in its own self-contained reference file under `skills/nestjs-pr-review/references/`, loaded only when scoring that rule.

```
nestjs-review-skill/          ← Git repository (this repo)
├── README.md
├── LICENSE
└── skills/
    └── nestjs-pr-review/      ← skill package (what agents load)
        ├── SKILL.md
        └── references/
            ├── 01-transactional-management.md
            ├── 02-repository-definition.md
            ├── 03-service-definition.md
            ├── 04-event-emission.md
            ├── 05-decorator-pattern.md
            ├── 06-folder-structure.md
            ├── 07-strategy-pattern.md
            ├── 08-adapter-pattern.md
            ├── 09-error-handling.md
            ├── 10-logging.md
            ├── 11-circular-dependencies.md
            └── 12-dependency-vs-composition.md
```

After `gh skill install`, your agent sees only the inner `nestjs-pr-review/` tree (with `SKILL.md` at the skill root).

The rubric is opinionated for NestJS/TypeScript codebases but most rules transfer to other backend stacks (Spring, Django, Rails) with minor adjustments to the example code in the references.

## Customizing

The skill is intentionally readable and editable. To adapt for your team:

1. Fork the repo.
2. Edit `skills/nestjs-pr-review/SKILL.md` to adjust scoring weights, merge thresholds, or output format.
3. Edit individual files under `skills/nestjs-pr-review/references/` to refine examples, detection patterns, or naming conventions for your stack.
4. Republish under your own GitHub handle and install from there.

## Compatibility notes

- Pure markdown — no shell injection, no forked-subagent execution. Fully portable across every Agent Skills-compatible host.
- The `allowed-tools` frontmatter field is intentionally omitted so the skill doesn't pre-approve any tools. Your agent's normal permission settings stay in control.
- Claude Code-specific frontmatter fields (`context: fork`, `${CLAUDE_SKILL_DIR}` substitution, `` !`command` `` shell injection) are not used, ensuring Cursor and other hosts work identically.

## Contributing

Issues and PRs welcome. Particularly useful contributions:
- Detection patterns that catch additional violation cases.
- Real-world (sanitized) examples illustrating tricky edge cases.
- Adaptation guides for other backend frameworks.

## License

MIT. See [`LICENSE`](LICENSE).
