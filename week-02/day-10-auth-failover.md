# Day 10 — Multi-account auth and failover

You hit a 429. Then another. Then your weekly window resets and you discover
your "free" plan was the only thing standing between you and a workflow that
actually finishes. Today is about wiring OpenClaw so that does not happen
twice — by stacking multiple credentials per provider and letting the runner
rotate through them, then falling over to a different model entirely when a
whole provider gives up.

## Mental model

One provider, many profiles. Profiles rotate on failure. If all profiles for a
provider die, the runner falls over to a different model.

The cleanest way to picture it:

```
agents.defaults.model.primary  →  anthropic/claude-sonnet-4-6
                                   ├── auth profile: anthropic:work
                                   │     fail → cooldown (1m, 5m, 25m, 1h)
                                   ├── auth profile: anthropic:personal
                                   │     fail → cooldown
                                   └── (all in cooldown? give up on provider)

agents.defaults.model.fallbacks →  openai/gpt-5
                                   ├── openai-codex:default  (OAuth)
                                   └── openai:api-key
```

Two stages, in this order: **auth profile rotation** inside the current
provider, then **model fallback** to the next entry in
`agents.defaults.model.fallbacks`. The first stage is invisible to the user —
the model id never changes. The second stage rewrites session state
(`providerOverride`, `modelOverride`, `modelOverrideSource: "auto"`) so the
next turn does not probe the known-bad primary again.

The trigger that bridges stage 1 to stage 2 is `FailoverError`. If the runner
exhausts every profile for a provider with a failover-worthy error, it throws
`FailoverError` and the outer `runWithModelFallback(...)` walks to the next
candidate.

## How it works internally

The auth profile store is loaded once per run by `ensureAuthProfileStore`:

```typescript
const authStore = ensureAuthProfileStore(agentDir, { allowKeychainPrompt: false });
const profileOrder = resolveAuthProfileOrder({ cfg, store: authStore, provider, preferredProfile });
```

`ensureAuthProfileStore` reads `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`,
imports the legacy `~/.openclaw/credentials/oauth.json` on first use, and
hands back a store with both API key and OAuth credentials keyed by profile id
(`provider:default`, `provider:<email>`, etc.).

`resolveAuthProfileOrder` produces the rotation list. It checks, in order:

1. `auth.order[provider]` if explicitly configured
2. `auth.profiles` filtered by provider (config metadata)
3. raw entries in `auth-profiles.json` for that provider

When no explicit order exists, the round-robin is **OAuth before API keys**
(primary key), then `usageStats.lastUsed` oldest-first (secondary key).
Cooldown/disabled profiles are pushed to the tail, ordered by soonest expiry.
Sessions also pin the chosen profile across turns to keep provider caches
warm — rotation happens on failure, not on every request.

When a profile fails, `markAuthProfileFailure` writes to
`~/.openclaw/agents/<agentId>/agent/auth-state.json` under `usageStats`:

```typescript
await markAuthProfileFailure({ store, profileId, reason, cfg, agentDir });
const rotated = await advanceAuthProfile();
```

The cooldown ladder is exponential: **1 minute → 5 minutes → 25 minutes → 1
hour cap**. Billing failures take a different lane — `disabledUntil` /
`disabledReason: "billing"` with a **5h start, 24h cap**. `advanceAuthProfile`
rotates to the next entry in `profileOrder` and returns `null` once the
provider is exhausted.

Exhaustion is where the second stage starts. The runner classifies the error
text and decides whether to escalate:

```typescript
if (fallbackConfigured && isFailoverErrorMessage(errorText)) {
  throw new FailoverError(errorText, {
    reason: promptFailoverReason ?? "unknown",
    provider,
    model: modelId,
    profileId,
    status: resolveFailoverStatus(promptFailoverReason),
  });
}
```

`isFailoverErrorMessage` matches the broad rate-limit/timeout/auth bucket
documented in `model-failover.md` (429, `ThrottlingException`,
`Too many concurrent requests`, `weekly limit reached`,
`Unhandled stop reason: error`, etc.). `classifyFailoverReason(errorText)`
returns one of:

```
"auth" | "rate_limit" | "quota" | "timeout" | "unknown"
```

The reason flows into `FailoverError` and into the structured
`model_fallback_decision` log so `/status` and `FallbackSummaryError` can
explain *why* the chain advanced. `quota` and `auth` typically jump straight
to the next provider; `rate_limit` and `timeout` allow a same-provider sibling
profile to be tried first.

Model resolution is owned by `resolveModel`:

```typescript
const { model, error, authStorage, modelRegistry } = resolveModel(
  provider, modelId, agentDir, config,
);
authStorage.setRuntimeApiKey(model.provider, apiKeyInfo.apiKey);
```

The resolution order is **explicit override → agent default → global default**:
a session `/model …@profile`, then the agent entry in `agents.list[].model`,
then `agents.defaults.model.primary`. Custom providers are written into
`~/.openclaw/agents/<agentId>/agent/models.json` (merged by default; set
`models.mode: "replace"` to overwrite). `models.json` is the registry the pi
runtime reads to translate `provider/model` strings into actual SDK calls.

