# Decision model (d1) — one-call decisions (`/v1/systemone`)

> Minimum engine: **v2.5.0-ce**. The endpoint is additive; nothing else changes.

Most "should the agent do X?" questions don't need a chat completion. You don't want prose,
you want a **typed decision you can branch on**: which team to route to, whether the customer
is angry, how urgent a ticket is. SynapCores exposes exactly that as a single endpoint —
`POST /v1/systemone` — backed by **d1**, a purpose-built decision model. One HTTP call in, a
structured answer out, **zero generated tokens**.

## What you get

You send a `state` (the situation) plus one or more typed `questions`. For each question you
get back a structured answer:

| Question `type` | You provide | You get back |
|---|---|---|
| `choice` | `criteria`: a map of `{option_name: description}` | the chosen option, a probability for every option, and a confidence |
| `noul`   | just `instructions` (a yes/no question) | `noul`: the probability of "yes", in `[0, 1]` |
| `score`  | `criteria`: an ordered list of levels, lowest first (2–10) | `score`: the expected (fractional) level, per-level probabilities, a legend, and a confidence |

Because it answers by weighing the options rather than writing text, a decision returns in a
single forward pass and reports `output_tokens: 0`.

## Step 1 — Install d1 (it is not bundled)

d1 is **pull-on-demand**, not shipped inside the engine image — so a fresh install does not
have it until you pull it once. Pull it with the `pull` subcommand (it streams the GGUF into
your configured `data_dir`):

```bash
# ~2.7 GiB, one-time. Point --config at the same gateway.toml your server uses,
# so the model lands in that server's data_dir.
synapcores --config /etc/synapcores/gateway.toml pull d1:latest
```

```
installed library/d1:latest (architecture=lfm2, size=2.7 GiB, digest=sha256:fe008a6acf70)
```

Confirm it is present:

```bash
synapcores --config /etc/synapcores/gateway.toml models list
```

```
NAME                       ARCH   SIZE      PULLED
library/all-minilm:latest  bert   43.8 MiB  ...
library/d1:latest          lfm2   2.7 GiB   ...
library/qwen2.5:7b         qwen2  4.4 GiB   ...
```

The pull is idempotent and resumable: re-running it when the model is already present
re-verifies the checksum and returns without re-downloading; an interrupted pull resumes.

## Step 2 — Enable it

The decision endpoint does **not** have its own model setting — it shares
`[query.ai_service].model`, which is your engine's **text-generation** model (the one
`GENERATE`, chat, and `AGENT_RUN` use). So the rule is simple:

!!! warning "Leave your generation model where it is, and name `d1:latest` per request"
    Keep `[query.ai_service].model` as your chat/generation model, and name `d1:latest`
    explicitly on every `/v1/systemone` call. That is the only setup that keeps both paths
    working — decisions on d1, generation on your chat model.

A typical config — the generation model stays put, and `auto_pull` lets the first decision
call fetch d1 on demand:

```toml
[query.ai_service]
provider  = "local"
model     = "qwen2.5:7b"   # your chat / GENERATE / AGENT_RUN model — leave it as-is
auto_pull = true           # a /v1/systemone call naming d1 pulls it on first use if missing
```

**Why not just set `model = "d1:latest"` here?** Because that key is the *generation*
default, not a decision default. Pointing it at d1 also repoints `GENERATE`, chat, and
`AGENT_RUN` at a decision model that returns **zero generated tokens** — your text generation
would stop producing text. d1 is selected **per request**, never by swapping your generation
model.

How `/v1/systemone` resolves which model to run, in order:

1. the `"model"` field in the request body — **set this to `"d1:latest"`** for a calibrated
   decision;
2. otherwise `[query.ai_service].model` — your *generation* model, which is **not** calibrated
   for decisions;
3. otherwise a built-in `d1:latest` fallback (reached only when no generation model is
   configured at all).

On any real install step 2 is your chat model, so **always name the decision model explicitly
in the request** (Step 3). No restart is needed to use a model already in the store.

!!! note "Dedicated decision instance (advanced)"
    If an engine is deployed *only* to serve `/v1/systemone` and never runs `GENERATE` /
    chat / `AGENT_RUN`, you may set `model = "d1:latest"` under `[query.ai_service]` so
    callers can omit the field. Do this **only** when you are certain no text-generation
    traffic reaches that instance — otherwise keep the per-request pattern above.

## Step 3 — Call it

