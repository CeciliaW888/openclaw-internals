# Day 8 — Tools architecture

Welcome to Week 2. Last week you traced a message end-to-end; this week you start extending. The natural place to begin is tools — because "what can my agent actually do?" is, mechanically, "what tools are in its registry after the pipeline filters run?" Today we open that pipeline up.

## Mental model

Tools are functions. The model decides when to call one; the runtime executes it; the result goes back into the conversation. That part is boring. The interesting part is that the agent never sees the raw set of functions you wrote — it sees a *projection* of that set, filtered by policy, normalized for the current provider's schema quirks, wrapped to honor cancellation. By the time a tool definition reaches the model in `tools: [...]`, it has been through seven distinct stages.

`pi.md` lays the pipeline out exactly:

```
1. Base Tools          (pi codingTools: read, bash, edit, write)
2. Custom Replacements (OpenClaw swaps bash → exec/process; sandbox-aware read/edit/write)
3. OpenClaw Tools      (messaging, browser, canvas, sessions, cron, gateway, ...)
4. Channel Tools       (Discord/Telegram/Slack/WhatsApp action tools)
5. Policy Filtering    (profile / provider / agent / group / sandbox allow/deny)
6. Schema Normalization(Gemini & OpenAI quirks fixed up)
7. AbortSignal Wrapping(every tool wrapped to respect cancellation)
```

The ordering matters. Stages 1–4 are *additive* — the registry grows. Stage 5 is *subtractive* — policy removes things based on who you are, where you're running, what model you're talking to. Stages 6–7 are *transformative* — the surviving tools are reshaped, but the set is fixed. If a tool is missing in your agent and you didn't deny it, the bug is almost always between stages 5 and 6: profile too narrow, sandbox clamping, or provider-specific exclusion. Fix your mental search to that band first.

## How it works internally

**Stage 1 — Base Tools.** pi-coding-agent ships with a default set: `read`, `bash`, `edit`, `write`. These are the primitives every coding agent needs. If you embed pi directly, you get this set and nothing else.

**Stage 2 — Custom Replacements.** OpenClaw doesn't accept pi's defaults wholesale. It replaces `bash` with split `exec` / `process` tools (better cancellation, better sandbox semantics) and wraps `read` / `edit` / `write` so they can refuse paths outside the active sandbox root. See `pi-tools.read.ts` and friends.

**Stage 3 — OpenClaw Tools.** `createOpenClawCodingTools()` (in `pi-tools.ts`) layers in the agent-suite tools: `messaging`, `browser`, `canvas`, the `sessions_*` family (`sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`, `sessions_yield`, `subagents`, `session_status`), `cron`, `gateway`, image and node tools, and so on. These live under `src/agents/tools/*.ts` plus the higher-level `openclaw-tools.ts`.

**Stage 4 — Channel Tools.** `channel-tools.ts` injects channel-specific actions when the active channel needs them — Discord reactions, Telegram message edits, Slack threads, WhatsApp media. A tool that exists in a Discord session may not exist in a CLI session; that's by design.

**Stage 5 — Policy Filtering.** `pi-tools.policy.ts` and `tool-policy.ts` apply allow/deny in layers: the **profile** (`coding`, `messaging`, …) sets a baseline; **provider**, **agent**, **group**, and **sandbox** policies subtract from that baseline. `alsoAllow` and explicit deny lists are merged in here. The `/tools` slash command in any session prints the *post-filter* list — that's your ground-truth view.

**Stage 6 — Schema Normalization.** `pi-tools.schema.ts` cleans the JSON Schema of each tool so it survives the active provider. Gemini rejects certain `$defs` constructs and unknown JSON Schema keywords; OpenAI's strict-mode tool calling has its own rules around `additionalProperties` and required fields. Same `ToolDefinition` in, different sanitized schema out per provider.

**Stage 7 — AbortSignal Wrapping.** `pi-tools.abort.ts` wraps each surviving tool so it sees the session's `AbortSignal`. When the user hits Esc, or a parent agent kills a sub-agent via `subagents action: "kill"`, every in-flight tool call gets the abort and is expected to bail.

**The interface mismatch.** pi-agent-core exposes `AgentTool`, whose `execute` signature is `(toolCallId, params, signal, onUpdate)`. pi-coding-agent's `ToolDefinition.execute` is `(toolCallId, params, onUpdate, ctx, signal)` — different parameter order, plus a context slot. `pi-tool-definition-adapter.ts` bridges the two with `toToolDefinitions`:

```typescript
export function toToolDefinitions(tools: AnyAgentTool[]): ToolDefinition[] {
  return tools.map((tool) => ({
    name: tool.name,
    label: tool.label ?? name,
    description: tool.description ?? "",
    parameters: tool.parameters,
    execute: async (toolCallId, params, onUpdate, _ctx, signal) => {
      return await tool.execute(toolCallId, params, signal, onUpdate);
    },
  }));
}
```

