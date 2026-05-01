# Day 3 — Pi, the engine inside

## Mental model

Pi is the engine. OpenClaw is the chassis.

OpenClaw does not implement an LLM agent loop, a model router, or coding tools. It imports those from Pi — an SDK by Mario Zechner (`@badlogic`) published as the `pi-mono` monorepo. When a Telegram message turns into a `read` tool call and a streamed reply, almost every interesting verb in that sentence is Pi's verb. OpenClaw's job is to decide *when* to call Pi, *with what prompt*, *under whose identity*, and *to which channel* the streamed bytes go back.

Stack it in layers, bottom to top:

```
+----------------------------------------------+
| OpenClaw                                     |
|   gateway, channels, sessions-per-key,       |
|   auth profiles, sandbox, system prompt      |
+----------------------------------------------+
| pi-coding-agent  (the SDK surface)           |
|   createAgentSession, SessionManager,        |
|   AuthStorage, ModelRegistry, builtin tools  |
+----------------------------------------------+
| pi-agent-core    (the loop)                  |
|   AgentMessage, tool execution, turns        |
+----------------------------------------------+
| pi-ai            (the model abstraction)     |
|   Model, streamSimple, provider APIs         |
+----------------------------------------------+
```

A fourth package, `pi-tui`, is parallel rather than under the others — it's the terminal UI Pi ships with, and OpenClaw also pulls it in for its own `pnpm tui` mode. Everything you observed yesterday in Day 2's agent loop — turn boundaries, tool execution events, compaction — is happening *inside Pi*. OpenClaw subscribes to those events and adapts them to channels.

## How it works internally

### The four packages

From `docs/pi.md`:

