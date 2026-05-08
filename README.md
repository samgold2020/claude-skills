# claude-skill-react-testing-library

A [Claude Code skill](https://docs.claude.com/en/docs/claude-code/skills) capturing best practices and common mistakes for **React Testing Library** tests, distilled from Kent C. Dodds' [*Common Mistakes with React Testing Library*](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library).

When loaded, Claude will apply these patterns when writing, reviewing, or debugging RTL tests.

## What it covers

- Query priority (`*ByRole` first, test IDs last)
- `getBy*` vs `queryBy*` vs `findBy*` — which to use when
- `userEvent` vs `fireEvent`
- `waitFor` patterns (single assertion, no side-effects)
- jest-dom matchers vs raw DOM property assertions
- When `act()`, `cleanup()`, and `wrapper` are unnecessary
- ARIA on native elements

See [`SKILL.md`](./SKILL.md) for the full content.

## Install

Clone (or symlink) this repo into your user-level skills directory so it's picked up across all projects:

```sh
git clone https://github.com/samgold2020/claude-skill-react-testing-library.git \
  ~/.claude/skills/react-testing-library-best-practices
```

Or as a symlink, if you want to edit the repo in place:

```sh
git clone https://github.com/samgold2020/claude-skill-react-testing-library.git ~/code/claude-skill-react-testing-library
ln -s ~/code/claude-skill-react-testing-library ~/.claude/skills/react-testing-library-best-practices
```

To use it in a single project instead, clone into `<project>/.claude/skills/` with the same target name.

## Update

```sh
cd ~/.claude/skills/react-testing-library-best-practices  # or wherever you cloned
git pull
```

## License

MIT — see content adapted from Kent C. Dodds' blog post (linked above) for the original source.
