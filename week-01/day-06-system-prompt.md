# Day 6 — The system prompt is dynamic

> "If AGENTS.md is the soul, the system prompt is the *body* — assembled fresh every turn from a dozen organs."

If you spent yesterday inside a session and wondered why the model "knew" your timezone, your sandbox status, and your skill list without you telling it — today is for you. The system prompt is not a single file on disk. It is a structured document built per-run by `buildAgentSystemPrompt()` in `system-prompt.ts`, and once you can enumerate its inputs you can finally stop guessing where behavior is coming from.

## Mental model

The system prompt is **assembled, not stored**. There is no `system-prompt.txt` you can edit. Each model run, OpenClaw walks a builder that pulls fragments from the runtime, the workspace, the channel, the active skill set, and any per-run override, then concatenates them in a fixed order before handing the result to Pi.

```
                workspace files          channel context
              (AGENTS.md, SOUL.md,     (Slack/WhatsApp/CLI,
               TOOLS.md, USER.md,         direct vs group,
               IDENTITY.md, MEMORY.md)    silent replies)
                       \                   /
   sandbox info ────►   buildAgentSystemPrompt()  ◄──── runtime metadata
   (mode, paths,                  │                     (host, OS, model,
   workspaceAccess)                │                      thinking level,
                       /          ▼          \           timezone)
              skills list      promptMode    runtime override
              (eligibility,    (full | minimal | none)   (createSystemPromptOverride
              allowlist,                                  → applySystemPromptOverrideToSession)
              budget)                  │
                                       ▼
                        ┌──────────────────────────┐
                        │   final system prompt    │
                        │   (sent to model run)    │
                        └──────────────────────────┘
```

The same agent talking on Slack and on the CLI sees different prompts. The same workspace running a main session vs a spawned subagent sees different prompts. That is by design.

## How it works internally

`buildAgentSystemPrompt()` (in `system-prompt.ts`, called per run, applied via `applySystemPromptOverrideToSession()`) walks a known list of sections. Per `pi.md`'s "System prompt construction" section, the assembled prompt includes:

- **Tooling** — structured-tool source-of-truth + runtime guidance (cron vs exec, `sessions_spawn` vs polling, `update_plan` rules when enabled).
- **Tool Call Style** — provider-overlayable. GPT-5 family, for example, gets persona/concision/parallel-lookup tuning here.
- **Safety guardrails** — short advisory text. Real enforcement lives in tool policy and exec approvals.
- **OpenClaw CLI reference** — `config.schema.lookup`, `config.patch`, `config.apply`, `update.run`, gateway-tool refusals.
- **Skills** — an `<available_skills>` block produced by `formatSkillsForPrompt`. Only injected when eligible skills exist after metadata gates, env checks, and `agents.defaults.skills` / `agents.list[].skills` allowlist.
- **Docs** — local docs path (Git checkout or npm package), source location, ClawHub, Discord, fallback to `https://docs.openclaw.ai`.
- **Workspace** — `agents.defaults.workspace`.
- **Sandbox** — only when sandboxing is enabled. Includes mode, sandbox paths, and whether elevated exec is available.
- **Messaging** — outbound message tool guidance, native-approval-card hint when the channel supports it.
- **Reply Tags** — optional, only for providers that support them.
- **Voice** — voice/transcription guidance when voice is on.
- **Silent Replies** — `NO_REPLY` token semantics. Omitted on auto-reply runs when channel context already encodes the resolved silent-reply behavior.
- **Heartbeats** — heartbeat ack rules when enabled for the default agent.
- **Runtime metadata** — host, OS, node, model, repo root, thinking level, plus a `Current Date & Time` section keyed off `agents.defaults.userTimezone`.
- **Memory + Reactions** — only when those subsystems are enabled.
- **Optional context files / extra system prompt content** — operator-supplied.

