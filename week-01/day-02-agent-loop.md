# Day 2 — The agent loop: runs, turns, events, streaming

Day 1 gave you the static map: gateway, channels, agent, sessions, tools. Today is the dynamic view. Between the moment a user hits send and the moment a reply appears in their chat, OpenClaw spins up a *run*, drives one or more *turns*, and emits a stream of *events* that get translated into channel messages. If you want to build your own agent on top of this stack, you have to be fluent in those three nouns.

## Mental model

Three words, three scopes:

- **Run** = the lifecycle. One inbound prompt produces one run. It's serialized per session key, has a `runId`, starts with `agent_start` and ends with `agent_end` (or a lifecycle error). `agent.wait` resolves on lifecycle end.
- **Turn** = one round-trip with the model. A run typically contains several turns: one to think + call a tool, one to read the tool result and reply, etc. Bracketed by `turn_start` / `turn_end`.
- **Event** = a single signal on the stream. `message_start`, `message_update`, `tool_execution_end`, `compaction_start`, and so on. Events are what `subscribeEmbeddedPiSession` listens to.

Here is one run with a tool call, in time order:

```
prompt arrives
  │
  ▼
[run]  agent_start ──────────────────────────────────────────────► agent_end
        │
        ├─ [turn 1]  turn_start ─► message_start ─► message_update* ─► message_end
        │             │                                                  │
        │             └─► tool_execution_start ─► tool_execution_update* ┘
        │                  └─► tool_execution_end ─► turn_end
        │
        └─ [turn 2]  turn_start ─► message_start ─► message_update* ─► message_end ─► turn_end
                                            │
                                            └─► onBlockReply / onPartialReply ─► channel send
```

Reasoning, compaction, and retries all slot into this same skeleton. Internalize the diagram and the rest of today is just labeling pieces of it.

## How it works internally

Open `pi.md`'s "Core integration flow" section side-by-side with `pi-embedded-subscribe.ts` and you'll see the loop is a thin event router on top of pi-agent-core's `AgentSession`.

**Lifecycle events.** `agent_start` marks the run beginning; `agent_end` marks completion. `subscribeEmbeddedPiSession` translates these into the OpenClaw `lifecycle` stream with `phase: "start" | "end" | "error"`. `agent.wait` blocks on lifecycle end/error for a given `runId`. Anything that needs to fire exactly once per run (final delivery, transcript flush, hook `agent_end`) hangs off lifecycle end.

**Turn events.** `turn_start` and `turn_end` bracket each model round-trip. A run with no tool calls is one turn. A run with a tool call is at least two: one to emit the tool call, one to consume the tool result and reply. This matters when you reason about cost and context — the system prompt is sent every turn, not every run.

**Message events.** `message_start` opens an assistant message. `message_update` carries text deltas (and reasoning deltas, if streaming reasoning). `message_end` closes it. These deltas are what `EmbeddedBlockChunker` consumes. The chunker buffers text until it has enough to emit a coarse "block" — at minimum `minChars`, preferably split on a paragraph/newline/sentence boundary, never inside a fenced code block. When `blockStreamingBreak` is `text_end`, blocks are flushed as the chunker fills; when it's `message_end`, the chunker waits and may still emit multiple chunks if the buffered text exceeds `maxChars`.

**Tool events.** `tool_execution_start`, `tool_execution_update`, `tool_execution_end` bracket each tool call. The subscriber calls `onToolResult` on `tool_execution_end` (after sanitizing for size and image payloads) and feeds tool-progress text into preview streaming when the channel supports it.

**Compaction events.** `compaction_start` and `compaction_end` flank automatic context compaction; the run can retry afterward, and OpenClaw resets in-memory buffers and tool summaries on retry to avoid duplicate output.

**Channel callbacks.** `subscribeEmbeddedPiSession` exposes a callback surface — `onBlockReply`, `onPartialReply`, `onToolResult`, `onReasoningStream`, `onAgentEvent`. Each chunk that the block chunker emits is run through `consumeReplyDirectives`, which parses and strips reply directives like `[[media:url]]`, `[[voice]]`, and `[[reply:id]]`, returning `{ text, mediaUrls, audioAsVoice, replyToId }`. The cleaned text is then run through `stripBlockTags`, which removes `<think>` / `<thinking>` content and, when `enforceFinalTag` is on, keeps only what's inside `<final>...</final>`. Only after that does `onBlockReply` fire with a payload the channel can actually deliver.

