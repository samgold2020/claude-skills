---
name: pr-design-enablement
description: Create a PR description using the Design Enablement team template. Covers the enhanced Summary prompt, Linear ticket field, Screenshots & Recordings section, Intentional Visual Changes, and the collapsible feature-flag/observability/rollout block. Detects net-new dependencies and conditionally adds Open Questions and Rejected Alternatives sections. Ask the user whether to collapse unused lower sections.
---

# PR Description — Design Enablement Template

Use this skill when creating or updating a PR description for the Design Enablement team. It uses a team-specific template derived from analysis of the team's PR patterns.

## Steps

### 1. Gather context

Run these commands to understand what the PR contains:

```bash
git log main..HEAD --oneline
git diff main...HEAD --stat
```

Also check if there's an existing PR body:

```bash
gh pr view --json body,title,url 2>/dev/null || true
```

### 2. Detect net-new dependencies

Check whether this PR introduces any net-new dependencies by inspecting the diff for additions to dependency manifests:

```bash
git diff main...HEAD -- package.json go.mod yarn.lock go.sum 2>/dev/null | grep '^+' | grep -v '^+++' | head -40
```

A "net-new dependency" means a package or module that wasn't present before this PR — not just a version bump. If you see additions to `dependencies` or `devDependencies` in `package.json`, or new `require` lines in `go.mod`, treat it as a net-new dependency.

If one or more net-new dependencies are detected, ask the user (using AskUserQuestion) two questions at once:

1. "Are there any open questions you'd like reviewers to focus on related to the new dependency/dependencies (e.g. security, bundle size, API surface)?" — free text, optional.
2. "Were there any alternatives you considered and rejected?" — free text, optional.

Store these answers. If either answer is non-empty, it will become a top-level section in the PR body (see step 5).

### 3. Check for an existing PR body

If the PR already exists and has a non-empty body (not just the default template scaffolding), **do not overwrite it**. Instead, skip to step 3 to ask about collapsing unused sections, and offer to make targeted additions (e.g. fill in a missing Linear ticket or strengthen an empty Summary) only if the user asks.

If no PR exists yet, or the body is blank/default template, proceed to draft below.

### 4. Draft the PR body

Use the template below. Fill in each section based on the diff and commit history.

**Summary** — the most important part is explaining what this PR changes and why. Before writing this section, ask the user (using AskUserQuestion):

> "Would you like to brainstorm the summary together, or are you happy for me to take the wheel and write it based on the diff?"

- If they want to brainstorm: ask "What's the core reason for this change?" and use their answer as the foundation of the summary narrative.
- If they want Claude to take the wheel: write a narrative summary based on the diff and commit history.

The summary should also address:
- What should reviewers focus on?
- Is there anything uncertain or worth flagging?

Write a narrative, not a changelog. Never produce a bulleted list of files changed — the reviewer will read the diff for that. Keep the summary brief: 1–3 sentences covering the high-level "what and why" is the target. Only expand beyond that when something genuinely needs calling out — a non-obvious decision, a subtle behavior change, a known limitation, or something the reviewer might otherwise miss.

**Linear** — extract from the branch name or commit messages if present (look for patterns like `APP-1234`, `ENG-567`, a full Linear URL). If not found, leave the placeholder.

**How to Test** — prefer specific Storybook story paths or UI navigation steps. State what the reviewer should expect to see. Do not write "run the test suite." When the testing steps have a meaningful sequence or involve multiple discrete actions, break them into a numbered list. Use a numbered list any time a reviewer would need to follow the steps in order to reproduce or verify the change.

**Screenshots & Recordings** — if the diff includes visual/UI changes, note that screenshots or `.mp4`/`.gif` clips should be added. Leave the Before/After subsections empty for the author to fill in.

### 5. Ask about collapsing unused sections

Before finalizing, identify which of the four lower sections are unused for this PR:

- **Classic vs Environments & Services Teams** — unused if the PR is purely frontend/UI with no Classic-specific behavior
- **Feature Flag(s) in Use** — unused if no LaunchDarkly flags are referenced in the diff
- **Observability Plan** — unused if no instrumentation, metrics, or query links are relevant
- **Docs & Rollout Plan** — unused if no docs changes and no staged rollout needed

