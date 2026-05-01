# Day 12 — Hooks and Pi extensions

You have spent eleven days watching Pi behave. Today you learn how to change its behavior. Not by editing prompts, not by registering tools, not by writing skills — by inserting code into the Pi runtime itself. That seam is called a Pi extension, and OpenClaw is one of its heaviest users.

## Mental model

A Pi session is a state machine: it loads messages, sends them to a provider, parses tool calls, runs them, persists results, and repeats until the agent says it is done. A Pi **extension** is a programmable plug-in that hooks into specific points in that loop and modifies what happens there. The hooks are the named attachment points; the extension is the code that runs at one or more of them.

```
        prompt arrives
              │
              ▼
   ┌────────────────────┐
   │ session loop       │
   │                    │
   │  build context  ───┼──▶ context-pruning extension
   │  call provider     │     (drops old prunable tool results)
   │  execute tools     │
   │  persist results   │
   │  about to compact  ┼──▶ compaction-safeguard extension
   │                    │     (adaptive budget, failure summaries)
   │  emit events       │
   └────────────────────┘
              │
              ▼
        reply / next turn
```

Distinguish this from skills and tools. A **skill** changes *what the model does* by giving it new guidance and triggers. A **tool** gives the model a new action to take. A Pi **extension** does neither — it changes *how the runtime behaves* underneath the model. The model never sees an extension; it just notices that compaction came out cleaner or that its context did not blow up. Extensions are where you put behavior that has to be invariant of what the agent decides on any given turn.

## How it works internally

Pi exposes its extension loader through `DefaultResourceLoader`, which OpenClaw constructs in `runEmbeddedAttempt` (`src/agents/pi-embedded-runner/run/attempt.ts`). The loader scans three places: the agent directory (`~/.openclaw/agents/<agentId>/extensions/`), the standard Pi paths (`~/.pi/extensions/` and the cwd), and any `additionalExtensionPaths` the host passes in. Files it finds are imported as ES modules, registered against named hook points, and invoked when the session runs.

```typescript
const resourceLoader = new DefaultResourceLoader({
  cwd: resolvedWorkspace,
  agentDir,
  settingsManager,
  additionalExtensionPaths,
});
await resourceLoader.reload();
```

OpenClaw's twist is that some extensions are not files on disk; they are programmatic. The helper `loadEmbeddedExtensions` (in `src/agents/pi-embedded-runner/extensions.ts`) decides which built-in extensions to enable for this run, calls each one's `setRuntime` function to bind it to the active session, and *then* appends a path so the loader pulls in the matching extension module. The runtime call gives the extension session-scoped state; the path makes Pi actually wire it into the hook surface.

Two production extensions ship with OpenClaw:

**compaction-safeguard.** When `agents.defaults.compactionMode === "safeguard"` (also forced when a custom compaction provider is configured), OpenClaw enables this extension:

```typescript
if (resolveCompactionMode(params.cfg) === "safeguard") {
  setCompactionSafeguardRuntime(params.sessionManager, { maxHistoryShare });
  paths.push(resolvePiExtensionPath("compaction-safeguard"));
}
```

The safeguard does two things during the compaction lifecycle. First, **adaptive token budgeting**: instead of letting Pi summarize with a fixed token cap, it computes the share of history the summary is allowed to occupy (`maxHistoryShare`) and clamps the budget so the summary never crowds out recent turns. Second, **tool failure and file operation summaries**: it scans the chunk being summarized for failed tool calls and write/edit operations and synthesizes a short structured note that gets prepended to the summary. The model losing chat history is one thing; losing the fact that three writes failed in a row is another, and the safeguard makes sure that signal survives.

**context-pruning.** When `cfg?.agents?.defaults?.contextPruning?.mode === "cache-ttl"`, OpenClaw enables a pruner that watches *prompt cache TTLs* rather than message age:

```typescript
if (cfg?.agents?.defaults?.contextPruning?.mode === "cache-ttl") {
  setContextPruningRuntime(params.sessionManager, {
    settings,
    contextWindowTokens,
    isToolPrunable,
    lastCacheTouchAt,
  });
  paths.push(resolvePiExtensionPath("context-pruning"));
}
```

`isToolPrunable` is a per-tool predicate (large blob-like outputs from `read`, `web_fetch`, search results — yes; small status replies — no). `lastCacheTouchAt` records the last time the provider's prompt cache served a hit on a given prefix. When a tool result has not been touched in cache for longer than the TTL window and the session is approaching `contextWindowTokens`, the extension drops it from the replayed message stream. The transcript on disk is untouched; only the model-visible context shrinks. Compaction summarizes; pruning forgets. Together they keep long sessions stable without the heavy summary cost on every turn.

Both extensions resolve their bundled module via `resolvePiExtensionPath(name)`, which points into OpenClaw's installed package. So the two-step pattern — call `setRuntime`, append path — is the contract: the runtime call configures the session-bound singleton, the path tells Pi's loader to import the module that will actually call into that singleton at hook time.

