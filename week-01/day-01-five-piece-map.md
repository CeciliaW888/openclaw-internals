# Day 1 — The 5-piece map: Gateway, Agent, Session, Channel, Node

You've already used OpenClaw — you know what it feels like. Today is about understanding *why* it works the way it does. Five words keep appearing throughout the docs: **Gateway**, **Agent**, **Session**, **Channel**, **Node**. Once you have a clear picture of each, everything else in the docs will feel familiar.

## The big picture

Think of OpenClaw like a **radio station**.

The station has one central control room (the **Gateway**) that keeps everything running. The station has on-air personalities (the **Agents** — your AI). Each personality keeps a conversation log (the **Session**). The station broadcasts and receives on different frequencies — WhatsApp, Telegram, Slack, and so on (the **Channels**). And the station has remote equipment out in the field — phones, cameras, laptops — that feed it extra capabilities (the **Nodes**).

Here's how they relate:

```
                      [ Gateway — the control room ]
                               |
          ┌────────────────────┼────────────────────┐
          │                    │                    │
    Channels              Agents               Nodes
  (WhatsApp,          (your AI bots,        (your phone,
  Telegram,            each with its          laptop,
  Slack…)              own Sessions)          camera…)
```

Everything flows through the Gateway. Nothing talks to anything else directly.

## The five pieces, one by one

### Gateway — the control room

The Gateway is the always-on background process that holds everything together. It's the only part of OpenClaw that talks directly to WhatsApp, Telegram, Slack, and your other messaging services. It also listens for your Agents and your Nodes.

You don't usually interact with the Gateway directly — it just runs quietly in the background. Think of it like the engine in your car: you drive using the steering wheel and pedals, not by reaching into the engine.

**One machine, one Gateway.** You only ever have one Gateway running on a device at a time.

### Agent — the personality

An Agent is your AI — its name, its tone, what it knows, what it's allowed to do. You might have one Agent for personal use and another for work. Each Agent has its own identity, its own memory files, and its own conversation history.

When a message comes in from WhatsApp or Telegram, the Gateway decides which Agent should handle it (based on rules you've set up), and hands it off.

**Analogy:** Each Agent is like a different staff member at a helpdesk. The Gateway is the receptionist who decides which staff member takes your call.

### Session — the conversation thread

Every ongoing conversation between an Agent and a person is a **Session**. It's a log of the full back-and-forth — everything said, in order. Sessions persist, meaning if you close the app and come back tomorrow, the Agent can still remember the conversation.

If you message your Agent on WhatsApp, that's one Session. If the same Agent is also chatting with someone else on Telegram, that's a different Session. They don't mix.

**Analogy:** Sessions are like separate chat threads. Each conversation has its own thread, even if the same Agent is in all of them.

### Channel — where messages come from

A Channel is a messaging service — WhatsApp, Telegram, Slack, Discord, iMessage, and others. Your Gateway can be connected to several Channels at once.

One Channel can also have multiple accounts. For example, you might connect both your personal WhatsApp number and a WhatsApp Business number — those are two **accounts** on the same Channel.

**Analogy:** Channels are like different phone lines coming into the control room. One line might be your personal number, another your business number, another a Telegram bot.

### Node — a device in the field

A Node is a device — your phone, your laptop, a tablet — that connects to the Gateway and gives it extra abilities. For example, a phone Node can share your location or take a photo. A laptop Node can share its screen.

Nodes are optional. If you just want a chatbot on WhatsApp, you don't need any Nodes. But if you want your Agent to be able to see through your phone's camera or know where you are, a Node makes that possible.

**Analogy:** Nodes are like field reporters calling in to the radio station with live footage and on-the-ground updates. The station (Gateway) incorporates what they send.

## How a message flows

Here's what happens when someone sends your Agent a WhatsApp message:

1. **WhatsApp** receives the message and passes it to your **Gateway** (via the Channel connection).
2. The **Gateway** looks at who sent it and routes it to the right **Agent**.
3. The **Agent** reads the message, thinks about it, and writes a reply — all recorded in the **Session**.
4. The reply travels back through the **Gateway** to **WhatsApp**, and the person receives it.

That's the whole loop. Nodes can feed extra information into step 3 if needed (e.g. your current location), but they're not required for the basic flow.

## What this means for you

You don't need to configure any of this to use OpenClaw — you've already done that. But knowing the five pieces helps you in two practical ways:

- **When something goes wrong**, you'll know which piece to look at. Message not arriving? Probably a Channel issue. Agent giving a weird response? Probably a Session or Agent config issue.
- **When you want to customise**, you'll know what to change. Want a different personality? Edit the Agent. Want to add WhatsApp Business? Add a second account to your WhatsApp Channel.

## Reflect

Before moving on, see if you can answer these in plain words (no right or wrong answer — just check your own understanding):

1. Someone sends you a WhatsApp message. Name the five pieces in the order they get involved.
2. You have two WhatsApp numbers — personal and business. Is that two Channels or two accounts on one Channel?
3. What's the difference between a Channel and a Node? (Hint: think about where each one *sends from*.)

---
[← Course home](../README.md) · [Glossary](../glossary.md) · [Day 2 →](day-02-agent-loop.md)