Ask the user (using AskUserQuestion) with a question like:

> "The following sections appear unused for this PR: [list them]. Would you like to collapse them into a hidden `<details>` block, keep them visible with a note, or remove them entirely?"

Options:
1. **Collapse into `<details>`** — wraps all four in a `<details><summary>Feature flags, observability, and rollout — expand if applicable</summary>` block (recommended for most UI-only PRs)
2. **Keep visible** — leave them as-is (blank or with N/A)
3. **Remove entirely** — delete unused sections from the output

Apply the user's choice before writing the final output.

### 6. Write the final PR body

Output the complete PR body using this template structure:

```markdown
<!-- PR titles should follow conventional commits: feat(poodle):, fix:, maint:, instr:, etc. -->

**Linear:** <!-- paste issue URL, or "none" -->

## Summary

<!-- What does this PR change and why?
     What should reviewers focus on?
     Is there anything you're uncertain about? -->

## Open Questions

<!-- Optional: questions for reviewers about this dependency -->

## Rejected Alternatives

<!-- Optional: alternatives considered and why they were ruled out -->

## How to Test

<!-- Step-by-step test plan. Reference specific Storybook story names, UI paths,
     or commands. State what the reviewer should expect to see.
     Use a numbered list when steps have a meaningful sequence or involve
     multiple discrete actions a reviewer needs to follow in order. -->

## Screenshots & Recordings

<!-- Before/after screenshots or screen recordings if applicable.
     Video clips (.mp4, .gif) are encouraged for interaction-heavy changes. -->

### Before

### After

<details>
<summary>Feature flags, observability, and rollout — expand if applicable</summary>

## Classic vs Environments & Environments Teams

## Feature Flag(s) in Use

## Observability Plan

## Docs & Rollout Plan

</details>
```

Fill in the sections using the content gathered in steps 3–4:
- **Summary**: use the narrative from the brainstorm conversation, or your own analysis of the diff — covering what changed and why, what reviewers should focus on, and anything uncertain.
- **Linear**: extracted ticket ID or URL, or the placeholder if not found.
- **Open Questions**: include this section only if a net-new dependency was detected AND the user provided a non-empty answer. Remove it entirely otherwise.
- **Rejected Alternatives**: include this section only if a net-new dependency was detected AND the user provided a non-empty answer. Remove it entirely otherwise.
- **How to Test**: specific Storybook paths or UI steps with expected outcomes. Use a numbered list when the steps have a meaningful sequence or involve multiple discrete actions.
- Keep HTML comments only in sections the author needs to fill themselves (e.g. screenshots/recordings).

### 7. Apply to the PR

If a PR already exists, update its body:

```bash
gh pr edit --body "$(cat <<'BODY'
<final body here>
BODY
)"
```

If no PR exists yet, open a draft PR with `gh pr create --draft` using the final body.

### 8. After opening the PR

Report the PR URL and stop. NEVER start watching CI unprompted — the work may still be in progress. Then ask the user: "Would you like me to babysit CI?"

If the user opts in, babysit CI:
1. Poll `gh pr checks` until RWX and required checks finish.
2. On failure, use the `fetch-ci-logs` skill to get the failing task's output.
3. Fix in-scope failures (format errors, broken tests from your wiring changes, snapshot updates) and push; repeat until green or blocked by something outside your control.
4. Do not treat "CI is running" as done. Report blocked failures to the user with context on why you're stuck.
5. Report: PR URL, what failed (if anything), what you fixed, and current CI status.

## Key differences from the base template

| Base template | This template |
|---|---|
| Linear ID buried in HTML comment | `**Linear:**` visible field at the top |
| Single-sentence Summary prompt | Sub-question prompts; asks to brainstorm "why" live |
| `## Before/After screenshots` | `## Screenshots & Recordings` (invites video) |
| Sections 4–7 always visible | Collapsed in `<details>` by default (ask first) |
| No dependency awareness | Detects net-new deps; conditionally adds Open Questions + Rejected Alternatives sections |