Authenticate as you would any endpoint (a JWT from `POST /v1/auth/login`, or an `aidb_…` API
key), then `POST /v1/systemone`. **Always include `"model": "d1:latest"` in the request
body** — that is what routes the call to the calibrated decision model; omit it and the
endpoint falls back to your text-generation model (Step 2), which is not calibrated for
decisions:

```bash
curl -sS -X POST https://your-host/v1/systemone \
  -H "Authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' \
  -d '{
    "model": "d1:latest",
    "state": "You charged my card twice and still have not shipped my order. I need this sorted TODAY or I am disputing the charge.",
    "questions": {
      "route":   { "type": "choice", "instructions": "Which team should handle this?",
                   "criteria": { "billing": "payments/refunds", "shipping": "delivery", "technical": "bugs" } },
      "angry":   { "type": "noul",   "instructions": "Is the customer angry?" },
      "urgency": { "type": "score",  "instructions": "How urgent?",
                   "criteria": ["can wait", "this week", "today", "right now"] }
    }
  }'
```

Response (actual output from v2.5.0-ce running d1):

```json
{
  "model": "d1:latest",
  "answers": {
    "route":   { "type": "choice", "choice": "billing",
                 "probabilities": { "billing": 0.926, "shipping": 0.062, "technical": 0.012 },
                 "confidence": 0.926 },
    "angry":   { "type": "noul", "noul": 0.775 },
    "urgency": { "type": "score", "score": 2.20,
                 "legend": { "0": "can wait", "1": "this week", "2": "today", "3": "right now" },
                 "probabilities": { "0": 0.001, "1": 0.001, "2": 0.792, "3": 0.206 },
                 "confidence": 0.792 }
  },
  "usage": { "input_tokens": 188, "output_tokens": 0 }
}
```

The double-charge routes to **billing** (0.93), the tone reads **angry** (0.78), and the
urgency lands on **"today"** (0.79). You branch your workflow on `choice`, on `noul > 0.5`,
or on the `score` bucket, with the probability maps available when you want a threshold
instead of the argmax.

### Field notes

- **`state`** may be a string, or any JSON object/array (it is rendered to text for the model).
- **`choice`** — `criteria` is a map; a description of `null` is fine when the option name is
  self-explanatory. Up to **26 options** per choice in this release.
- **`score`** — `criteria` is an ordered array of 2–10 levels, **lowest first**. The returned
  `score` is the expected (fractional) level, so `2.20` means "mostly *today*, leaning toward
  *right now*."
- **`noul`** — a plain yes/no question; `noul` is the probability of "yes".

## Using a model other than d1

The endpoint is **not locked to d1**. The `model` field accepts **any causal model you have
pulled** — the decision is produced by reading the model's own preference across the option
tokens, which any such model supports. So you can pull another model and name it:

```bash
synapcores --config /etc/synapcores/gateway.toml pull <some-model>
# then POST /v1/systemone with "model": "<some-model>"
```

Two things to keep in mind:

- **d1 is the calibrated, recommended choice.** The confidence/probability values are tuned
  for d1; another model will still pick sensible answers, but its probabilities are not
  calibrated to the same scale — treat them as relative rather than absolute.
- **Options must be single tokens** for the model's tokenizer (the endpoint presents choices
  as single-letter options `A`–`Z`). d1 satisfies this; a model with an unusual tokenizer may
  not, in which case prefer d1.

## Current limits

- **Text only in this release.** Sending an `images` array returns `400`; multimodal
  decisions are planned.
- **Up to 26 options** per `choice` question; more returns `400`.
- The endpoint needs a decision-capable model in the store — if none is configured or pulled,
  the call returns an error telling you to pull one.

## Where it fits

Use `/v1/systemone` wherever an agent or pipeline needs a *decision*, not a paragraph: routing
and triage, guardrail yes/no checks, priority scoring, or any branch you'd otherwise coax out
of a chat model with brittle prompt-parsing. It pairs naturally with the in-database agent
(`AGENT_RUN`), with [claim verification (NLI)](nli.md) for gating what gets stored, and with
`GENERATE` for the steps that genuinely need prose — decide with d1, act with the rest.

## See also

- [Release notes — v2.5.0-ce](releases/v2.5.0-ce.md)
- [First-run AI setup](ai-setup.md) · [Capabilities & how to enable each](capabilities.md)
- Full walkthrough on the blog: [One-call decisions: the d1 decision model](https://synapcores.com/blog/decision-model-d1-systemone)
