# claude-skills

A personal collection of reusable [Claude Code skills](https://docs.claude.com/en/docs/claude-code/skills). Each top-level directory is one skill; install the ones you want as symlinks into `~/.claude/skills/` so they load globally across every project.

## Skills

| Skill | Description |
|---|---|
| [`react-testing-library-best-practices`](./react-testing-library-best-practices) | Common mistakes and best practices for React Testing Library tests (queries, async, `userEvent`, `waitFor`, jest-dom matchers). Source: Kent C. Dodds. |

## Install

Clone the repo once, then symlink each skill you want into `~/.claude/skills/`:

```sh
git clone git@github.com:samgold2020/claude-skills.git ~/code/claude-skills

# Install all skills
for skill in ~/code/claude-skills/*/; do
  name=$(basename "$skill")
  ln -snf "$skill" ~/.claude/skills/"$name"
done

# Or install just one
ln -snf ~/code/claude-skills/react-testing-library-best-practices \
        ~/.claude/skills/react-testing-library-best-practices
```

To use a skill in a single project instead of globally, symlink into `<project>/.claude/skills/` with the same target name.

## Update

```sh
cd ~/code/claude-skills
git pull
```

Symlinks point at the working tree, so updates take effect immediately — no re-symlinking needed.

## Adding a new skill

1. Create a new top-level directory named after the skill (e.g. `my-new-skill/`).
2. Inside it, write `SKILL.md` with frontmatter:
   ```yaml
   ---
   name: my-new-skill
   description: One sentence on what this skill does and when to use it. The description is what Claude matches against to decide when to load it — be specific.
   ---
   ```
3. Add a row to the **Skills** table above.
4. Commit and push.

## License

MIT — see [LICENSE](./LICENSE). Individual skill content may credit external sources (linked from the skill's own `SKILL.md`).
