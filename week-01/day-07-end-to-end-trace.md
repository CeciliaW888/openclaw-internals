# Day 7 — Synthesis: trace one message end-to-end

## Mental model

You have now seen the five pieces (Day 1), the agent loop (Day 2), Pi (Day 3), harnesses (Day 4), sessions (Day 5), and the system prompt (Day 6). Today is the capstone: one concrete inbound message, walked from arrival to reply, naming every layer it crosses.

The scenario: a WhatsApp DM from an approved sender saying *"summarize the last 10 emails in my inbox"*.

Every message takes the same trip. There are five layers, and each is a gate the message has to pass:

1. **Channel** — the plugin that owns the wire (WhatsApp, Slack, Telegram, etc.).
2. **Policy** — sender allowlist, DM pairing, group rules. The "should this even run?" gate.
3. **Session** — routing to the right session key, opening the JSONL transcript, queueing if needed.
4. **Run** — the embedded Pi loop: prompt assembly, model call, tool execution, more model calls, final payload.
5. **Reply path** — block streaming, reply directives, channel send, transcript persistence.

Internalize this trip and every OpenClaw bug becomes a "which layer?" question. That is the entire skill. From now on, when something breaks, you will know which step to point at.

## The trace

A WhatsApp DM lands. Here is the full journey, step by step.

1. **Webhook arrival.** WhatsApp Business API POSTs to the gateway's WhatsApp plugin. The plugin parses the payload into a normalized inbound message — sender id, account id, body, message id, timestamp.

2. **Channel ingestion.** The WhatsApp plugin (one of the five pieces from Day 1) decodes the body into the gateway's internal `Body` / `CommandBody` shape from the messages doc. `Body` is what the model eventually sees; `CommandBody` is the raw text used for directive parsing.

3. **Inbound dedupe.** The gateway checks a short-lived dedupe cache keyed by `channel/account/peer/session/message id`. If WhatsApp redelivered after a reconnect, the duplicate dies here. Our message is fresh.

4. **Sender allowlist + DM policy check.** The channel docking layer (Day 1) checks whether this sender is allowed to talk to this account at all. For WhatsApp DMs, this is the "approved sender" gate the security docs describe. Unknown senders are rejected here — no session, no run.

5. **Multi-agent routing.** The gateway maps `(channel, account, sender)` to a target agent — the Day 1 binding: one gateway, many agents, the channel picks one.

6. **Inbound debouncing.** WhatsApp has `debounceMs: 5000`. If the user is mid-thumb-typing a follow-up, the debouncer waits for quiet before flushing.

7. **Session key resolution.** The gateway computes the session key — for a DM this collapses to the agent's main session key (e.g. `main:whatsapp:+1...`). Same key, same session file across messages.

8. **Queue admission.** The command queue (Day 2 / queue doc) admits the run on the per-session lane (`session:<key>`) and the global `main` lane. If a run were already in flight, we'd steer, follow up, or collect per `messages.queue.mode`. A typing indicator fires immediately so the user sees activity.

9. **Session open + write lock.** `SessionManager.open(sessionFile)` is called (cached if warm — Day 5). The session write lock is acquired. The inbound message is appended to the session tree (id/parentId linked).

10. **Embedded run entry.** `runEmbeddedPiAgent` is invoked with `sessionId`, `sessionKey`, `sessionFile`, `workspaceDir`, `prompt`, `provider`, `model`, `runId`, and an `onBlockReply` callback supplied by the WhatsApp plugin. This is the Day 4 moment: the harness hands control to the runtime.

11. **Auth profile + model resolution.** `resolveModel` picks the provider/model; `resolveAuthProfileOrder` selects an auth profile honoring cooldowns. The runtime API key is set on `authStorage`. Profiles in cooldown rotate before the model is contacted.

12. **System prompt assembly.** `buildAgentSystemPrompt` (Day 6) assembles base prompt, skills, bootstrap context, sandbox info, messaging info, reply-tag rules, runtime metadata. The `agent:bootstrap` hook fires here. Applied via `applySystemPromptOverrideToSession`.