**Final assembly.** When the run ends, the final payload is built from assistant text (plus reasoning, when visible), inline tool summaries (when verbose allows), and assistant error text on errors. The exact silent token `NO_REPLY` / `no_reply` is filtered out, messaging-tool duplicates are removed, and a `chat: final` is emitted on lifecycle end/error.

## Knobs you control

The loop is fixed; how it surfaces is configurable. The dials that matter:

- **`thinking` level** — `low` / `medium` / `high` per agent or per run; falls back if a model rejects it (`pickFallbackThinkingLevel`). Controls reasoning effort, not visibility.
- **`/reasoning on|off|stream`** — controls reasoning *visibility*. `stream` writes reasoning deltas as block replies (or into the preview bubble on Telegram).
- **`/verbose` and `/think`** — chat commands that flip verbose level and thinking on a session without editing config.
- **Block streaming** — `agents.defaults.blockStreamingDefault` (`on`/`off`), `blockStreamingBreak` (`text_end` or `message_end`), `blockStreamingChunk` (`minChars`/`maxChars`/`breakPreference`), `blockStreamingCoalesce` (`idleMs` to merge tiny blocks). Non-Telegram channels also need `*.blockStreaming: true`.
- **`humanDelay`** — `off` / `natural` / `custom` randomized pause between block replies.
- **Preview streaming** — `channels.<ch>.streaming` with modes `off` / `partial` / `block` / `progress`, plus `streaming.preview.toolProgress` to show or hide "searching the web" status lines.
- **Typing** — `agents.defaults.typingMode` (`never` / `instant` / `thinking` / `message`) and `typingIntervalSeconds`.
- **Reply directives in the system prompt** — telling the agent it can emit `[[media:url]]`, `[[voice]]`, `[[reply:id]]`, or `NO_REPLY` is what makes those directives actually fire.
- **Message tool action** — channel-specific message tools let the agent send out-of-band; `EmbeddedMessagingSentTracker` suppresses duplicate assistant confirmations for those sends.

## Power-user pattern

In a noisy group chat, you usually don't want the bot to shout one big reply that has to mention three different people. Combine two source-grounded mechanisms:

1. **Per-message threading with `[[reply:id]]`.** Have the agent emit `[[reply:<sourceMessageId>]]` at the top of each block targeted at a specific user. `consumeReplyDirectives` extracts it and the channel delivers as a quote-reply to that exact message. Combined with `blockStreamingBreak: "text_end"`, you get one quoted bubble per addressee instead of one long broadcast.
2. **Silent replies for internal-only turns.** When the agent only needs to call a tool (e.g. log a memory, label a thread) without saying anything, have it return the literal token `NO_REPLY`. Groups/channels allow silence by default; OpenClaw filters the token from the outgoing payload but still delivers any pending tool media. In direct chats, `silentReplyRewrite` will convert it to a short visible fallback — leave that on unless you explicitly want a silent DM.

The combo lets one run produce N targeted quote-replies plus zero broadcast noise, all driven by what the model writes — no channel-specific glue code.

## Try this

Pick one inbound message you ran today and reconstruct its event timeline.

1. Find the session JSONL under `~/.openclaw/agents/<agentId>/sessions/` (or `$OPENCLAW_STATE_DIR/...`).
2. Open it and identify: where does the new run start? How many `turn_start` / `turn_end` pairs are inside it? How many `tool_execution_*` triples?
3. Re-run a prompt with verbose logging on (`/verbose on` in the chat, or the `--verbose` flag on the CLI). In the gateway log, find the `lifecycle`, `assistant`, and `tool` stream lines. Mark which `message_update` deltas got coalesced into a single `onBlockReply` and which got split.
4. Bonus: turn on `/reasoning stream` and watch reasoning deltas appear *before* `message_start`. That's `thinking` typing-mode territory.

Ten minutes, one terminal. You will never read the rest of this course's code the same way.

## Reflection

1. Why does the system prompt get sent on every `turn_start`, not just once per run? What does that imply for cost when an agent makes five tool calls?
2. If `blockStreamingBreak` is `message_end`, can `onBlockReply` still fire more than once for a single assistant message? Under what condition?
3. You see a duplicate media attachment in a Telegram reply. Which of these is the more likely culprit — `EmbeddedBlockChunker`, `consumeReplyDirectives`, or the final-payload media-dedupe in the streaming pipeline — and why?

---
[← Day 1](day-01-five-piece-map.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 3 →](day-03-pi-engine.md)
