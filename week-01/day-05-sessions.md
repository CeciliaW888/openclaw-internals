# Day 5 — Sessions and memory

You've had it happen. A long-running agent humming along, then suddenly a `🧹 Auto-compaction complete` line scrolls past and the next reply feels... lossy. The agent forgot the exact filename you were debugging. It re-read a file it had already read three turns ago. Today is about understanding what actually happened on disk and in memory when that compaction fired — and the related machinery (history limiting, context-pruning, memory flush) that runs alongside it.

## Mental model

A session is **not a flat list of messages**. It is a tree of JSON entries, one per line, persisted as JSONL, with each entry carrying an `id` and a `parentId` linking back to its predecessor. The pi-coding-agent docs and OpenClaw's `pi.md` are explicit: "Sessions are JSONL files with tree structure (id/parentId linking)." The `SessionManager` walks that tree to reconstruct the active conversation thread.

Most of the time the tree is degenerate — a straight line — because each new message just appends with `parentId` pointing at the previous tip. But the structure is real: branching, retries, forks, and compaction all create siblings. When compaction "rewrites" history, it does not actually rewrite the file in place. It writes a successor transcript and keeps the old JSONL as an archived checkpoint source.

```
turn-1 (user)
  └── turn-2 (assistant)
        └── turn-3 (tool_call)
              └── turn-4 (tool_result)
                    ├── turn-5a (assistant, original)   ← old branch
                    └── turn-5b (compaction-summary)    ← successor head
                          └── turn-6 (assistant, post-compact)
```

The active thread is whichever leaf `SessionManager` resolves as current. Everything else is still on disk; nothing is destroyed. Once you have this picture, the rest of today's machinery — history limiting, auto-compaction, pruning, memory flush — stops feeling like magic and starts feeling like a set of well-defined transformations on a tree.

## How it works internally