OAuth is a separate code path inside the same store. Both
**OpenAI Codex (ChatGPT OAuth)** and Anthropic Claude CLI subscription auth
land in `auth-profiles.json` as `type: "oauth"` with
`{ access, refresh, expires, accountId? }`. Refresh happens under a file lock
when `expires` is in the past.

## Knobs you control

Auth profile config (`openclaw config get auth` to inspect):

- `auth.profiles` — metadata + routing only, no secrets. Use it to register a
  profile id without writing a credential.
- `auth.order[provider]` — pin rotation order, e.g.
  `["anthropic:work", "anthropic:personal"]`.
- `auth.cooldowns.billingBackoffHours` /
  `auth.cooldowns.billingBackoffHoursByProvider` — billing-disable backoff.
- `auth.cooldowns.billingMaxHours` — cap on billing backoff (default 24h).
- `auth.cooldowns.failureWindowHours` — window after which counters reset
  (default 24h).
- `auth.cooldowns.overloadedProfileRotations` — rotations allowed for
  overloaded errors before model fallback (default 1).
- `auth.cooldowns.overloadedBackoffMs` — backoff before that rotation
  (default 0).
- `auth.cooldowns.rateLimitedProfileRotations` — same idea for rate-limited
  errors.
- `OPENCLAW_SDK_RETRY_MAX_WAIT_SECONDS` — env var to cap Stainless SDK
  retry-after sleeps (default 60s). Without this, the SDK will swallow your
  failover window.

OAuth login and profile management:

- `openclaw models auth login --provider <id>` — interactive PKCE for
  OpenAI Codex, Anthropic, plugin providers.
- `openclaw models auth login --provider <id> --set-default` — also bumps
  `agents.defaults.model.primary`.
- `openclaw onboard` → auth choice `openai-codex` for the wizard path.
- `openclaw channels list --json` — shows registered profile ids per channel.

Model defaults and fallback chain:

- `agents.defaults.model.primary` — the starting model.
- `agents.defaults.model.fallbacks` — ordered list, walked on failover.
- `agents.list[].model` — per-agent override; **strict** unless the entry
  includes its own `fallbacks: [...]`. Use `fallbacks: []` to be explicit.
- `openclaw models fallbacks add <provider/model>` /
  `openclaw models fallbacks list` — manage the chain from the CLI.
- `openclaw models set <model>` — replace the primary.

## Power-user pattern

**The rate-limit-immune stack**: 2–3 OpenAI profiles + 1 Anthropic profile +
a Sonnet fallback. Order them by cost so the cheaper credential is tried
first.

```toml
[auth.order]
"openai-codex" = ["openai-codex:personal", "openai-codex:work"]
"openai" = ["openai:cheap-key"]

[agents.defaults.model]
primary = "openai/gpt-5"
fallbacks = ["anthropic/claude-sonnet-4-6"]
```

The math: with three OpenAI profiles each carrying an independent rate-limit
window, you have ~3× the per-minute capacity of a single account before
*any* failover happens. Each profile cooldown starts at 1 minute, so a single
profile burning out only blackouts that profile for 60s — the next profile
takes over within milliseconds. Only when *all three* OpenAI profiles are in
cooldown simultaneously does `FailoverError` fire and the runner switches to
Anthropic. By the time Sonnet is also unhappy, the first OpenAI profile's
1-minute cooldown has long expired and the next turn probes it again
(per-candidate decision, not permanent skip). Effective failure window for the
whole stack: ~seconds, not minutes.

## Try this

1. Inspect what you have:
   ```bash
   openclaw channels list --json | jq '.[].auth'
   openclaw models status
   ```
   `models status` shows the resolved primary, fallbacks, and OAuth expiry
   warnings (24h window by default).

2. Find your single point of failure. If any provider in your fallback chain
   has only one profile, that's the bottleneck.

3. Add redundancy — either a second profile on the weak provider:
   ```bash
   openclaw models auth login --provider openai-codex
   ```
   or a model fallback on a different provider:
   ```bash
   openclaw models fallbacks add anthropic/claude-sonnet-4-6
   ```

4. Verify with `openclaw models status --probe` — the `--probe` flag does a
   live auth check per profile, so you see which credentials actually work
   instead of which ones merely exist on disk.

## Reflection

1. Your primary uses OAuth and you have one API-key profile as backup. Round-
   robin tries OAuth first. If the OAuth token silently expires mid-session,
   what does the user *see* — and how does session pinning interact with the
   cooldown that gets written?

2. You set `agents.list[0].model = "anthropic/claude-sonnet-4-6"` with no
   `fallbacks`. The configured global default has a long fallback chain. A
   user `/model`s to Opus and it 429s. Which chain (if any) does the runner
   walk, and why?

3. `classifyFailoverReason` returns `"quota"` instead of `"rate_limit"`. What
   does that change about how OpenClaw treats the failing profile — cooldown
   length, whether sibling profiles are tried, whether the primary is probed
   again on the next turn?

---
[← Day 9](day-09-skills.md) · [Course home](../README.md) · [Glossary](../glossary.md) · [Day 11 →](day-11-sandbox.md)
