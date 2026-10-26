# Natural Language Inference (NLI)

> Verified against **v2.1.2-ce**. Every request and response on this page was
> executed against a running v2.1.2-ce gateway; outputs are real, truncated
> only where marked.

SynapCores ships an in-process NLI cross-encoder for **claim verification**:
given a *statement* and one or more pieces of trusted evidence (*truth
anchors*), it answers `TRUTH`, `LIE`, or `UNVERIFIED_PLAUSIBLE`. The flagship
use is hallucination-gating: verify an LLM or agent answer against the
retrieval set it was built from, before you store or display it.

The same pinned model serves three surfaces — REST endpoints, SQL functions
(v2.1.2+), and the SQL result feeds your application — with no sidecar, no
network hop, and no external API.

- **Model:** `cross-encoder/nli-deberta-v3-small` (DeBERTa-v3), FP32 ONNX,
  pinned by SHA-256 at install and verified at load. Labels:
  `contradiction` / `entailment` / `neutral`; 512-token window.
- **Performance (v2.1.2+, measured on a 6-core i5-10400F, CPU):** ~15 ms per
  warm single-anchor verify; 16 anchors ≈ 192 ms; 32 anchors ≈ 335 ms.
  Engines before v2.1.2 cost ~0.37 s per pair — same scores, 26–33× slower.

---

## Setup

NLI is **off until a model bundle is installed and the gateway restarts with
it present**. The runtime loads models from a process-start snapshot: pulling
a bundle while the gateway runs does not make it servable.

### Docker

```bash
docker run -d -p 8080:8080 \
  -v synapcores-data:/var/lib/synapcores \
  -e AIDB_ACCEPT_LICENSE=1 \
  -e AIDB_NLI_ENABLED=1 \
  synapcores/community:latest
```

The entrypoint pulls `nli-default` (~576 MB, one time, SHA-pinned) **before**
starting the server when `AIDB_NLI_ENABLED=1`. Nothing else to do.

### Installer (`install-ce.sh`) — the opt-in flag

NLI is an **explicit, independent opt-in** at install time; it does not depend
on the chat-model hardware tier, and the installer leaves it off unless you ask
for it. Pass `--with-nli` (interactive installs also prompt with a y/N), and
optionally pick the smaller bundle:

```bash
./install-ce.sh --with-nli                           # pulls nli-default (~577 MB) and enables it
./install-ce.sh --with-nli --nli-model nli-xsmall    # smaller footprint (~284 MB)
```

For unattended installs, set the environment instead of the flag:

```bash
SYNAPCORES_WITH_NLI=1 SYNAPCORES_NLI_MODEL=nli-default ./install-ce.sh
```

Either way the installer pulls the bundle **and** persists enablement into the
gateway config (it runs `models enable-nli` for you), so NLI warms on the next
engine start with no `AIDB_NLI_ENABLED` env var to set. Verify afterwards:

```bash
synapcores --config <cfg> models status nli-default        # disk-side bundle state
# then, with the engine running:
curl -s http://localhost:8080/v1/ai/nli/models -H "Authorization: Bearer $KEY"
```

### Bare metal (existing install or plain tarball)

If NLI was not selected at install time (or you are on a hand-extracted
tarball), pull the bundle **before** the gateway starts, persist enablement,
then start (or restart) — all with the same `--config` the service uses:

```bash
synapcores --config /etc/synapcores/gateway.toml pull nli-default
# installed NLI bundle: {"alias":"nli-default","id":"nli-deberta-v3-small-fp32", ...}
synapcores --config /etc/synapcores/gateway.toml models enable-nli nli-default
systemctl restart synapcores
```

`models enable-nli` writes the `[nli]` section into the config so enablement
survives restarts. Alternatively, enable per process via environment
(`AIDB_NLI_ENABLED=1`, optional `AIDB_NLI_MODEL=nli-default`) — handy for Docker
(above) and ephemeral runs. Check disk-side state any time with
`synapcores --config <cfg> models status nli-default`; **runtime** readiness is
only ever reported by `GET /v1/ai/nli/models`.

