# Day 1 — The 5-piece map: Gateway, Agent, Session, Channel, Node

You have OpenClaw running. You have run `openclaw agent`. You have probably also stared at the docs and noticed that the same five words keep showing up: **Gateway**, **agent**, **session**, **channel**, **node**. Today is the architectural floor. Once these five surfaces sit cleanly in your head, every later page in the docs collapses into "ah, that's a thing one of the five owns."

## Mental model

OpenClaw is one long-lived **Gateway** process per host. It is the only place that opens a WhatsApp Baileys session, holds Telegram/Slack/Discord/Signal/iMessage connections, and exposes a typed WebSocket API on `127.0.0.1:18789`. Everything else hangs off that one daemon.

```
                   +-----------------------------------+
                   |  Gateway (one per host, :18789)   |
                   |  - provider connections           |
                   |  - WS API + JSON Schema validate  |
                   |  - canvas host /__openclaw__/*    |
                   +-----------------------------------+
                     |          |              |     |
       channels  ----+          |              |     +----  control clients
      (WA/TG/Slack/             |              |          (CLI / mac app /
       Discord/...)             |              |           web Control UI)
                                |              |
                       +--------+              +--------+
                       |                                |
                  +---------+                     +-----------+
                  |  Agent  |   ... per persona   |   Nodes   |
                  | (brain) |                     | (mac/iOS/ |
                  +---------+                     |  Android) |
                       |                          +-----------+
                  +---------+
                  | Session | (one per (agent, peer) thread)
                  | JSONL   |
                  +---------+
```

Channels and nodes both ride the Gateway. Agents are the brains the Gateway runs. Sessions are the persistent threads agents talk through. Control clients are how *you* talk to the Gateway.

## How it works internally

**Gateway (daemon).** Started with `openclaw gateway`. It maintains every provider connection, validates inbound WS frames against JSON Schema, and emits events (`agent`, `chat`, `presence`, `health`, `heartbeat`, `cron`). Handshake is mandatory: any non-`connect` first frame is a hard close. Exactly one Gateway controls a single Baileys (WhatsApp) session per host. The HTTP server on the same port also serves `/__openclaw__/canvas/` and `/__openclaw__/a2ui/`.

**Agent.** A *fully scoped persona*: workspace, state directory, auth profiles, model registry, session store. The Gateway can host one agent (default `agentId: main`) or many side-by-side. Per-agent paths:

- Workspace: `~/.openclaw/workspace` (or `~/.openclaw/workspace-<agentId>`) — the agent's only `cwd` for tools and context. Holds `AGENTS.md`, `SOUL.md`, `TOOLS.md`, `BOOTSTRAP.md`, `IDENTITY.md`, `USER.md`, plus `skills/`.
- State / `agentDir`: `~/.openclaw/agents/<agentId>/agent/` — auth profiles, model registry, per-agent config. `auth-profiles.json` lives here and is **not** shared between agents.
- Sessions: `~/.openclaw/agents/<agentId>/sessions/<SessionId>.jsonl`.

The runtime is the embedded Pi agent core: models, tools, prompt pipeline are Pi; session management, discovery, tool wiring, and channel delivery are OpenClaw layers on top. On the first turn of a new session, OpenClaw injects the bootstrap files directly into the agent context.

**Session.** A persistent JSONL transcript with tree structure (id/parentId), opened by Pi's `SessionManager`. The session ID is stable and chosen by OpenClaw. Direct chats collapse to the agent's **main session key** (`agent:<agentId>:<mainKey>`); groups get their own keys. Sessions also hold delivery state — `lastChannel`, `lastTo`, `lastAccountId` — which is what channel docking rewrites.

**Channel.** A messaging surface (WhatsApp, Telegram, Slack, Discord, Signal, iMessage, WebChat, Matrix, etc.). A channel can have multiple **accounts** (`accountId`) — e.g. WhatsApp `personal` and `biz` are two accounts on one channel. Inbound messages are routed to an agent through **bindings**, evaluated most-specific-first: peer → parentPeer → guildId+roles → guildId → teamId → accountId → channel-wide → default agent.

**Node.** A device that connects to the same WS server but declares `role: node` with explicit caps/commands (`canvas.*`, `camera.*`, `screen.record`, `location.get`). Nodes are macOS, iOS, Android, or headless. Pairing is **device-based** and lives in the device pairing store; new device IDs require approval and the Gateway issues a device token for subsequent connects. All connects sign the `connect.challenge` nonce (signature payload `v3` binds `platform` + `deviceFamily`).

