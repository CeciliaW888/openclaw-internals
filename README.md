# OpenClaw + Pi: Architecture for Builders

A 14-day deep dive into the architecture behind OpenClaw — the personal AI assistant — and Pi, the agent SDK that powers it. Written for people who already use OpenClaw and want to **build their own agent** for personal or work use.

> This is not a beginner tour. We don't install anything, we don't explain what an LLM is. We trace runs, decode harnesses, and look at what's actually happening when a message arrives on Telegram and a reply lands back two seconds later.

---

## Who this is for

You should already be able to:

- Run `openclaw onboard` and `openclaw agent` without looking it up
- Send a message through at least one channel (WhatsApp, Telegram, Slack, Discord…)
- Recognize chat commands like `/status`, `/new`, `/think`, `/compact`

If any of that is unfamiliar, do the OpenClaw [Getting Started](https://docs.openclaw.ai/start/getting-started) flow first. This course assumes that knowledge as the floor.

## What you'll be able to do at the end

- Read the OpenClaw source/docs and follow conversations about runtimes, harnesses, runs, turns, hooks, and sessions
- Trace any inbound message end-to-end and predict where it'll fail
- Design an agent — workspace, system prompt, skills, tools, sandbox, channel, triggers — that fits a real personal or work use case
- Decide *when* to write a custom skill vs. a tool vs. a hook vs. a plugin

## Time commitment

- **1 hour per day, 14 days** (≈ 2 weeks if you do it daily, longer if you don't)
- Each lesson is ~1000–1500 words plus a 10–20 minute hands-on exercise

## How each lesson is structured

Every day follows the same six sections:

1. **Mental model** — the picture you should hold in your head
2. **How it works internally** — what's actually happening, with file/function references where it helps
3. **Knobs you control** — config keys, CLI flags, env vars, design choices
4. **Power-user pattern** — a non-obvious technique you can apply now
5. **Try this** — a 10–20 minute hands-on exercise
6. **Reflection** — three questions; if you can answer them, you understood the lesson

---

## Curriculum

### Week 1 — Architecture deep dive

| Day | Lesson |
|---|---|
| 1 | [The 5-piece map: Gateway, Agent, Session, Channel, Node](week-01/day-01-five-piece-map.md) |
| 2 | [The agent loop: runs, turns, events, streaming](week-01/day-02-agent-loop.md) |
| 3 | [Pi, the engine inside](week-01/day-03-pi-engine.md) |
| 4 | [Harnesses and agent runtimes](week-01/day-04-harnesses.md) |
| 5 | [Sessions and memory](week-01/day-05-sessions.md) |
| 6 | [The system prompt is dynamic](week-01/day-06-system-prompt.md) |
| 7 | [Synthesis: trace one message end-to-end](week-01/day-07-end-to-end-trace.md) |

### Week 2 — Extend and build YOUR agent

| Day | Lesson |
|---|---|
| 8 | [Tools architecture](week-02/day-08-tools.md) |
| 9 | [Skills as a runtime contract](week-02/day-09-skills.md) |
| 10 | [Multi-account auth and failover](week-02/day-10-auth-failover.md) |
| 11 | [Sandbox and the trust boundary](week-02/day-11-sandbox.md) |
| 12 | [Hooks and Pi extensions](week-02/day-12-hooks.md) |
| 13 | Triggers and the automation surface — *coming soon* |
| 14 | Capstone: design YOUR agent — *coming soon* |

Glossary: [glossary.md](glossary.md) — plain-English definitions for every term you'll see.

---

## How to use this course

- **Read in order.** Concepts compound — Day 8 (tools) makes a lot more sense after Day 4 (harnesses).
- **Do the "Try this" exercises.** Reading is half the value. The other half is poking at your own running OpenClaw and seeing what changes.
- **Keep `docs.openclaw.ai` open in another tab.** This course is a guided tour through the architecture, not a replacement for the official docs.
- **Don't trust me — verify.** Source paths and file names are anchored to the OpenClaw repo. If something looks off in your version, the source is the source of truth.

## Source material

Every lesson cites:

- The relevant OpenClaw docs page on [docs.openclaw.ai](https://docs.openclaw.ai)
- Specific source files in the [openclaw/openclaw](https://github.com/openclaw/openclaw) repo when implementation details matter
- The Pi packages on [GitHub: badlogic/pi-mono](https://github.com/badlogic/pi-mono) when we cross into Pi territory

If a doc page is reorganized after this course is written, the linked URL might 404. The concepts under it won't.

---

## License

MIT — copy, adapt, fork. The course is content; OpenClaw and Pi are their respective projects' licenses.