!!! warning "Upgrade the whole folder"
    The engine loads ONNX Runtime from `lib/` **beside the executable**.
    Copying only the binary during an upgrade breaks NLI model loads; since
    v2.1.2 the error names the three searched paths. Upgrade the whole
    extracted folder, or set `ORT_DYLIB_PATH`.

### Model catalog

| Alias | Upstream (pinned revision) | Download | Notes |
|---|---|---|---|
| `nli-default`, `nli-small` | `cross-encoder/nli-deberta-v3-small` @ `fa28048` | ~576 MB | the two aliases share one artifact |
| `nli-xsmall` | `cross-encoder/nli-deberta-v3-xsmall` @ `a150876` | ~284 MB | smaller footprint for tight memory |

All bundles are Apache-2.0, immutable, and content-verified: the installer
checks SHA-256 on download, and the loader re-verifies before serving. A new
alias becomes servable only after a gateway restart.

---

## REST API

Three endpoints, all under the authenticated AI router (JWT or
`aidb_…` API key as `Authorization: Bearer …`; standard body-size and rate
limits apply). All success responses use the gateway envelope
`{"data": …, "meta": …}`; errors use `{"error": {"code", "message"}, "meta"}`.

| Purpose | Endpoint | Method |
|---|---|---|
| Runtime model state | `/v1/ai/nli/models` | `GET` |
| Verify a statement against anchors | `/v1/ai/nli/verify` | `POST` |
| Score premise/hypothesis pairs | `/v1/ai/nli/pairs` | `POST` |

Both POST bodies reject unknown fields (`deny_unknown_fields`) — a misspelled
key is a 400, never silently ignored.

### `GET /v1/ai/nli/models` — readiness and identity

```bash
curl -s http://localhost:8080/v1/ai/nli/models \
  -H "Authorization: Bearer $KEY"
```

```json
{
  "data": {
    "enabled": true,
    "default_model": "nli-default",
    "restart_required_for_new_installs": true,
    "models": [
      {
        "model": "nli-default",
        "state": "ready",
        "identity": {
          "alias": "nli-default",
          "id": "nli-deberta-v3-small-fp32",
          "repository": "cross-encoder/nli-deberta-v3-small",
          "revision": "fa2804872c3b4bd748f38c0185cc85775361e735",
          "license": "Apache-2.0",
          "bundle_sha256": "3d809ec29d1a31af5db11075598a7c89ef4ba42fd29daad6ca80d25d7373d768",
          "labels": ["contradiction", "entailment", "neutral"],
          "max_length": 512
        },
        "loaded_bundle_sha256": "3d809ec2…"
      }
    ]
  }
}
```
*(trimmed: `identity` also reports exact byte counts and free-space guidance;
a ready model adds an `execution` capability profile.)*

`state` is one of `ready`, `missing`, `failed`, or an intermediate load
state. Poll this after install/restart; serve traffic only on `ready`.

### `POST /v1/ai/nli/verify` — the verification verdict

Request fields:

| Field | Type | Required | Default | Notes |
|---|---|---|---|---|
| `statement` | string | yes | — | the claim to check, ≤ 16,384 UTF-8 bytes |
| `truth_anchors` | string[] | yes | — | 1–32 trusted evidence texts, each ≤ 16,384 bytes. **The field is `truth_anchors` — `anchors` is rejected.** |
| `model` | string | no | `nli-default` | an installed alias |
| `closed_world` | bool | no | `false` | `true`: anything the anchors don't entail is a `LIE` |
| `contradiction_threshold` | number | no | `0.6` | score needed to call a contradiction |
| `entailment_threshold` | number | no | `0.6` | score needed to call an entailment |

```bash
curl -s http://localhost:8080/v1/ai/nli/verify \
  -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{
    "statement": "The rollout completed without failures.",
    "truth_anchors": [
      "The deployment finished at 14:02 with zero failed checks.",
      "All 46 post-deploy probes passed."
    ]
  }'
```

