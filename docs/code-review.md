# GitHub Copilot Code Review

[← Copilot SDK](copilot-sdk.md) | [Part II Overview](part-2-primitives.md) | [Part III Reference →](part-3-reference.md)

*Updated: April 16, 2026 · Validated against GitHub Copilot docs as of April 16, 2026.*

---

## Overview

**Where it runs:** GitHub.com pull request reviews, VS Code (review selection), Visual Studio, JetBrains, Xcode, GitHub CLI, GitHub Mobile
**Best for:** Automated first-pass review of PRs against team conventions, security rules, and architectural standards

**Official docs:** [Using GitHub Copilot code review](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review) · [Code review custom instructions](https://docs.github.com/en/copilot/how-tos/use-code-review/code-review-instructions) · [Code review concepts](https://docs.github.com/en/copilot/concepts/code-review/code-review)

[GitHub Copilot code review](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review) is a cross-cutting feature, not a primitive. It consumes the same instruction files and memory that shape Copilot's behavior in chat and agent mode — and it surfaces that customization as pull request comments. Done well, it catches the boring review comments (convention drift, missing error handling, security smells) before a human reviewer sees the diff, and leaves humans to focus on design and intent.

This guide is **customization-focused**: how the [eight primitives](part-2-primitives.md) steer what Copilot flags in reviews. For enablement, billing, and organization-level policy, defer to the [official docs](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review).

---

## How Code Review Reads Your Customization

Copilot code review reads customization files the same way chat and agent mode do — with a few specifics worth knowing:

- **Base branch wins.** Copilot reviews a pull request using the instruction files on the **base branch**, not the feature branch. A contributor cannot change the review rules in their own PR.
- **Character budget.** `.github/copilot-instructions.md` contributes up to **4,000 characters** to code review context. Content beyond that is ignored. Long instructions should be split into path-scoped files under `.github/instructions/`.
- **Repository beats organization.** If an org defines default review instructions and a repository defines its own, the repository's rules take priority.
- **Toggle at the repository.** Custom instructions for code review can be disabled per-repository from the repo's Copilot settings without removing the files.
- **Review type is "Comment" only.** Copilot leaves suggestions as review comments. It does not approve, request changes, or block merges — and its reviews do not count toward required-approval thresholds.

See the [custom instructions reference](https://docs.github.com/en/copilot/how-tos/use-code-review/code-review-instructions) for the authoritative list of which files are read and the character budget per file.

---

## Primitive-by-Primitive Leverage

Not every primitive affects code review today. The table below is the quick summary; the sections that follow expand the four that do.

| Primitive | Affects code review? | How |
|-----------|----------------------|-----|
| [Always-on Instructions](primitive-1-always-on-instructions.md) | ✅ Yes | First 4,000 chars of `copilot-instructions.md` / `AGENTS.md` are included as review context |
| [File-based Instructions](primitive-2-file-based-instructions.md) | ✅ Yes | `applyTo` globs activate path-specific rules when reviewing matching files |
| [Custom Agents](primitive-5-custom-agents.md) | Indirect | Reviewer personas don't auto-run in PRs today, but the instruction files they reference do influence reviews |
| [Copilot Memory](primitive-8-memory.md) | ✅ Yes | Learned repository patterns carry into review comments |
| [Prompts](primitive-3-prompts.md) | ❌ No | Prompts are invoked by humans in chat — reviews are non-interactive |
| [Skills](primitive-4-skills.md) | ❌ No (today) | Skills are discovered during agent sessions, not PR reviews |
| [MCP](primitive-6-mcp.md) | ❌ No (today) | External tools are not invoked during code review |
| [Hooks](primitive-7-hooks.md) | ❌ No | Hooks govern local agent sessions, not cloud review |

### Always-on instructions

Use [`copilot-instructions.md`](primitive-1-always-on-instructions.md) (or `AGENTS.md`) for universal conventions every review should enforce — logging requirements, error handling patterns, forbidden APIs, naming rules. Keep it tight: you have 4,000 characters before content is truncated for review.

✅ **Good — specific and testable**
```markdown
## Error handling
- Every `async` function must wrap its body in try/catch and log errors via `logger.error` with a stable code prefix (e.g. `ERR_AUTH_001`).
- Never swallow errors silently. If an error is intentionally ignored, add a `// intentional:` comment explaining why.
```

❌ **Bad — vague and unreviewable**
```markdown
## Error handling
- Handle errors properly.
- Write clean code.
```

Copilot has no way to act on the second example in a review comment — there's no concrete rule to compare the diff against.

### File-based instructions

Path-scoped instructions under [`.github/instructions/*.instructions.md`](primitive-2-file-based-instructions.md) are the sharpest tool for code review. The `applyTo` glob pattern means rules only activate when the reviewed file matches — so your security rules don't fire on test fixtures, and your test-style rules don't fire on production code.

```markdown
---
applyTo: "src/auth/**/*.ts"
---

# Authentication review rules

- Never log request bodies, tokens, session IDs, or password fields.
- All session comparisons must use constant-time comparison (`crypto.timingSafeEqual`).
- Authorization checks must happen before any database mutation.
- New endpoints must document the permission they require in a JSDoc `@permission` tag.
```

A PR that touches `src/auth/session.ts` picks up these rules automatically; a PR that only changes `tests/**` does not. This is how you keep review noise low while still enforcing the strict rules where they matter.

### Custom agents (indirect)

Copilot code review does not run [custom agents](primitive-5-custom-agents.md) as part of a PR today — a `security-reviewer.agent.md` persona won't auto-assign itself. But the underlying instruction files that back that agent (always-on and path-scoped) are still read. The practical pattern: extract the *rules* from the reviewer persona into a path-scoped `.instructions.md` file so they flow to code review, and keep the *workflow* in the agent file for interactive chat use.

### Copilot Memory

[Copilot Memory (Preview)](primitive-8-memory.md) observes interactions across a repository and surfaces learned patterns as context in future sessions — including code review. A team that has corrected the same review comment across dozens of PRs will find Copilot starts flagging that pattern without it being written anywhere explicitly. Memory complements instruction files; it does not replace them. Explicit instructions are deterministic, auditable, and versioned with the repo. Memory is implicit and mutable. Use both.

---

## Surface Support

Copilot code review runs in multiple places, but the feature set differs by surface. The canonical matrix lives in the [official feature matrix](https://docs.github.com/en/copilot/reference/copilot-feature-matrix); the summary below is a sanity check as of April 2026.

| Surface | PR review (github.com) | In-IDE review of local changes |
|---------|------------------------|--------------------------------|
| [GitHub.com](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review) | ✅ Primary | — |
| [VS Code](surfaces/vscode.md) | ✅ (from PR UI) | ✅ "Review selection" — `github.copilot.chat.reviewSelection.enabled` |
| [Visual Studio](surfaces/visual-studio.md) | ✅ | ✅ |
| [JetBrains](surfaces/jetbrains.md) | ✅ | Preview |
| [Xcode](surfaces/xcode.md) | ✅ | Not available |
| [Eclipse](surfaces/eclipse.md) | ✅ (github.com) | ❌ Not on roadmap |
| [GitHub CLI](surfaces/copilot-cli.md) | ✅ (via `gh` PR commands) | — |

**VS Code editor review** uses a dedicated setting, [`github.copilot.chat.reviewSelection.enabled`](https://code.visualstudio.com/docs/copilot/reference/copilot-settings), and can be pointed at a custom instructions file via `github.copilot.chat.reviewSelection.instructions`. This is separate from PR review on github.com — but both read your repository's instruction files.

---

## 💬 Try This Prompt

> "Read the top files under `src/` and the existing `.github/copilot-instructions.md`. Draft a companion file at `.github/instructions/review-rules.instructions.md` with `applyTo: \"src/**/*.{ts,tsx}\"` that captures the non-obvious conventions a human reviewer would catch in this codebase — error handling, logging, auth, and data access patterns. Keep each rule specific enough that a code review tool could cite a line of diff against it. Do not duplicate rules already in `copilot-instructions.md`."

Run this once per directory tree with distinct conventions (backend, frontend, infra, tests). The output becomes the starting point for path-scoped review rules you can refine over the next few PRs.

---

## Patterns That Work

1. **One file, one applyTo scope.** Don't try to capture every rule in `copilot-instructions.md`. Split by directory or stack: `frontend.instructions.md`, `api.instructions.md`, `db.instructions.md`. Each stays focused and well under the character cap.
2. **Write rules as diff-citable statements.** Every rule should be specific enough that Copilot can point at a line of code and say "this violates it." If a rule requires interpretation, a human reviewer will need to apply it anyway.
3. **Prefer allow/forbid lists over philosophy.** "Use `pino` for logging; do not import `console.*` or `winston`" outperforms "Use structured logging."
4. **Mirror the review you wish you didn't have to give.** The best source of review rules is the review comments you keep re-typing. Grep your PR history for repeat patterns and codify them.
5. **Version the rules with the repo.** Instruction files live in `.github/` and ship with the code. When a rule changes, it changes in a PR — so the change itself goes through review.

---

## Limitations

- **4,000 character cap** on `copilot-instructions.md` for code review. Longer content is silently truncated. Split into path-scoped files under `.github/instructions/` when you hit the limit.
- **No merge blocking.** Copilot reviews are Comment-only. Enforcement of any rule still requires a human reviewer, branch protection, or a CI check.
- **Model is not switchable.** Unlike chat and agent mode, code review does not expose a model selector.
- **Re-review may repeat comments.** When you re-request review after changes, Copilot may repeat comments you previously resolved. This is a known behavior; see the [official docs](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review) for current status.
- **Skills, MCP, and hooks do not participate in code review today.** Plan customization for review around instructions and memory.

---

## Further Reading

- [Using GitHub Copilot code review](https://docs.github.com/en/copilot/using-github-copilot/code-review/using-copilot-code-review) — how to request, configure, and interpret reviews
- [Code review custom instructions](https://docs.github.com/en/copilot/how-tos/use-code-review/code-review-instructions) — which files are read, the character budget, and precedence rules
- [Code review concepts](https://docs.github.com/en/copilot/concepts/code-review/code-review) — architecture and behavior reference
- [Copilot Feature Matrix](https://docs.github.com/en/copilot/reference/copilot-feature-matrix) — per-surface support grid
- [VS Code Copilot settings reference](https://code.visualstudio.com/docs/copilot/reference/copilot-settings) — `reviewSelection` and related editor settings

---

[← Copilot SDK](copilot-sdk.md) | [Part II Overview](part-2-primitives.md) | [Part III Reference →](part-3-reference.md)
