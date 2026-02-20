# Copilot Memory (Preview)

[← Hooks](part-2-7-hooks.md) | [Part II Overview](part-2-primitives.md)

*Published: February 20, 2026 · Validated against VS Code 1.109 and GitHub Copilot docs as of this date.*

---

## Overview

The first seven customization primitives are all *authored* — someone on the team writes instructions, prompts, skills, agents, MCP configs, or hooks. Copilot Memory is different. It's *learned*. Copilot discovers information about a codebase through its own activity and stores it for future use.

Think of the difference this way: customization files are the employee handbook you write for every new hire. Memory is the tribal knowledge that same employee picks up after months of working on the project — the undocumented patterns, the quirks of the codebase, the decisions that never made it into a README.

**Status:** Public Preview — subject to change
**Scope:** Repository-scoped (memories stored for a repo are available to all users with Copilot Memory access in that repo)
**Used by:** Copilot coding agent, Copilot code review, and Copilot CLI

**Official docs:**
- [About agentic memory](https://docs.github.com/en/copilot/concepts/agents/copilot-memory)
- [Enabling and curating Copilot Memory](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/copilot-memory)
- [VS Code 1.109 release notes](https://code.visualstudio.com/updates/v1_109#_copilot-memory-preview)

---

## How Memory Works

Copilot Memory operates through two complementary interfaces:

### Agentic Memory (Automatic)

As Copilot works on a repository — through coding agent sessions, code review, or CLI interactions — it automatically identifies and stores "memories." These are tightly scoped pieces of information about the codebase: how the project handles database connections, which patterns the team uses for error handling, what settings need to stay synchronized across files.

Memories created by one surface carry over to others. If the coding agent discovers how a repository handles authentication, code review can later apply that knowledge when reviewing a PR that touches the same area. If code review learns that two configuration files must stay synchronized, the coding agent will know to update both when it modifies one.

### VS Code Memory Tool (Explicit)

In VS Code, the memory tool lets developers explicitly tell the agent to remember something. Telling the agent "always ask clarifying questions before refactoring" or "this project uses named exports — never use default exports" saves that as a memory. In future sessions, the agent recalls that context automatically.

**Important:** Because Memory is repository-scoped, anything saved through the memory tool is shared with all users who have Copilot Memory enabled in that repo. Frame memories as codebase conventions, not personal preferences.

Enable the VS Code memory tool:

```json
"github.copilot.chat.copilotMemory.enabled": true
```

---

## Memory Architecture

| Aspect | Details |
|--------|---------|
| **Storage scope** | Repository-specific — memories for repo A are never used in repo B |
| **Visibility** | All users with Copilot Memory access in a repository share the same memory pool |
| **Citations** | Each memory is stored with references to specific code locations that support it |
| **Validation** | Before using a memory, Copilot checks its citations against the current codebase to confirm the information is still accurate and relevant to the current branch |
| **Expiration** | Memories auto-delete after 28 days to prevent stale information from affecting decisions |
| **Renewal** | If a memory is validated and used, a new memory with the same details may be stored, extending its longevity |
| **Access control** | Only created from activity by users with write permission who have Copilot Memory enabled |

### Memory Lifecycle

```text
Copilot works on the codebase
     ↓
Discovers a pattern, convention, or constraint
     ↓
Stores memory with citations to specific code locations
     ↓
Future Copilot session begins (any user with Memory enabled)
     ↓
Copilot retrieves potentially relevant memories
     ↓
Validates citations against current codebase
     ↓
Uses memory only if validation passes
     ↓
After 28 days without renewal → auto-deleted
```

---

## Enabling Copilot Memory

Memory requires enablement at two levels: the GitHub platform (where memories are stored) and the client (VS Code).

### GitHub Platform

Copilot Memory is off by default. It must be enabled in enterprise, organization, or personal settings depending on how the Copilot subscription is managed.

**Enterprise:** Enterprise owners define a policy under **AI controls > Copilot > Features > Copilot Memory**. Options: *Let organizations decide*, *Enabled everywhere*, or *Disabled everywhere*.

**Organization:** Organization owners enable it under **Settings > Copilot > Policies > Features > Copilot Memory**. If the org belongs to an enterprise, the enterprise policy may override this.

**Individual (Pro/Pro+):** Enable in personal Copilot settings on GitHub under **Features > Copilot Memory**.

If a user receives Copilot from multiple organizations, the most restrictive setting applies — Memory is only active if all organizations have it enabled.

### VS Code

Enable the memory tool in VS Code settings:

```json
"github.copilot.chat.copilotMemory.enabled": true
```

This gives the agent access to the memory tool, allowing it to store and retrieve memories during chat sessions.

### Availability by Plan

| Plan | Copilot Memory Available |
|------|--------------------------|
| Copilot Enterprise | ✅ (enterprise/org admin enables) |
| Copilot Business | ✅ (enterprise/org admin enables) |
| Copilot Pro+ | ✅ (user enables in personal settings) |
| Copilot Pro | ✅ (user enables in personal settings) |
| Copilot Free | ❌ |

---

## Good vs. Bad Memory Candidates

Not everything belongs in memory. The best memory candidates are things Copilot discovers organically while working on the codebase.

| | Example | Why |
|-|---------|-----|
| ✅ | "This project uses a custom ORM wrapper — always use `db.query()` instead of raw SQL" | Specific, discoverable convention that prevents repeated corrections |
| ✅ | "The `config/sync.yml` and `config/deploy.yml` files must stay synchronized" | Cross-file constraint that Copilot can learn and enforce |
| ✅ | "This project uses Vitest with `describe`/`it` blocks for all tests" | Testable convention that reduces repeated corrections |
| ✅ | "The API base path is `/api/v2`" | Factual context that reduces repeated corrections |
| ❌ | Storing an entire style guide in memory | Too large — use always-on instructions or file-based instructions instead |
| ❌ | "Write good code" | Too vague to influence behavior |
| ❌ | Project-wide architecture decisions the whole team needs to follow deterministically | Team standards belong in customization files, not probabilistic memory |

---

## Memory vs. Customization Primitives

Memory is complementary to repository customization — not a replacement for any primitive. The key distinction: customization files are **deterministic** (loaded every time, applied consistently), while memories are **probabilistic** (validated, may expire, may not surface in every session).

| Layer | Scope | Audience | Lives In | Deterministic? |
|-------|-------|----------|----------|----------------|
| Customization files (instructions, skills, agents, etc.) | Repository | Everyone on the team | `.github/` | Yes — always loaded when conditions match |
| Copilot Memory | Repository | All users with Memory enabled | GitHub's cloud storage | No — validated, may expire, may not surface |

### When to Use Which

| Situation | Use Customization Files | Use Memory |
|-----------|------------------------|------------|
| Team coding standards everyone must follow | ✅ Always-on instructions | ❌ Not reliable enough for enforcement |
| File-type-specific patterns | ✅ File-based instructions | ❌ |
| Codebase conventions discovered through usage | ❌ Would require manual authoring | ✅ Learned and shared across the repo |
| Cross-file dependencies | Possible via instructions, but hard to maintain | ✅ Copilot learns these organically |
| Security-critical rules that must never be violated | ✅ Instructions + Hooks for enforcement | ❌ Not suitable for enforcement |

The rule of thumb: if a convention must be followed consistently by every team member in every session, encode it in customization files. If it's something Copilot can learn and validate from the codebase itself, let Memory handle it.

---

## Managing Memories

Repository owners can review and manage stored memories through the GitHub web UI.

### Viewing Memories

1. Navigate to the repository on GitHub
2. Go to **Settings > Code & automation > Copilot > Memory**
3. Memories are displayed in chronological order, most recent first

### Deleting Memories

Delete individual memories using the trash icon, or select multiple using checkboxes and click **Delete**. Deletion is useful for memories that are incorrect, misleading, or no longer relevant.

Copilot validates memories before using them — if the code that generated a memory no longer exists, the memory won't be applied. But explicit deletion is cleaner than waiting for validation to filter it out.

### Auto-Expiration

Memories auto-delete after **28 days**. If a memory is validated and used during that window, Copilot may re-store it with updated details, effectively renewing its lifespan. Actively useful memories persist; stale ones naturally decay.

---

## Practical Considerations

### Memory and Pull Requests

Memories can be created from code in pull requests, including PRs that are closed without merging. The validation mechanism ensures these memories won't affect Copilot's behavior if there's no substantiating evidence in the current codebase — so a rejected approach in a closed PR won't pollute future work.

### Cross-Surface Knowledge Transfer

This is one of Memory's most powerful properties. Memories created by one Copilot surface are available to all others:

- **Coding agent** discovers a pattern → **Code review** enforces it on future PRs
- **Code review** identifies a constraint → **Coding agent** respects it during implementation
- **CLI** learns project build conventions → **Coding agent** uses them in automated tasks

### Memory Does Not Replace Instructions

A common misconception: "If Memory learns everything, do I still need customization files?" Yes. Memory is probabilistic — it may not surface in every session, it expires, and it can't be audited or version-controlled the way `.github/` files can. Use Memory as a supplement that fills gaps between what's documented and what's actually practiced.

---

## What's Coming

The official documentation notes that Copilot Memory will be extended to "other parts of Copilot, and for personal and organizational scopes, in future releases." Currently, Memory is repository-scoped and used by the coding agent, code review, and CLI. Expect the scope to broaden as the feature matures.

---

## Further Reading

- [About agentic memory for GitHub Copilot](https://docs.github.com/en/copilot/concepts/agents/copilot-memory) — Conceptual overview
- [Enabling and curating Copilot Memory](https://docs.github.com/en/copilot/how-tos/use-copilot-agents/copilot-memory) — Setup and management
- [VS Code 1.109 release notes: Copilot Memory](https://code.visualstudio.com/updates/v1_109#_copilot-memory-preview) — VS Code memory tool details

---

[← Hooks](part-2-7-hooks.md) | [Next: Part III - Reference →](part-3-reference.md)
