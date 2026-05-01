# Day 9 — Skills as a runtime contract

## Mental model

Skills are how OpenClaw extends the agent's behavior at runtime — not by editing the system prompt, not by writing a tool, but by dropping a folder on disk and letting the model decide when to load it.

A skill is a directory containing a `SKILL.md` file. The file has YAML frontmatter (`name`, `description`) and a Markdown body of instructions. The frontmatter is metadata; the body is procedural guidance the agent will read on demand.

What makes skills feel magic is the discovery path. OpenClaw scans known skill roots at startup, builds a **skill snapshot** (just names + descriptions + paths), and injects that snapshot — not the bodies — into the system prompt under an `<available_skills>` block. The model sees a small catalog. When user intent looks like it matches a description, the model calls `read` on the skill's `SKILL.md` and follows what's inside.

```
on-disk skill folders
        │
        ▼
ResourceLoader scan ──► skill snapshot (name + description + path)
                              │
                              ▼
                  injected into system prompt
                              │
                              ▼
            model sees catalog, matches user intent
                              │
                              ▼
              model invokes read on SKILL.md path
                              │
                              ▼
                 instructions enter context, agent acts
```

The crucial consequence: the model picks skills **by their description**. Not by file name, not by directory location, not by how good the body is. The description is the only signal the model has at decision time.

## How it works internally

**The SKILL.md file.** Every skill folder must contain a `SKILL.md`. The frontmatter is short:

```markdown
---
name: gitcrawl
description: Use gitcrawl for OpenClaw issue and PR archive search, duplicate discovery, related-thread clustering, and local GitHub mirror freshness checks.
metadata:
  openclaw:
    requires:
      bins:
        - gitcrawl
---

# Gitcrawl

Use this skill before live GitHub search...
```

`name` is the canonical identifier (lowercased, `[a-z0-9_-]`, max 80 chars — `SkillWorkshop` normalizes names this way when it writes new skills). `description` is the runtime contract. `metadata.openclaw.requires` is an optional eligibility gate — environment, binaries, config flags. Skills that fail their gate are filtered out of the snapshot before injection.

**Snapshot building.** `src/agents/skills.ts` (referenced in `docs/pi.md`) is the subsystem that owns this. On agent startup, OpenClaw constructs a `DefaultResourceLoader` (from `pi-coding-agent`) pointed at the active workspace and agent dir, then `reload()` scans every skill root. Each eligible skill becomes one entry. The renderer (`formatSkillsForPrompt`) emits the snapshot:

```xml
<available_skills>
  <skill>
    <name>gitcrawl</name>
    <description>Use gitcrawl for OpenClaw issue and PR archive search...</description>
    <location>/Users/.../skills/gitcrawl/SKILL.md</location>
  </skill>
  ...
</available_skills>
```

The total snapshot is bounded by `skills.limits.maxSkillsPromptChars` (per-agent override: `agents.list[].skillsLimits.maxSkillsPromptChars`). When no skills are eligible, the entire **Skills** section is omitted from the system prompt.

**Scope hierarchy.** OpenClaw resolves skill roots with explicit precedence (`docs/concepts/agent.md`):

1. Workspace: `<workspace>/skills` — yours, project-local
2. Project agent skills: `<workspace>/.agents/skills`
3. Personal agent skills: `~/.agents/skills`
4. Managed/local: `~/.openclaw/skills` (auto-installed via ClawHub)
5. Bundled — shipped with the install, or contributed by enabled plugins
6. Extra: `skills.load.extraDirs`

Plugin-bundled skills are eligible **only when their owning plugin is enabled** — that's how channel plugins like `wecom` ship "document/meeting/messaging skills" without bloating every install.

**When a skill "fires."** This is the line most people get wrong. Skills are not tools. A tool is invoked by name with a typed arguments payload, validated by schema, executed by the runtime. A skill has no schema and no execute path. The model "fires" a skill by reading the `SKILL.md` path with the standard `read` tool and then *behaving as instructed*. There is no skill router on the OpenClaw side beyond the snapshot injection — the model is the router. Once the body is in context, the agent's subsequent tool calls and prose follow that guidance.

**Hot refresh.** Skill writes (including those by `skill_workshop`) refresh the in-memory snapshot atomically — no Gateway restart needed. Add a SKILL.md, the next turn sees it.

## Knobs you control