## Knobs you control

The extension surface is configured under `agents.defaults` in `openclaw.json`:

- `agents.defaults.compactionMode`: `"safeguard"` enables compaction-safeguard. Default is unset (Pi's stock compaction). Setting `agents.defaults.compaction.provider` forces safeguard mode automatically.
- `agents.defaults.compaction.maxHistoryShare`: the budget cap the safeguard enforces. Lower values shrink summaries and keep more recent tail.
- `agents.defaults.contextPruning.mode`: `"cache-ttl"` enables context-pruning. Other values disable it.
- `agents.defaults.contextPruning` sub-keys configure the TTL window, the tool prunability list, and the context-window threshold the pruner watches.

For your own extensions, two config paths matter:

- `additionalExtensionPaths` (passed into the resource loader) lets the host hand Pi extra search roots without the user touching disk. Plugins can add to this list at registration time.
- The agent directory: dropping `~/.openclaw/agents/<agentId>/extensions/<name>/index.ts` (or `.js`) is the no-config way to install a custom extension for a single agent. The loader picks it up on the next `reload()`.

A plugin can also auto-install extensions. The plugin manifest's `openclaw.extensions` field (see `building-plugins`) lists relative paths that `ResourceLoader` includes when the plugin is enabled, so a published plugin can ship runtime behavior changes alongside its tools and hooks. This is the right surface for an extension that should travel with a plugin rather than live as a one-off file in someone's agent dir.

## Power-user pattern

The smallest useful extension is one that observes without modifying. Here is the scaffold for an extension that logs every turn boundary to a file in the agent directory. Drop this at `~/.openclaw/agents/main/extensions/turn-logger/index.ts`:

```typescript
import fs from "node:fs";
import path from "node:path";

export default {
  name: "turn-logger",
  hooks: {
    onTurnStart(ctx: { sessionId: string; turnIndex: number }) {
      const line = JSON.stringify({
        ts: new Date().toISOString(),
        sessionId: ctx.sessionId,
        turn: ctx.turnIndex,
        event: "start",
      });
      fs.appendFileSync(path.join(ctx.agentDir, "turn-log.jsonl"), line + "\n");
    },
    onTurnEnd(ctx: { sessionId: string; turnIndex: number; tokensIn: number; tokensOut: number }) {
      const line = JSON.stringify({
        ts: new Date().toISOString(),
        sessionId: ctx.sessionId,
        turn: ctx.turnIndex,
        event: "end",
        tokensIn: ctx.tokensIn,
        tokensOut: ctx.tokensOut,
      });
      fs.appendFileSync(path.join(ctx.agentDir, "turn-log.jsonl"), line + "\n");
    },
  },
};
```

That is the entire shape. The default export is an object with a `name` and a `hooks` map; each hook key matches a Pi runtime hook name; the function receives a context object specific to that hook. Extend by adding more keys. Pi's hook catalog is the authoritative list — match the names exactly.

The decision tree, when you are not sure where to put new behavior:

- **New action the model can take** → tool. Registers via `api.registerTool`, schema-validated, model-visible.
- **New guidance triggered by a situation** → skill. Lives in `~/.claude/skills/`, attached via triggers, model-visible.
- **Runtime behavior independent of what the model decides** → Pi extension. Loaded by `ResourceLoader`, hooks into the session loop, model-invisible.

If the change must hold even when the agent forgets to invoke it, it is an extension.

## Try this

Enable verbose logging and the safeguard, then push a session over the compaction line:

```json
{
  "agents": {
    "defaults": {
      "compactionMode": "safeguard",
      "compaction": { "notifyUser": true, "maxHistoryShare": 0.4 }
    }
  }
}
```

Run a long session — easiest way is to feed it a few large file reads in a row until you see `🧹 Auto-compaction complete`. Tail `~/.openclaw/logs/` (or run with `--verbose`) and look for the safeguard's adaptive-budget log line announcing the clamped token target plus the failure-summary block prepended to the compaction input. Confirm `/status` shows the compaction count incremented.

For pruning, set `agents.defaults.contextPruning.mode = "cache-ttl"`, run a session that fetches several large tool outputs (web search, big reads), and watch the active-token count in `/status` between turns. When prunable tool results age out of cache, you should see token usage drop without a corresponding compaction count increase — that is the pruner working in-memory only.

## Reflection

1. The safeguard runs only when `compactionMode === "safeguard"`. What is the trade-off the default mode is making, and when should you accept it?
2. Compaction summarizes; pruning forgets. If you had to keep exactly one of them on for an always-on long-running session, which would you pick, and what failure mode does that choice expose you to?
3. You want every tool call over 30 seconds to be logged with the full parameters. Should that be a tool wrapper, a plugin hook, or a Pi extension? Why?

---
[← Day 11](day-11-sandbox.md) · [Course home](../README.md) · [Glossary](../glossary.md) · *Day 13 coming soon*
