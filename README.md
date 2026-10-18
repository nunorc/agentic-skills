# agentic-skills

[![skills.sh](https://skills.sh/b/nunorc/agentic-skills)](https://skills.sh/nunorc/agentic-skills)

A personal collection of specialized agent skills. Each skill lives in its own
directory under `skills/` and follows the standard skill layout:

```
skills/<skill-name>/
├── SKILL.md        # frontmatter (name, description) + full instructions
└── references/     # optional supporting material (sources, templates, scripts)
```

## Skills

- **[deep-book-summary](skills/deep-book-summary/SKILL.md)** — deep-research a
  non-fiction book and produce a structured markdown summary with personal
  reflection, cross-checked against independent sources.

## Usage

Install skills with the [skills.sh](https://skills.sh) CLI (requires this repo
to be pushed to GitHub):

```sh
# list available skills
npx skills add nunorc/agentic-skills --list

# install a specific skill
npx skills add nunorc/agentic-skills --skill deep-book-summary

# install all skills to all detected agents
npx skills add nunorc/agentic-skills --all
```

The CLI auto-detects installed agents (Claude Code, OpenCode, Cursor, Codex, …)
and symlinks each skill into the agent's skills directory. Add `-g` to install
globally instead of per-project.

Alternatively, symlink a skill manually:

```sh
ln -s "$(pwd)/skills/deep-book-summary" ~/.claude/skills/deep-book-summary
```

## Adding a new skill

1. Create a directory under `skills/` named after the skill (lowercase, hyphenated).
2. Add a `SKILL.md` with frontmatter:
   ```yaml
   ---
   name: <skill-name>
   description: <what it does and when to use it>
   ---
   ```
3. Put helper scripts, templates, and source material in `references/`.
4. List the skill in this README.

## License

MIT — see [LICENSE](LICENSE).