The boundary is sharp: channels carry messages from humans; nodes carry capabilities from devices. Both speak the Gateway WS protocol; only nodes set `role: node`.

## Knobs you control

Config lives at `~/.openclaw/openclaw.json` (or `OPENCLAW_CONFIG_PATH`). State root is `~/.openclaw` (or `OPENCLAW_STATE_DIR`).

- **Agent shape**
  - `agents.defaults.workspace` — the agent's `cwd`.
  - `agents.defaults.model`, `agents.defaults.models` — `provider/model`, split on first `/`.
  - `agents.defaults.skipBootstrap` — turn off bootstrap file creation for pre-seeded workspaces.
  - `agents.defaults.sandbox.mode` (`off` / `non-main` / `all`), `sandbox.scope`, `sandbox.workspaceRoot` for per-session workspaces.
  - `agents.defaults.blockStreamingDefault`, `blockStreamingBreak`, `blockStreamingChunk`, `blockStreamingCoalesce` — streaming behavior.
  - `agents.list[]` — multiple isolated personas with their own `workspace`, `agentDir`, `model`, `tools.allow/deny`, `groupChat.mentionPatterns`.
- **Routing**
  - `bindings[]` — `{ agentId, match: { channel, accountId, peer, parentPeer, guildId, teamId, roles } }`.
  - `channels.<channel>.defaultAccount`, `channels.<channel>.accounts.<id>.*`.
  - `session.identityLinks` — required for `/dock-*` channel docking.
- **Gateway**
  - `gateway.auth.mode` (`none` / `trusted-proxy` / shared-secret), `gateway.auth.allowTailscale`.
  - Default bind `127.0.0.1:18789`. Start: `openclaw gateway`.
- **Tools / skills**
  - `tools.exec.applyPatch`, `tools.agentToAgent.{enabled,allow}`.
  - Skill load order: workspace → `<workspace>/.agents/skills` → `~/.agents/skills` → `~/.openclaw/skills` → bundled → `skills.load.extraDirs`.

Run `openclaw agents list --bindings` and `openclaw channels status --probe` to see the resolved view.

## Power-user pattern

**Treat each agent's workspace as a versioned repo.** The workspace at `~/.openclaw/workspace-<agentId>` is just a directory of plain Markdown (`AGENTS.md`, `SOUL.md`, `USER.md`, `TOOLS.md`, `IDENTITY.md`) plus a `skills/` folder. Because OpenClaw injects those files on the first turn of a new session, edits become persona changes the moment a new session starts.

Make the workspace a git repo. Now persona evolution has a diff. You can branch a "more terse" `SOUL.md`, A/B it against your everyday persona by spinning up a second agent (`openclaw agents add terse`, point its `workspace` at the branch worktree), bind it to a throwaway Telegram bot via `bindings`, and compare. Roll back with `git revert` if it gets weird. The same trick lets you share a persona — push the workspace, let someone else clone it into their `~/.openclaw/workspace-<agentId>` and add an `agents.list[]` entry. You are versioning a personality, not a config file.

This works *because* of the split: workspace is files, `agentDir` is credentials, sessions are transcripts. Only the first one is safe to put under git.

## Try this

Ten-to-twenty-minute exercise on your running Gateway:

1. `openclaw agents list --bindings` — read the output, identify your `agentId` (likely `main`) and any active bindings.
2. `ls ~/.openclaw/agents/<agentId>/` — note the `agent/` (state) and `sessions/` (JSONL) split. Open one `.jsonl` file and confirm it is a tree of messages.
3. `ls ~/.openclaw/workspace*` — open `AGENTS.md` and `SOUL.md`. These are what get injected on session start.
4. Send one message from your favorite channel. Then `tail -n 5` the newest file under `sessions/`. You just watched a channel-routed message land in a specific agent's session store.
5. Bonus: in another terminal, `openclaw channels status --probe`. Map every channel/account row back to a binding row from step 1.

You should now be able to point at any byte on disk and say which of the five owns it.

## Reflection

1. A WhatsApp DM from a new number arrives. Trace, in order, which of the five touches it and where it ends up on disk.
2. What is the difference between `agentDir` and `workspace`, and why is it unsafe to share `agentDir` across agents but safe (encouraged) to git-track a workspace?
3. A node and a control client both open a WebSocket to `:18789`. What single field in `connect` distinguishes them, and what does pairing look like for each?

---
[← Course home](../README.md) · [Glossary](../glossary.md) · [Day 2 →](day-02-agent-loop.md)