**Where to put a skill.** For a project-specific skill, drop it under `<workspace>/skills/<skill-name>/SKILL.md`. For a personal skill that follows you across workspaces, use `~/.agents/skills/<skill-name>/SKILL.md`. For a skill that should ship with your plugin, bundle it inside the plugin's `skills/` subtree and gate it on the plugin being enabled.

**SKILL.md frontmatter.** Required: `name`, `description`. Optional: `metadata.openclaw.requires` to gate on binaries, env vars, or config flags so the skill stays out of the snapshot when its preconditions aren't met. Keep `description` to one or two sentences — the snapshot is char-budget-bounded, and verbose descriptions crowd out other skills.

**ClawHub installs.** ClawHub is the discovery surface for community skills and plugin-bundled skills:

```bash
openclaw plugins install <package-name>
```

OpenClaw checks ClawHub first, falls back to npm. Installed skills land under `~/.openclaw/skills` (managed scope) and refresh into the snapshot automatically.

**Enabling/disabling.** Skill eligibility honors several gates: metadata `requires`, runtime env/config checks, and the effective agent allowlist. Set `agents.defaults.skills` (or `agents.list[].skills`) to an explicit list to scope which skills any given agent can see. Combined with multi-agent routing, this lets you keep a "code-review" agent's snapshot tight while a generalist agent sees everything.

**Snapshot budget.** Tune `skills.limits.maxSkillsPromptChars` if you have many skills and the snapshot is getting trimmed. This is separate from `agents.defaults.contextLimits.*`, which sizes runtime tool reads.

## Power-user pattern

**Write skill descriptions as runtime contracts.** The model sees only the description in the snapshot. The body is invisible until something in the description makes the model decide to read it. So the description is doing all the routing work — it's not a label, it's a discriminator.

Bad description:

```yaml
description: Helps with code review.
```

This loses on every axis. "Helps" is mush. There's no trigger language, no scope, no anti-trigger. The model will either fire it on every turn that mentions code (noise) or never (dead skill).

Good description:

```yaml
description: Use this skill when the user asks for a code review, PR feedback, or wants bugs/regressions found in a diff. Triggers on "review", "PR", "diff", "feedback", "look over". Do not use for fresh code authoring, refactor planning, or design-doc work.
```

The good version names: (1) the **task class** (review, PR feedback, bug-finding); (2) the **input shape** (a diff); (3) **trigger phrases** the user actually says; (4) an **anti-trigger** — when *not* to use it. That last one matters most. Without anti-triggers, well-written skills bleed into adjacent intents and degrade the agent's judgment for everything nearby.

A useful test: read your description, cover the rest of the file, and ask "would I, as the model, know exactly when to fire this and when to stay out?" If the answer needs the body, the description is broken. Rewrite the description, not the instructions.

## Try this

Write a minimal personal skill that summarizes the past week from your sessions. Save it at `~/.agents/skills/weekly-review/SKILL.md`:

```markdown
---
name: weekly-review
description: Use this skill when the user asks for a weekly recap, "what did I work on", "summarize my week", or a Friday/Monday review of past sessions. Triggers on "weekly review", "recap", "what did I do this week". Do not use for single-session summaries or live status checks.
---

# Weekly Review

- List sessions from the last 7 days using `sessions_list`.
- For each, pull the first user prompt and final assistant turn.
- Group by theme (project, channel, or topic).
- Output a 5-bullet summary plus one open thread to follow up.
- Keep it under 200 words unless the user asks for detail.
```

Now verify the snapshot picked it up. Start a session and ask the agent to print its `<available_skills>` block, or use `openclaw status` / `openclaw skills` if your build exposes it. You should see `weekly-review` listed with its description and absolute path. Then ask, "what did I work on this week?" — the model should `read` the SKILL.md before answering.

## Reflection

1. The system prompt only includes skill *descriptions* and *paths*, not bodies. What does that imply about how aggressively you should optimize description wording vs. instruction wording?
2. A skill silently stops firing after you renamed it. What's the most likely cause — the snapshot, the description, the eligibility gate, or the model? How would you isolate it?
3. You have a "code-review" skill and a separate "design-review" skill, and the model keeps firing the wrong one on architectural diff requests. Without changing either body, how do you fix it?

---
[← Day 8](day-08-tools.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 10 →](day-10-auth-failover.md)
