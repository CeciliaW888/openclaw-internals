# Glossary

Plain-English definitions for every architectural term used in this course. Sorted by where you'll first hit them.

---

## Core surface

**Gateway** — The single always-running OpenClaw process on your machine. It owns channels, sessions, tools, auth, and the WebSocket that everything else connects to. Think "the kernel" of your assistant.

**Agent** — A configured AI assistant living in a workspace folder. One Gateway can run multiple agents, each with its own personality, model, skills, and channel routing.

**Workspace** — The folder where an agent's files live (`~/.openclaw/agents/<agentId>/` by default). Holds session transcripts, agent-specific config, custom skills, bootstrap files.

**Session** — One conversation thread, persisted as a JSONL file with a tree structure (each message has an `id` and a `parentId`, so you can branch).

**Channel** — A messaging surface OpenClaw can talk through (WhatsApp, Telegram, Slack, Discord, iMessage, WebChat, etc.). Channels are configured per agent.

**Node** — A device (phone, tablet, another Mac) that pairs to your Gateway over WebSocket and can contribute capabilities (camera, mic, screen, location).

**Control client** — The interface you use to drive the Gateway: the macOS menu-bar app, the CLI (`openclaw …`), or the web dashboard.

---

## The agent loop

**Run** — One full execution lifecycle in response to a prompt. Starts with `agent_start`, ends with `agent_end`. May contain multiple turns.

**Turn** — One round-trip with the model inside a run: send context → receive response → maybe execute tools → maybe send again. Marked by `turn_start` / `turn_end` events.

**Event** — A structured message the agent emits during a run (`message_start`, `message_update`, `tool_execution_start`, `tool_execution_end`, `compaction_start`, `compaction_end`, etc.). OpenClaw subscribes to these and turns them into channel replies.

**Block reply** — A discrete chunk of text the agent sends back to a channel, separated from other blocks (e.g., by paragraph or directive). The `EmbeddedBlockChunker` decides where the splits go.

**Reply directive** — Inline markers like `[[media:url]]`, `[[voice]]`, `[[reply:id]]` that the agent emits to control how a block is delivered (attach media, send as voice, reply-to a specific message).

**Streaming** — Receiving the model's output token-by-token (or chunk-by-chunk) instead of waiting for the full response. Lets the channel show "typing" and start delivering blocks before the run is complete.

---

## Pi

**Pi** — The agent SDK by Mario Zechner (`@mariozechner` / `badlogic`). Four packages: `pi-ai`, `pi-agent-core`, `pi-coding-agent`, `pi-tui`. OpenClaw embeds Pi rather than running it as a subprocess.

**`pi-ai`** — Core LLM abstractions: provider APIs, message types, `streamSimple`, `Model` types.

**`pi-agent-core`** — The agent loop and tool execution machinery. Defines `AgentMessage`, `AgentTool`.

**`pi-coding-agent`** — High-level SDK: `createAgentSession`, `SessionManager`, `AuthStorage`, `ModelRegistry`, the default coding tools (read, write, edit, bash).

**`pi-tui`** — Terminal UI components used by Pi's native CLI and OpenClaw's local TUI mode.

**`createAgentSession()`** — The Pi function OpenClaw calls to instantiate a fresh agent session in-process. The thing that makes "embedded" possible.

**`AgentSession`** — The Pi object that owns one conversation: receives `prompt()`, emits events.

---

## Harnesses & runtimes

**Harness** — The execution engine for embedded agent turns. The "loop" that runs one turn against a model. OpenClaw ships two: the default Pi harness, and the Codex harness (for Codex/GPT-5.5-style models that have their own native runtime).

**Agent runtime** — The setting (`agentRuntime.id`) that picks which harness runs an agent's turns: `"pi"`, `"codex"`, or `"auto"`.

**Runtime pinning** — Once a session has run on one harness, it stays on that harness. To switch, you `/new` or `/reset`.

**`openai-codex/*` vs `openai/*`** — `openai-codex/...` is a *model ref* that tells Pi to use Codex auth/route. `agentRuntime.id: "codex"` is a *harness choice* that tells OpenClaw to run the turn through the Codex loop. They're orthogonal.

---

## Sessions & memory

**JSONL session file** — The transcript file: one JSON object per line, each with `id`, `parentId`, `role`, `content`, etc. The `parentId` link makes it a tree, not a list — branching is "what if we replied differently from this point?"

**SessionManager** — Pi's class for reading/writing the JSONL file. Cached per file path so we don't re-parse on every turn.

**Compaction** — Summarizing older history to free up context room. Triggers automatically when the model says "context too large" or manually via `/compact`.

**Compaction safeguard** — An OpenClaw Pi extension that adds adaptive token budgeting and tool-failure summaries to compaction so important context isn't lost.