```json
{
  "data": {
    "model": "nli-default",
    "bundle_sha256": "3d809ec2…",
    "verdict": "TRUTH",
    "reason": "An anchor reaches the entailment threshold and no anchor reaches the contradiction threshold.",
    "confidence": 0.9977391796557792,
    "violated_anchor": null,
    "matched_anchor": "The deployment finished at 14:02 with zero failed checks.",
    "all_scores": [
      [0.0001135, 0.9977392, 0.0021473],
      [0.0023965, 0.9266143, 0.0709892]
    ]
  }
}
```

`all_scores` is one `[contradiction, entailment, neutral]` triple per anchor,
in request order. The statement is checked against **each anchor
independently** — two anchors are never chained into a multi-step proof.

**The three verdicts, observed:**

| Verdict | When | Observed `reason` |
|---|---|---|
| `TRUTH` | some anchor entails the statement, none contradicts it | "An anchor reaches the entailment threshold and no anchor reaches the contradiction threshold." |
| `LIE` | some anchor contradicts it — **contradiction always wins over entailment** | "An anchor reaches the contradiction threshold; contradiction takes precedence over entailment." |
| `UNVERIFIED_PLAUSIBLE` | no anchor reaches either threshold (open world) | "No anchor reaches either threshold; open-world policy leaves the statement unverified." |

With `"closed_world": true` the third case becomes a `LIE`:

```json
{
  "verdict": "LIE",
  "reason": "Closed-world policy rejects a statement without qualifying entailment; no anchor reaches the contradiction threshold.",
  "confidence": 0.9994635870868093
}
```

Use closed-world when the anchors are a complete allow-list (a policy, a
spec); use the open-world default when they are merely what you happened to
retrieve.

### `POST /v1/ai/nli/pairs` — raw classification scores

For when you want probabilities, not a verdict — ranking, custom thresholds,
batch screening. 1–32 pairs per call.

```bash
curl -s http://localhost:8080/v1/ai/nli/pairs \
  -H "Authorization: Bearer $KEY" -H 'content-type: application/json' \
  -d '{
    "pairs": [
      {"premise": "The invoice was paid on March 3.", "hypothesis": "The invoice is unpaid."},
      {"premise": "The invoice was paid on March 3.", "hypothesis": "Payment was received in March."}
    ]
  }'
```

```json
{
  "data": {
    "model": "nli-default",
    "bundle_sha256": "3d809ec2…",
    "scores": [
      {"contradiction": 0.9994857, "entailment": 0.0003246, "neutral": 0.0001897},
      {"contradiction": 0.0001434, "entailment": 0.9962150, "neutral": 0.0036416}
    ]
  }
}
```

The `premise` is the evidence, the `hypothesis` is the claim — direction
matters to an NLI model.

### Error reference (all observed)

| HTTP | `code` | Trigger | Observed message |
|---|---|---|---|
| 400 | `invalid_nli_request` | unknown/misspelled field (e.g. `anchors` instead of `truth_anchors`) | "Invalid NLI verification JSON request" |
| 400 | `invalid_nli_request` | 0 or > 32 anchors | "Invalid argument: NLI verification requires between 1 and 32 anchors" |
| 400 | `invalid_nli_request` | a text over 16,384 UTF-8 bytes | names the offending statement/anchor |
| 404 | `nli_model_not_found` | unknown alias, or pulled after start | "The NLI model alias is unknown; newly installed aliases require a gateway restart" |
| 503 | `nli_disabled` | NLI not enabled | "NLI is disabled; install a model and enable [nli] or AIDB_NLI_ENABLED" |
| 503 | `nli_model_not_installed` | enabled but bundle absent at process start | "…pull or import it and restart the gateway" |
| 503 | `nli_model_unavailable` | bundle failed integrity/load | "…reinstall and restart the gateway" |
| 401 | `missing_claims` | no/invalid auth | "Authentication required" |