| Package           | Purpose                                                                                                |
| ----------------- | ------------------------------------------------------------------------------------------------------ |
| `pi-ai`           | Core LLM abstractions: `Model`, `streamSimple`, message types, provider APIs                           |
| `pi-agent-core`   | Agent loop, tool execution, `AgentMessage` types                                                       |
| `pi-coding-agent` | High-level SDK: `createAgentSession`, `SessionManager`, `AuthStorage`, `ModelRegistry`, built-in tools |
| `pi-tui`          | Terminal UI components (used in OpenClaw's local TUI mode)                                             |

OpenClaw pins all four to the same version (currently `0.70.2`). They're versioned together because the inner types — `AgentMessage`, `ToolDefinition`, `AgentTool` — leak across the boundary, and a mismatch will cost you a typecheck.

### Embedded, not subprocess

The most consequential decision OpenClaw made about Pi: it does not spawn the `pi` CLI as a child process and it does not use Pi's RPC mode. It imports Pi as a library and calls `createAgentSession()` directly. The pi.md "Overview" lists the benefits this buys, and they're all things you cannot get over a subprocess boundary:

- Full control over session lifecycle and event handling
- Custom tool injection (messaging, sandbox, channel-specific actions)
- System prompt customization per channel/context
- Session persistence with branching/compaction support
- Multi-account auth profile rotation with failover
- Provider-agnostic model switching

A subprocess gives you stdout. Embedding gives you typed events, in-process tool functions that close over the channel they should reply to, and a `SessionManager` you can wrap with safety guards. Day 2's agent loop only looks tractable because of this choice.

### The actual session call

Inside `runEmbeddedAttempt()` (under `src/agents/pi-embedded-runner/`), the call shape is:

```ts
const { session } = await createAgentSession({
  cwd: resolvedWorkspace,
  agentDir,
  authStorage: params.authStorage,
  modelRegistry: params.modelRegistry,
  model: params.model,
  thinkingLevel: mapThinkingLevel(params.thinkLevel),
  tools: builtInTools,
  customTools: allCustomTools,
  sessionManager,
  settingsManager,
  resourceLoader,
});

applySystemPromptOverrideToSession(session, systemPromptOverride);
```

Note `builtInTools` is empty in OpenClaw — `splitSdkTools()` returns `{ builtInTools: [], customTools: ... }`. OpenClaw replaces every tool, including `bash` (becomes `exec`/`process`) and the read/edit/write trio (sandbox-aware variants). The last line is also load-bearing: instead of letting Pi build its own system prompt from `AGENTS.md`, OpenClaw computes one in `buildAgentSystemPrompt()` and pushes it onto the session.

### Where state lives

| Aspect          | Pi CLI                  | OpenClaw                                                                                       |
| --------------- | ----------------------- | ---------------------------------------------------------------------------------------------- |
| Sessions        | `~/.pi/agent/sessions/` | `~/.openclaw/agents/<agentId>/sessions/` (or `$OPENCLAW_STATE_DIR/...`)                        |
| Config          | `AGENTS.md` + prompts   | `openclaw.json` + dynamic per-channel prompt                                                   |
| Auth            | Single credential       | `auth-profiles.json` with rotation + cooldown                                                  |
| Event handling  | TUI rendering           | Callbacks (`onBlockReply`, `onToolResult`, ...)                                                |

`~/.openclaw` *shadows* `~/.pi` for everything OpenClaw touches. If you've used Pi standalone before, your `~/.pi` is untouched and still works for `pi` directly.

## Knobs you control

You don't write Pi code from OpenClaw config — but OpenClaw exposes a small set of keys that map straight onto Pi parameters. Knowing the mapping is the difference between guessing and steering.

**Thinking level.** OpenClaw's `--thinking low|medium|high` (and the per-message override) maps via `mapThinkingLevel()` to Pi's `thinkingLevel` argument on `createAgentSession`. If a model rejects the level, `pickFallbackThinkingLevel()` downgrades automatically.

**Model selection.** `provider` and `model` flow into `resolveModel()`, which uses Pi's `ModelRegistry` and `AuthStorage`. OpenClaw's multi-profile rotation sits *above* `AuthStorage` — it picks a profile, then calls `authStorage.setRuntimeApiKey(...)` so Pi sees a single credential per attempt.

**System prompt override.** OpenClaw assembles a prompt with sections for Tooling, Safety, Skills, Workspace, Sandbox, Messaging, Voice, Reply Tags and more, then injects it via `applySystemPromptOverrideToSession()`. You influence this through channel config, skills, sandbox mode, and the `extraSystemPrompt` knob — not by editing Pi.

**Settings overrides.** `src/agents/pi-settings.ts` is the seam where OpenClaw applies its own values to Pi's `SettingsManager` before the session is built. Compaction mode (`safeguard`) and context pruning (`cache-ttl`) are wired in through Pi *extensions* registered via `resourceLoader` — set them in `cfg.agents.defaults` and they're loaded as Pi extension paths, not custom OpenClaw code paths.

The mental rule: anything labelled "thinking", "model", "compaction", "context window", or "system prompt" in OpenClaw config ends up as a Pi argument. Anything labelled "channel", "auth profile", "sandbox", "skills" or "messaging" is an OpenClaw concept that wraps Pi.

## Power-user pattern

**Locate the layer before you debug.**

When the agent does something weird, the first triage question is: is this an OpenClaw layer issue or a Pi layer issue? They have completely different fixes.

OpenClaw layer symptoms: the wrong channel got the reply; the system prompt didn't include a skill; the wrong auth profile was used; a tool call was filtered out by policy; an image wasn't injected for this turn; a `[[media:...]]` directive wasn't parsed. Fix in `src/agents/` files outside the `pi-embedded-runner/` core — `channel-tools.ts`, `system-prompt.ts`, `auth-profiles.ts`, `pi-tools.policy.ts`.

Pi layer symptoms: the model refused; the streamed text included raw `<thinking>` tags; the tool schema was rejected by Gemini; compaction kicked in unexpectedly; a turn ordering error came back from Anthropic. Fix is usually a Pi version bump, a setting in `pi-settings.ts`, or a hook in `src/agents/pi-hooks/`.

The fastest disambiguator: run the same prompt under `pnpm tui` (Pi-native TUI on the same session files) and under `pnpm openclaw agent`. If the bug reproduces in TUI, it's Pi. If only OpenClaw shows it, it's the chassis.

## Try this

1. From an OpenClaw checkout, find Pi's installed package metadata:

   ```bash
   cat node_modules/@mariozechner/pi-coding-agent/package.json | head -40
   ```

   Confirm the version matches the `0.70.2` pin in `docs/pi.md`. Look at the `exports` field — every named export listed there (`createAgentSession`, `SessionManager`, `AuthStorage`, `ModelRegistry`, `DefaultResourceLoader`, `SettingsManager`) is something OpenClaw imports.

2. List the `~/.openclaw` paths that shadow `~/.pi`. Start at `~/.openclaw/agents/<agentId>/`. Compare:

   - `agents/<agentId>/sessions/` ↔ `~/.pi/agent/sessions/`
   - `agents/<agentId>/agent/auth-profiles.json` ↔ Pi's single-credential storage
   - `openclaw.json` (top-level) ↔ `AGENTS.md`

   If `OPENCLAW_STATE_DIR` is set, swap that for `~/.openclaw`. Notice what does *not* exist on the OpenClaw side: there's no equivalent of Pi's standalone CLI history, because OpenClaw sessions are keyed per channel.

## Reflection

1. Why does OpenClaw set `builtInTools: []` and pass everything through `customTools`? What invariant would break if it accepted Pi's defaults?
2. If Pi released a `0.71.0` with a breaking change to `AgentTool.execute`, which OpenClaw file would break first, and why?
3. You hit a bug where the model answers fine in `pnpm tui` but produces empty replies through Telegram. Which layer do you investigate, and which two or three files do you open first?

---
[← Day 2](day-02-agent-loop.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 4 →](day-04-harnesses.md)