13. **Pi session created.** `createAgentSession` runs with resolved model, tools (OpenClaw's exec/process/read/edit/write replacements plus messaging, browser, and the installed email tool), `sessionManager`, and `settingsManager`. This is the Day 3 boundary — past this line we are in Pi.

14. **First turn — prompt → model.** `session.prompt(effectivePrompt)` is called. Pi sends system prompt + transcript + new user message to the model. The model responds with a tool call: e.g. `email.list({ count: 10 })`.

15. **Tool execution.** Pi dispatches the call through the OpenClaw adapter (`pi-tool-definition-adapter`). Policy filtering already happened at registration. The tool runs with the abort signal wired in, fetches 10 emails, and returns a structured result. `content` (model-visible) and `details` (runtime metadata, stripped before replay) stay separate.

16. **Second turn — tool results → model.** Pi feeds the tool result back. The model now has the 10 emails in context and produces the summary as assistant text. `subscribeEmbeddedPiSession` streams deltas as `assistant` events.

17. **Block reply assembly.** `EmbeddedBlockChunker` cuts text into channel-safe chunks (respecting WhatsApp caps, not splitting fenced code). Reply directives (`[[reply:id]]`, `[[media:url]]`) are extracted via `consumeReplyDirectives`. `<think>` tags are stripped. `NO_REPLY` would short-circuit here.

18. **Channel send + persistence.** Each finished block triggers the harness's `onBlockReply`, which the WhatsApp plugin sends via the WhatsApp Business API. `message_sending` / `message_sent` plugin hooks fire around the send. The assistant turn (with tool call, result, final text) is written to the JSONL under the same write lock from step 9. Lifecycle `end` fires, the queue lane releases, run done.

## Where things fail

The same trace doubles as a triage map:

- **Step 4 (DM policy).** Sender not on allowlist, or DM pairing not completed. Symptom: silent drop, no session created.
- **Step 8 (queue).** `debounceMs` too long, or stuck behind a prior run that never released its lane. Symptom: typing indicator fires but reply takes forever. Verbose logs show `queued for …ms` lines.
- **Step 9 (session write lock).** Another writer (stale process, hand-edit) holds the file lock. Symptom: run never starts, or compaction wedges.
- **Step 11 (auth profile cooldown).** All profiles in cooldown after rate-limit failures. Symptom: `FailoverError` or rotation exhausted.
- **Step 12 (system prompt size).** Bootstrap / skills pushed the system prompt over the model's reserve. Symptom: context overflow before the user message even gets a turn.
- **Step 14/16 (context overflow).** Long history + 10 full emails exceed the window. Symptom: `request_too_large` / `context length exceeded`. Recovery: auto-compaction kicks in (`compaction_start`), run retries with a compacted transcript.
- **Step 15 (tool sandbox / policy denial).** Email tool isn't installed, or sandbox blocks the network call. Symptom: tool error reply, or model loops trying alternatives.
- **Step 17/18 (reply path).** Block streaming misconfigured (`blockStreaming` not explicitly `true` for WhatsApp), or `NO_REPLY` in a DM gets rewritten. Symptom: silence, or duplicate-looking replies under `steer-backlog`.
- **Runtime pin mismatch (cross-cutting).** Pinned Pi version doesn't match what the harness shipped; an expected event never fires. Symptom: lifecycle never ends, `agent.wait` times out. A Day 4 failure mode that haunts the whole trace.

## Power-user pattern

**When debugging, always identify *which step* the failure happened at.** Different layers have different logs, different config knobs, and different fixes. Saying "the bot didn't reply" is useless. Saying "the bot didn't reply, the typing indicator never fired" tells you the failure is before step 8 — channel or policy. Saying "typing indicator fired but no reply for 90 seconds" tells you the run started — you're in steps 11–18 territory, look at provider logs and lifecycle events.

Concretely:

- No session JSONL entry → failed at steps 1–9 (channel / policy / dedupe / queue).
- Session entry exists but no assistant turn → steps 10–14 (run start, auth, prompt, first model call).
- Tool call started, tool result missing → step 15 (tool execution).
- Assistant text in the JSONL but nothing on WhatsApp → steps 17–18 (reply path / channel send).

The trace is your decision tree. Memorize the five layers, and use the JSONL to bisect.

## Try this

Pick one real recent message you sent your own agent. Find the corresponding session JSONL under `~/.openclaw/agents/<agentId>/sessions/` (or the equivalent state dir from Day 3).

Walk the trace and check off what you can verify:

- Step 7 — does the session key match the channel + sender you'd expect?
- Step 9 — is the inbound user message there as the first new entry?
- Step 14 — is there an assistant turn that issued tool calls?
- Step 15 — for each tool call, is there a paired tool result entry?
- Step 16 — is there a final assistant text turn after the tool results?
- Step 18 — does the timestamp on the last assistant entry roughly match when the reply hit your phone?

Anything you *can't* verify from the JSONL lives in the gateway logs instead — that's a hint about which layer owns it. Write down which steps you confirmed and which you couldn't.

## Reflection

1. If a WhatsApp message gets a typing indicator but no reply for two minutes, which steps in the trace are still in play, and which are already ruled out?
2. The same message produces an empty assistant turn in the JSONL plus a "tool error" reply on the channel. What is your top hypothesis for which step failed, and why?
3. You add a new email summarization skill that bloats the bootstrap context. The next message overflows context before the model can even respond. Predict: at which step does the failure surface, and which recovery path does the runtime take?

---
[← Day 6](day-06-system-prompt.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 8 →](../week-02/day-08-tools.md)