Model aliases are 1–128 ASCII letters, digits, `_` or `-`, starting with a
letter or digit.

---

## SQL functions (v2.1.2-ce+)

The same model, callable inside queries — see the
[SQL reference](sql-reference.md) for full semantics. NLI results are **never
served from the SQL result cache**.

**`NLI_CLASSIFY(model, premise, hypothesis)` → JSON** — one pair's scores:

```sql
SELECT NLI_CLASSIFY('nli-default',
  'The invoice was paid on March 3.',
  'The invoice is unpaid.') AS scores;
-- {"model":"nli-default","bundle_sha256":"3d809ec2…",
--  "scores":{"contradiction":0.99949,"entailment":0.00032,"neutral":0.00019}}
```

**`NLI_VERIFY(model, statement, anchors_json [, options_json])` → JSON** —
the same verdict object as the REST endpoint. Options keys: `closed_world`,
`contradiction_threshold`, `entailment_threshold`.

```sql
SELECT NLI_VERIFY('nli-default',
  'The rollout completed without failures.',
  JSON_ARRAY('The deployment finished at 14:02 with zero failed checks.',
             'All 46 post-deploy probes passed.'),
  JSON_OBJECT('closed_world', true)) AS v;
-- {"verdict":"TRUTH","confidence":0.99774,
--  "matched_anchor":"The deployment finished at 14:02 with zero failed checks.", …}
```

**`NLI_ANCHORS(text)`** — an aggregate that collects up to 32 rows into an
anchors array, built for feeding `NLI_VERIFY` from evidence rows:

```sql
CREATE TABLE IF NOT EXISTS run_evidence (id INT PRIMARY KEY, run_id INT, fact TEXT);
INSERT INTO run_evidence VALUES
  (1, 7, 'The deployment finished at 14:02 with zero failed checks.'),
  (2, 7, 'All 46 post-deploy probes passed.'),
  (3, 7, 'Rollback was not required.');

SELECT NLI_VERIFY('nli-default',
  'The rollout completed without failures.',
  NLI_ANCHORS(fact)) AS verdict
FROM run_evidence WHERE run_id = 7;
-- {"verdict":"TRUTH","confidence":0.99774, …}
```

Like the other AI functions, call it through an alias rather than directly in
`ORDER BY` (sort by the alias).

---

## The guardrail pattern

Gate what an agent stores or shows: verify the answer against the retrieval
set it was built from, in one statement.

```sql
-- store the answer only if the evidence entails it
INSERT INTO approved_answers (run_id, answer, verdict)
SELECT 7, $answer,
       NLI_VERIFY('nli-default', $answer, NLI_ANCHORS(fact))
FROM run_evidence
WHERE run_id = 7
  AND JSON_EXTRACT(NLI_VERIFY('nli-default', $answer, NLI_ANCHORS(fact)),
                   '$.verdict') = 'TRUTH';
```

Over REST the same shape is: retrieve evidence → `POST /v1/ai/nli/verify`
with the evidence as `truth_anchors` → act on `verdict`.

## Honest boundaries

- The model evaluates **the text you supply**. It does not retrieve facts,
  resolve pronouns across anchors, or compare numbers and dates with typed
  precision — use exact SQL comparisons for those.
- Anchors are checked independently; there is no multi-anchor chaining.
- **Contradiction precedence** means one contradicting anchor outvotes any
  number of entailing ones — curate anchor sets, don't dump whole documents.
- Scores describe model behavior over your text and thresholds. Calibrate the
  thresholds on your own data before gating anything irreversible.
- Limits: 32 anchors / 32 pairs per call, 16,384 UTF-8 bytes per text,
  512-token model window (longer inputs are truncated by the tokenizer).

## See also

- [SQL reference — NLI functions](sql-reference.md) · [Release notes v2.1.2-ce](releases/v2.1.2-ce.md)
- Worked recipes: [Semantic claim verification](https://synapcores.com/recipes/ml/020_semantic_claim_verification_nli) and the eight companion NLI recipes on the recipe catalog