Below those, the **bootstrap files** are appended under a "Project Context" header: `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `IDENTITY.md`, `USER.md`, `HEARTBEAT.md`, `BOOTSTRAP.md` (first run only), and `MEMORY.md` if present. Each is trimmed against `agents.defaults.bootstrapMaxChars` (default 12000), and the total against `agents.defaults.bootstrapTotalMaxChars` (default 60000). Truncations can emit a warning block via `agents.defaults.bootstrapPromptTruncationWarning`.

**Minimal mode for subagents.** The runtime sets `promptMode` per run (not user-facing). For sub-agents it is `minimal`: Skills, Memory Recall, OpenClaw Self-Update, Model Aliases, User Identity, Reply Tags, Messaging, Silent Replies, and Heartbeats are all stripped. Tooling, Safety, Workspace, Sandbox, Current Date & Time, Runtime, and injected context survive. Bootstrap injection is also pruned: subagents only get `AGENTS.md` and `TOOLS.md`. And when `promptMode=minimal`, extra injected prompt blocks are labeled "Subagent Context" instead of "Group Chat Context". `promptMode=none` returns only the base identity line.

**The override pattern.** For per-run additions you do not put into a workspace file, OpenClaw exposes:

```ts
const systemPromptOverride = createSystemPromptOverride(appendPrompt);
applySystemPromptOverrideToSession(session, systemPromptOverride);
```

`createSystemPromptOverride(appendPrompt)` builds the override object; `applySystemPromptOverrideToSession()` attaches it to a session after creation so the next run picks it up. This is how webhooks and triggered runs inject one-shot context without polluting `AGENTS.md`.

## Knobs you control

Stable, workspace-level:

- **`AGENTS.md`** — operating instructions, loaded every session as bootstrap.
- **`SOUL.md`, `IDENTITY.md`, `USER.md`** — persona, name/vibe/emoji, who the user is.
- **`TOOLS.md`** — local tool conventions (guidance, not policy).
- **`HEARTBEAT.md`** — heartbeat-only checklist; gated by heartbeat config and `agents.defaults.heartbeat.includeSystemPromptSection`.
- **`MEMORY.md`** — curated long-term memory. Watch its size; it counts against context every turn.

Per-agent and per-channel:

- **Agent-level system prompt additions** via `agents.list[]` — extra prompt content applied only to that agent.
- **Channel-specific system prompts** — additions tied to a route, so Slack-vs-WhatsApp tone differs without touching the global prompt.
- **`agents.defaults.skills` / `agents.list[].skills`** — skill allowlist, which decides what the Skills section shows.
- **Skills budget** — `skills.limits.maxSkillsPromptChars` globally, `agents.list[].skillsLimits.maxSkillsPromptChars` per agent.
- **Bootstrap caps** — `agents.defaults.bootstrapMaxChars`, `agents.defaults.bootstrapTotalMaxChars`, `agents.defaults.bootstrapPromptTruncationWarning`.

Per-run / runtime:

- **`createSystemPromptOverride(appendPrompt)` + `applySystemPromptOverrideToSession(session, ...)`** — programmatic per-run additions.
- **Memory toggle, Reactions toggle, Voice toggle** — each gates its own section.
- **Silent replies / Reply tags** — feature toggles whose presence in the prompt depends on channel state.
- **Time/timezone** — `agents.defaults.userTimezone`, `agents.defaults.timeFormat`.
- **"Extra system prompt content"** — operator-supplied free text appended at the end of the assembled prompt.

The rule of thumb: if the content is stable, put it in a bootstrap file. If it varies by surface, put it in agent or channel config. If it varies by *request*, use the override.

## Power-user pattern

**Layer your prompt by lifetime.** Use `AGENTS.md` for stable personality and operating rules — anything that should be true on every turn, on every channel, forever. Use channel-level system prompt additions for surface-specific tone: terse + emoji-light on Slack, chattier with reply tags on WhatsApp, structured headers on the CLI. Reach for run-time overrides only when *programmatic* context is needed — for example, a webhook firing with a specific payload that the model needs to see exactly once for that single run.

```ts
// inside the webhook handler, after session creation
const session = await createSession(agentId, channelCtx);
const append = `Webhook payload (one-shot):\n${JSON.stringify(payload, null, 2)}`;
applySystemPromptOverrideToSession(session, createSystemPromptOverride(append));
await runTurn(session, userMessage);
```

This keeps `AGENTS.md` clean (it does not grow whenever you add a new integration), keeps channel configs surface-shaped (you can read them and immediately know "ah, this is the Slack voice"), and isolates the noisy, ephemeral, payload-shaped context to the run that actually needs it. When something feels off, you check the layers in order: AGENTS.md → channel config → override. Three places, each with a clear job.

## Try this

~15 minutes.

1. Pick a workspace and run a normal turn on the CLI: `openclaw` then any prompt.
2. Run `/context list` to get the headline numbers, then `/context detail` for the per-section breakdown. Note the `System prompt (run)` line — that is the actually-built prompt for the last embedded run, not an estimate.
3. From the detail output, identify each section listed in this doc: Tooling, Safety, Skills, Workspace, Sandbox, Runtime, Project Context, etc. For each, decide whether it came from runtime, config, or a workspace file.
4. Now spawn a subagent (e.g. via `sessions_spawn`) and run `/context list` against *its* session. Diff the two. The minimal-mode pruning should be obvious: Skills, Memory, Messaging, Heartbeats gone; bootstrap reduced to `AGENTS.md` and `TOOLS.md`.
5. If you have `system-prompt-report.ts` (or equivalent debug output) wired up, dump the assembled text for both runs and confirm the section ordering matches `pi.md`'s "System prompt construction" list.

Goal: by the end you should be able to point at any line in the assembled prompt and name the source.

## Reflection

1. Which of your bootstrap files (AGENTS, SOUL, TOOLS, USER, IDENTITY, HEARTBEAT, MEMORY) is closest to its truncation cap, and what would you cut first?
2. If you need a behavior to apply only on Slack DMs but not Slack groups, do you express that in `AGENTS.md`, channel config, or a runtime override — and why?
3. A subagent is misbehaving in a way the parent agent never does. Given minimal-mode pruning, which removed sections are the most likely culprits, and how would you confirm?

---
[← Day 5](day-05-sessions.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 7 →](day-07-end-to-end-trace.md)
