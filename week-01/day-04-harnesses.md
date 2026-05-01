# Day 4 — Harnesses and agent runtimes

## Mental model

You were confused by the Codex harness page, and the confusion is reasonable: there are three things named Codex, and the docs use "harness," "runtime," and "model ref" almost interchangeably. Here is the crisp version.

A **harness** is the *execution engine* for one prepared agent turn. The SDK page calls it "the low level executor for one prepared OpenClaw agent turn. It is not a model provider, not a channel, and not a tool registry." OpenClaw resolves the provider, model, tools, prompt, and session first, then hands that prepared bundle to a harness which actually drives the model loop.

An **agent runtime** is the user-facing config knob (`agentRuntime.id`) that picks which harness runs the turn. The runtimes doc puts it bluntly: "A harness is the implementation that provides an agent runtime. For example, the bundled Codex harness implements the `codex` runtime."

A **runtime pin** is the rule that "session runtime pins are sticky" — once a session records which harness it used, OpenClaw refuses to silently swap to a different one mid-conversation.

Now the load-bearing distinction. **A model ref and a runtime answer different questions.** Quoting the harness doc directly:

> `openai-codex/*` answers "which provider/auth route should PI use?"
> `agentRuntime.id: "codex"` answers "which loop should execute this embedded turn?"

These two things look related but they answer different questions. `openai/gpt-5.5` is the model ref, `codex` is the runtime. You can mix and match, and most of your config-debugging time will go into not confusing them.

## How it works internally

OpenClaw ships **two embedded harnesses** out of the box:

- **PI** (`agentRuntime.id: "pi"`) — the built-in default. The PI runner owns the model HTTP/WebSocket call, native tool dispatch, transcript writes, compaction, and the rest of the embedded loop. It is what runs unless something explicitly says otherwise.
- **Codex** (`agentRuntime.id: "codex"`, provided by the bundled `codex` plugin) — runs the embedded turn through the Codex app-server. Codex owns "model discovery, native thread resume, native compaction, and app-server execution."