**Context pruning (cache-TTL)** — A Pi extension that drops old prunable tool results based on cache TTL, keeping context fresh without compaction.

**History limiting** — Trimming how many past turns are sent to the model, with different limits for DM vs group channels.

---

## System prompt

**System prompt** — The instructions the model sees before any user message. Built dynamically per-channel, not static.

**`buildAgentSystemPrompt()`** — The function in `src/agents/system-prompt.ts` that assembles the system prompt from sections (Tooling, Safety, Skills, Workspace, Sandbox, Memory, Reactions, Voice, Heartbeats, Runtime metadata, etc.).

**Minimal prompt mode** — A trimmed system prompt used by subagents, where most sections are stripped to keep the context tight.

**System prompt override** — A way to inject extra content at the end of the system prompt for a specific run, via `applySystemPromptOverrideToSession()`.

**Bootstrap files** — Files (like `AGENTS.md`, `TOOLS.md`) injected into the system prompt at session start, defining the agent's personality and operating rules.

---

## Tools

**Tool** — A function the agent can call (`read`, `bash`, `browser`, `canvas`, `cron`, `message`, `sessions_send`…). The agent picks tools by name during a turn.

**Built-in tools** — Pi's default coding tools: `read`, `bash`, `edit`, `write`. OpenClaw replaces some with custom versions (`exec`/`process`).

**Custom tools** — OpenClaw-specific tools: messaging, browser, canvas, sessions, cron, gateway, channel actions.

**`AgentTool` vs `ToolDefinition`** — Two slightly different tool interfaces in `pi-agent-core` and `pi-coding-agent`. The adapter in `pi-tool-definition-adapter.ts` bridges them.

**Tool policy** — Rules that filter which tools an agent can use, by profile, provider, agent, group, or sandbox.

**Schema normalization** — Cleaning tool parameter schemas to handle Gemini/OpenAI quirks before passing them to the model.

---

## Skills

**Skill** — A bundled or workspace-defined capability the agent can invoke when its description matches the user's intent. Lives as a folder with a `SKILL.md` (frontmatter + instructions).

**Skill snapshot** — The list of available skills + their descriptions, injected into the system prompt so the model knows what's there.

**ClawHub** — The community skill registry at [clawhub.ai](https://clawhub.ai).

**Bundled / managed / workspace skills** — Three scopes: bundled ships with OpenClaw, managed is auto-installed by config, workspace is yours.

---

## Auth, models, failover

**Auth profile** — One credential entry (API key or OAuth token) for one provider. You can have multiple per provider.

**Profile rotation** — Cycling through profiles when one hits a rate limit or fails.

**`FailoverError`** — A Pi exception that triggers fallback to a different model when raised, given a fallback is configured.

**Model resolution** — The process of picking which model to use for a run: explicit override → agent default → global default.

---

## Sandbox & security

**Sandbox** — A constrained execution environment for tools. Backends: Docker (default), SSH, OpenShell.

**`agents.defaults.sandbox.mode`** — Config key for when to sandbox: `"main"` (always), `"non-main"` (only for non-`main` sessions, i.e. when others are talking to your agent), `"never"`.

**`main` session** — The session that's "you" talking to your own agent. Trusted by default.

**DM policy** — How the agent handles direct messages from unknown senders: `"pairing"` (default — code required) or `"open"` (anyone can talk).

**Allowlist** — The list of sender IDs allowed to talk to the agent on a given channel.

---

## Hooks & extensions

**Pi extension** — A plugin loaded into a Pi session to modify its behavior (e.g., compaction safeguard, context pruning).

**Hook** — A specific extension point inside Pi where OpenClaw can inject logic.

**ResourceLoader** — Pi's mechanism for finding extensions, skills, and other resources in the agent directory and standard paths.

---

## Triggers & automation

**Cron job** — A scheduled trigger that fires a prompt at a recurring time (`0 9 * * *` etc.).

**Webhook** — An HTTP endpoint OpenClaw exposes that fires a prompt when called.

**Gmail Pub/Sub** — A specific webhook setup that fires when matching email arrives.

**Voice Wake** — Wake-word detection on macOS/iOS that opens a voice session.

**Talk Mode** — Continuous voice mode (Android default).

**Multi-agent routing** — Mapping inbound channels/accounts/peers to different agents (e.g., your work Slack → "ops" agent, your personal Telegram → "buddy" agent).

---

## Plugins (broader system)

**Plugin** — A bundled extension that adds channels, tools, harnesses, or providers to OpenClaw. Codex harness is shipped as a plugin.

**ClawHub plugin** — A community plugin you install via config.

**Bundle** — A group of plugins shipped together.