**File format.** Each session lives at `~/.openclaw/agents/<agentId>/sessions/<sessionId>.jsonl` (or under `$OPENCLAW_STATE_DIR/agents/<agentId>/sessions/` if you've set that). The store index is `~/.openclaw/agents/<agentId>/sessions/sessions.json`, which holds `sessionStartedAt`, `lastInteractionAt`, and `updatedAt` per row. Daily-reset freshness uses `sessionStartedAt`; idle-reset freshness uses `lastInteractionAt`. Bookkeeping writes (heartbeat, cron, exec system events) update `updatedAt` but do **not** extend either reset clock.

Each line of the JSONL is one entry — user message, assistant message, tool call, tool result, compaction summary, session header. Entries carry `id` and `parentId`. To replay the conversation, you walk from any leaf back up the parent chain.

**SessionManager.open().** OpenClaw embeds pi via `createAgentSession()`, and pi's `SessionManager.open(sessionFile)` is what parses the JSONL and exposes the tree. Because parsing is non-trivial for long sessions, `session-manager-cache.ts` keeps a cache of `SessionManager` instances keyed by file path:

```typescript
await prewarmSessionFile(params.sessionFile);
sessionManager = SessionManager.open(params.sessionFile);
trackSessionManagerAccess(params.sessionFile);
```

OpenClaw further wraps it with `guardSessionManager()` for tool-result safety.

**History limiting (DM vs group).** Before each model turn, `limitHistoryTurns()` in `pi-embedded-runner/history.ts` trims the assembled history. Per `pi.md`: "trims conversation history based on channel type (DM vs group)." DMs are intentionally allowed deeper history because continuity matters; group channels are clipped harder because they're noisier and isolation is per-group anyway.

**Auto-compaction triggers.** Two paths fire compaction. The proactive one: when the session approaches the model's context limit. The reactive one: when the model returns a context-overflow error and OpenClaw compacts and retries. The recognized overflow signatures (verbatim from `compaction.md`) are:

- `request_too_large`
- `context length exceeded`
- `input exceeds the maximum number of tokens`
- `input token count exceeds the maximum number of input tokens`
- `input is too long for the model`
- `ollama error: context length exceeded`

If `agents.defaults.compaction.maxActiveTranscriptBytes` is set, OpenClaw additionally triggers normal local compaction when the active JSONL grows past that size — useful when provider-side context management keeps the model healthy but the on-disk transcript still bloats.

**Manual compaction.** `compactEmbeddedPiSessionDirect()` (`pi-embedded-runner/compact.ts`) handles `/compact`. With `keepRecentTokens` set, manual compaction honors that cut-point and keeps the recent tail; without it, manual compaction is a hard checkpoint and the new summary is the new head.

**Compaction-safeguard extension.** `src/agents/pi-hooks/compaction-safeguard.ts` is a Pi extension (not vanilla pi behavior — OpenClaw injects it). It adds **adaptive token budgeting plus tool-failure and file-operation summaries** to compaction. Enabled when `agents.defaults.compactionMode === "safeguard"`:

```typescript
if (resolveCompactionMode(params.cfg) === "safeguard") {
  setCompactionSafeguardRuntime(params.sessionManager, { maxHistoryShare });
  paths.push(resolvePiExtensionPath("compaction-safeguard"));
}
```

**Context-pruning extension.** `src/agents/pi-hooks/context-pruning.ts` is a separate Pi extension, gated on `agents.defaults.contextPruning.mode === "cache-ttl"`. Pruning is **in-memory only** — it does not modify the JSONL. Per `session-pruning.md`: it waits for the cache TTL to expire (default 5m), finds old tool results, soft-trims oversized ones (head + `...` + tail), hard-clears the rest with a placeholder, then resets the TTL so follow-ups reuse the fresh cache. Compaction summarizes; pruning trims tool output. They complement each other.

## Knobs you control

**Compaction mode.** `agents.defaults.compactionMode: "safeguard"` enables the OpenClaw safeguard extension on top of pi's built-in compaction. Setting a custom `provider` under `agents.defaults.compaction` automatically forces `mode: "safeguard"`.

**Compaction model.** `agents.defaults.compaction.model` delegates summarization to a different model — for example a local `ollama/llama3.1:8b` dedicated to summarization, or a stronger model like `openrouter/anthropic/claude-sonnet-4-6`. `compaction.memoryFlush.model` separately controls the silent memory-flush turn that runs *before* compaction.

**Successor transcripts.** `agents.defaults.compaction.truncateAfterCompaction: true` makes compaction write a new active successor file rather than rewriting in place. The byte guard (`maxActiveTranscriptBytes`) requires this.

**Context pruning.** `agents.defaults.contextPruning: { mode: "cache-ttl", ttl: "5m" }`. Auto-enabled for Anthropic profiles. Set `mode: "off"` to disable.

**Chat commands.** `/compact [instructions]` forces compaction with optional summary guidance. `/new` starts a fresh session (no compaction, no summary carry-over). `/reset` is the same shape; `/new <model>` also switches the model.

**History/lifecycle.** `session.dmScope` (`main`, `per-peer`, `per-channel-peer`, `per-account-channel-peer`) controls DM isolation. `session.reset.idleMinutes` enables idle reset. `session.maintenance.mode: "enforce"` plus `pruneAfter` and `maxEntries` bounds the on-disk session store.

**Storage.** Default `~/.openclaw/agents/<agentId>/sessions/`. Override with `$OPENCLAW_STATE_DIR`.

## Power-user pattern

**Compact proactively, before the model forces it.** Auto-compaction fires either when you near the limit or — worse — when the provider returns `request_too_large` and OpenClaw has to compact-and-retry mid-turn. In the second case you don't get to influence the summary. The safeguard extension helps, but the summary boundary is whatever the runtime picked.

If you have a long session and you're about to pivot topics — say you finished the API design phase and you're moving to implementation — type `/compact Focus on the API design decisions and the final endpoint shapes` *before* you ask the next implementation question. You control three things: when the boundary lands, what gets emphasized in the summary, and which model writes the summary (via `compaction.model`). Pair this with `compaction.keepRecentTokens` so the immediately recent turns stay verbatim and only older history is summarized.

The cheap version of the same pattern: `/new` when you're starting truly unrelated work. No compaction overhead, no carry-over, fresh prompt cache. Use `/compact` when continuity matters, `/new` when it doesn't.

## Try this

Open a real session JSONL and read it with your eyes (10–20 min):

```bash
ls ~/.openclaw/agents/*/sessions/*.jsonl | head
# Pick a long-ish one, then:
head -50 <path-to-session>.jsonl | jq -c '{id, parentId, type: (.type // .role)}'
```

In the file, find:

1. **The first user message.** Look for an entry with `role: "user"` and no `parentId` (or one pointing at the session header). This is the seed of the tree.
2. **A tool call/result pair.** An assistant entry with a `tool_use` block, immediately followed by a `tool_result` entry whose `parentId` points back at it. Note that compaction keeps these pairs together when splitting.
3. **A compaction event, if one happened.** Look for a summary-shaped entry, or check the directory for an archived successor — the older JSONL kept as the checkpoint source. If `truncateAfterCompaction` is on, you'll see two files for what you thought was one session.

Bonus: run `/status` in chat and confirm `🧹 Compactions: <count>` matches what you found on disk.

## Reflection

1. Why does OpenClaw store sessions as JSONL trees instead of flat arrays — and what would break if you replaced the format with a single rewritten JSON blob per turn?
2. Pruning and compaction both reduce context. When would you reach for each, and why is pruning safe to run aggressively while compaction is a once-in-a-while operation?
3. The compaction-safeguard and context-pruning extensions are OpenClaw additions on top of pi. What does that tell you about the boundary between pi-the-engine and OpenClaw-the-runtime — and where you'd add your own extension if you wanted, say, semantic deduplication of tool results?

---
[← Day 4](day-04-harnesses.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 6 →](day-06-system-prompt.md)