(There are also CLI backends like `claude-cli` and `google-gemini-cli`, but those are not embedded harnesses — they shell out to a local CLI process while keeping the model ref canonical. Don't pass them to harness selection logic.)

**Embedded execution** means the harness runs *inside* OpenClaw's prepared agent loop. PI's flavor of embedded execution is "OpenClaw owns everything end to end." Codex's flavor is "Codex owns the model loop and native thread, OpenClaw projects context in and mirrors transcripts out." The Codex harness page states the boundary explicitly:

> OpenClaw still owns chat channels, session files, model selection, tools, approvals, media delivery, and the visible transcript mirror.

That sentence is the most important one on the whole page. **Switching the harness does not switch ownership of the surrounding state.** Telegram is still Telegram. Your `models.json` still picks the model. Your tools still appear in OpenClaw's tool list. Codex just runs the inner loop.

When the Codex harness is running, OpenClaw observes native lifecycle events as app-server notifications. The `codex_app_server.hook` events project Codex `hook/started` and `hook/completed` for trajectory and debugging. The harness also surfaces `turn/completed` as the natural finalization signal and routes `turn/steer` so the `/codex steer` chat command can inject mid-turn guidance into a running Codex thread without restarting it. These are adapter-level observations, not byte-for-byte captures.

### Model ref vs runtime — the table you need

| Question                                  | Model ref           | Runtime (`agentRuntime.id`) |
| ----------------------------------------- | ------------------- | --------------------------- |
| What does it answer?                      | "Which provider/auth route should PI use?" | "Which loop should execute this embedded turn?" |
| Example value                             | `openai/gpt-5.5`, `openai-codex/gpt-5.5`, `anthropic/claude-opus-4-6` | `pi`, `codex`, `auto`, `claude-cli` |
| Where it lives in config                  | `agents.defaults.model` (or per-agent) | `agents.defaults.agentRuntime.id` |
| Owns auth / provider catalog              | Yes                 | No                          |
| Owns the inner model loop                 | No                  | Yes                         |
| Sticky on a session?                      | No (you can `/model` switch any turn) | Yes (recorded as a runtime pin) |
| `openai-codex/gpt-5.5` + `runtime: pi`    | ChatGPT/Codex OAuth through OpenClaw's PI runner. Common. |
| `openai/gpt-5.5` + `runtime: codex`       | OpenAI model ref selected, Codex app-server runs the loop. Also common. |
| `openai-codex/gpt-5.5` + `runtime: codex` | Doctor warns. Pick one or the other. |

The two columns are orthogonal. Picking `openai-codex/*` does *not* enable the Codex harness; picking `agentRuntime.id: "codex"` does *not* change auth.

## Knobs you control

**`agents.defaults.agentRuntime.id`** (and the per-agent override on `agents.list[].agentRuntime.id`) is the master switch. Valid values:

- `pi` — force the built-in harness.
- `codex` — force the Codex app-server harness. Fails closed by default; set `fallback: "pi"` in the same scope if you want PI compatibility on miss.
- `auto` — let registered plugin harnesses claim provider/model pairs they understand. PI is the compatibility fallback when nothing claims it. Note: in `auto` mode, the Codex plugin deliberately does *not* claim `openai-codex/*`, "to avoid silently moving subscription-auth configs onto the native app-server harness."
- A CLI backend alias (`claude-cli`, `google-gemini-cli`, `codex-cli`) — runs a local CLI process.

**`OPENCLAW_AGENT_RUNTIME=<id>`** forces a runtime for new or reset sessions from the environment. Useful for live tests.

**`messages.visibleReplies`** controls how the visible transcript mirror surfaces assistant turns to users — independent of which harness produced them. Switching harnesses does not change visibility behavior.

**`/new`** starts a fresh OpenClaw session, which lets the harness selector run again from current config. **`/reset`** clears the OpenClaw session binding for the current thread so the next turn re-resolves. Both are required after changing `agentRuntime` config on an active conversation, because "session runtime pins are sticky" — config changes do not hot-switch an existing transcript.

**Model ref selection** stays in `models.json` / `agents.defaults.model`, and `/model` switches it live. A model switch does *not* change the runtime pin; only `/new` or `/reset` does.

## Power-user pattern

**Two agents, two runtimes, one config.** Don't force one harness globally if you sometimes want the other — a forced runtime applies to every turn, and selecting an Anthropic model under a forced Codex runtime makes the turn fail closed. Instead, put each runtime on a dedicated agent:

```json5
{
  agents: {
    defaults: { agentRuntime: { id: "auto", fallback: "pi" } },
    list: [
      { id: "main", default: true, model: "anthropic/claude-opus-4-6" },
      {
        id: "codex",
        name: "Codex",
        model: "openai/gpt-5.5",
        agentRuntime: { id: "codex", fallback: "none" },
      },
    ],
  },
}
```

Now your default `main` agent runs through PI with normal mixed-provider freedom, and `codex` is a Codex-only specialist for code-heavy work where you want native Codex behavior — its own queue, native interrupts via `/codex steer`, native compaction. Route channels or `/agent` switches to whichever you want. **Do not mix harnesses in one session**: the runtime pin will refuse to flip mid-conversation, and you'd be replaying one transcript through two incompatible native session systems anyway. One agent, one runtime, the whole way through.

## Try this

1. Open your OpenClaw config and find `agents.defaults.agentRuntime` (or add it). Set `id: "pi"` explicitly.
2. Start a fresh session with `/new`, send a prompt, and run `/status`. You should see `Runtime: OpenClaw Pi Default`.
3. Without ending the session, edit the config to `id: "codex"` (and enable the `codex` plugin if needed). Send another prompt. `/status` will still show PI — the runtime pin is sticky.
4. Run `/new`. Send a prompt. Now `/status` should show `Runtime: OpenAI Codex`.
5. Look at the session files on disk. Note that the OpenClaw session/transcript shape is the *same* in both cases — only the recorded harness id and the Codex sidecar binding differ. That's the "OpenClaw still owns session files" point made tangible.

If you don't have the Codex plugin set up, do steps 1–2 with `pi`, then try `id: "auto"` and observe that `/status` still resolves to PI because nothing else claimed the run.

## Reflection

1. A teammate sets `model: "openai-codex/gpt-5.5"` and `agentRuntime.id: "codex"` and asks why `openclaw doctor` warns. What does each of those config values mean independently, and which one should change depending on the intent?
2. You change `agentRuntime.id` from `pi` to `codex` in config. Why does an existing session keep running on PI, and what is the minimal command to opt that conversation into Codex?
3. In `auto` mode, why does the Codex plugin deliberately *not* claim `openai-codex/*` model refs? What would silently break if it did?

---
[← Day 3](day-03-pi-engine.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 5 →](day-05-sessions.md)
