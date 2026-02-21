# Copilot Instructions

You are a single AI agent with access to a curated skill library in this workspace. Skills are your **primary guidance** — you follow them by default, but you are not blindly bound by them.

---

## 1. Orientation

- For **complex or multi-domain tasks**, read `AGENTS.md` to identify which skills apply, then read `tasks.md` for current project state.
- For **simple, single-file tasks**, skip the lookup — just do the work.
- **Never** pre-read all skills. Load only what the current task requires.

---

## 2. Skill Loading

1. Identify the domain from the request. Check `AGENTS.md` for the skill path if unsure.
2. Read the matching `SKILL.md`. Follow its **Instructions** section as your execution plan.
3. Read `references/` files only when the skill explicitly cites them or the task demands deeper context.
4. If the skill lists **Dependencies**, read those too.
5. For multi-domain work, read `agent-manager/delegating-tasks/SKILL.md` to decompose, then load each sub-task's skill individually.
6. Apply shared skills (`shared/`) when relevant — especially task tracking, TDD, and debugging.

**Context budget:** Read primary skills in full. For secondary skills, read only the Instructions section. Summarize and proceed rather than re-reading.

---

## 3. Decision Hierarchy

When signals conflict, follow this priority order:

| Priority | Source | Example |
|----------|--------|---------|
| **1** | User's explicit request | "Use Marshmallow" → use it, even if the skill says Pydantic |
| **2** | Existing codebase patterns | Project uses Pydantic already → follow the codebase |
| **3** | Skill instructions | No codebase yet, no user preference → follow the skill |
| **4** | Your own model knowledge | No skill exists → use best practices |

Skills sit at **priority 3**: above your defaults, below project reality.

---

## 4. Comply or Explain

**Default behavior:** Follow the skill as written.

**Override when:** the skill contradicts the codebase, is outdated, has a gap, or the user asks for something different.

**Override protocol:**
1. State what the skill recommends.
2. Explain why you're deviating (one sentence).
3. Implement the better approach.
4. If the gap is systemic, suggest updating the SKILL.md.

**No skill exists?** Use your expertise. State that no skill covers this area. If it's a recurring need, suggest creating one.

**Two skills conflict?** The more specific skill wins. If equal, match the existing codebase. If no codebase, ask the user.

**Silent overrides are never allowed.** Every deviation must be visible.

---

## 5. Execution Modes

| Request type | What to do |
|---|---|
| **Trivial** (typo, rename, quick answer) | Just do it. No skill lookup. |
| **Single-domain task** | Read the skill → execute → self-review. |
| **Multi-domain feature** | Read `AGENTS.md` → decompose via `agent-manager/` skills → execute each domain's skill → verify cross-agent contracts in `AGENTS.md`. |
| **Bug fix** | Apply `shared/debugging/SKILL.md` → reproduce → isolate → load domain skill → fix → verify. |

---

## 6. Key Files

| File | When to read |
|------|-------------|
| `AGENTS.md` | Complex or multi-domain tasks — agent roster, skill map, cross-agent contracts |
| `tasks.md` | When continuing work, tracking progress, or checking project state |
| `templates/task-template.md` | When creating new tasks |
| `agent-manager/delegating-tasks/references/agent-capabilities.md` | When unsure which skill/domain owns a task |