Trivial code, load-bearing role: every OpenClaw tool flows through it.

**The split-tools choice.** pi exposes two slots when creating a session: `builtInTools` (which the SDK's own loop knows about) and `customTools` (extras layered on top). OpenClaw refuses to use `builtInTools`. From `pi.md`:

```typescript
export function splitSdkTools(options: { tools: AnyAgentTool[]; sandboxEnabled: boolean }) {
  return {
    builtInTools: [], // Empty. We override everything
    customTools: toToolDefinitions(options.tools),
  };
}
```

Why? Because if some tools came in via `builtInTools` they would skip OpenClaw's policy filter. Funneling everything through `customTools` keeps a single chokepoint where allow/deny, sandbox, and provider rules apply uniformly. One set of rules; one place to debug.

## Knobs you control

**Profile.** Top-level `tools.profile` picks a baseline: `"coding"` (full session orchestration including spawn), `"messaging"` (cross-session messaging only, no sub-agent spawn), and a few others. Pick the smallest profile that fits and add specifics rather than starting wide and denying.

**alsoAllow / deny.** Inside the `tools` block: `alsoAllow: ["sessions_spawn", "sessions_yield", "subagents"]` adds back tools the profile dropped; explicit deny lists strip them out. This is layered — provider, agent, group, and sandbox policies can each add their own deny list, and union of denies wins. The `sessions_*` tools (`sessions_list`, `sessions_history`, `sessions_send`, `sessions_spawn`) are the most commonly elevated *or* denied. They're powerful: read another session's transcript, message it, spawn children. Deny them by default in untrusted contexts.

**Per-channel tool config.** Channel plugins can inject their own tools (stage 4) and can also set `tools` overrides per channel — useful when you want WhatsApp restricted but Discord full-power on the same agent.

**Per-agent overrides.** In a multi-agent deployment, each agent's config can refine the base set: shop-front agent gets `messaging` + `gateway`; coding agent gets the full `coding` profile.

**Sandbox tool restrictions.** When a session is sandboxed, the policy clamps automatically: session tool visibility drops to `tree`, write tools route through sandbox-aware variants, and dangerous tools (raw `exec` outside the sandbox root, `cron`, sometimes `sessions_send`) are removed regardless of profile.

**`apply_patch` for OpenAI Codex.** OpenAI's Codex models prefer the `apply_patch` tool over OpenClaw's normal `edit`. `apply-patch.ts` provides it; `pi.md` calls out that it's enabled specifically when the provider/model is in the Codex family. You don't usually toggle this manually — model selection drives it.

## Power-user pattern

**Deny-list inversion for non-`main` sandbox sessions.** When you spawn a sub-agent into a sandbox session, the temptation is to give it the `coding` profile and call it done. The cleaner pattern is to start from `coding` and explicitly *deny* the small set of tools that can leak out of the sandbox:

```json5
{
  tools: {
    profile: "coding",
    deny: [
      "sessions_send",   // can't message arbitrary other sessions
      "sessions_spawn",  // can't fork more children
      "cron",            // can't schedule itself
      "gateway",         // can't reach external HTTP
    ],
  },
}
```

Why deny-list rather than allow-list? Because OpenClaw adds new tools regularly. An allow-list quietly excludes new capabilities you'd actually want; a deny-list keeps your sub-agent useful as the platform grows, and forces you to think about each new tool only when it becomes a hazard. The sandbox already clamps file writes and visibility — the deny list is the small extra perimeter on top.

## Try this

Roughly 15–20 minutes:

1. Open any active session and run `/tools`. That's the post-filter list — stages 1–5 already happened.
2. Cross-reference each tool: which are pi base (`read`, `edit`, `write`, `exec`/`process`)? Which are OpenClaw additions (anything `sessions_*`, `messaging`, `browser`, `canvas`, `cron`, `gateway`)? Which are channel-specific (only present because you're in Discord/Slack/etc.)?
3. Edit your agent config: add `tools.deny: ["cron"]`, restart the session, run `/tools` again, and verify `cron` is gone.
4. Spawn a sandboxed sub-agent (`sessions_spawn` with `sandbox: "require"`) and run `/tools` inside it. Compare. Note which tools the sandbox clamp removed even without explicit deny.

You're now reading the pipeline through its observable output.

## Reflection

1. Why does OpenClaw force every tool through `customTools` and leave `builtInTools` empty, instead of using both slots? What would break if a single tool slipped in via `builtInTools`?
2. The `AgentTool` ↔ `ToolDefinition` adapter is fifteen lines. What architectural cost would you pay if those two interfaces converged into one shared type across pi-agent-core and pi-coding-agent? What would you gain?
3. You discover one of your agents is calling `sessions_send` to message a session it shouldn't see. Walk the seven-stage pipeline and name the two stages where a fix could plausibly live. Which is the right place, and why?

---
[← Day 7](../week-01/day-07-end-to-end-trace.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 9 →](day-09-skills.md)
