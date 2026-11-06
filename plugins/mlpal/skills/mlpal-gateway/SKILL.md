---
name: mlpal-gateway
description: Use when calling LLMs through the MLPal AI Gateway: API keys, the chat and Anthropic-compatible Messages endpoints, OpenAI-compatible embeddings, the Python SDK, model routing with model="mlpal", streaming, embeddings, usage and billing in CU, errors.
---

# MLPal AI Gateway

One API key, one base URL, every model MLPal serves (OpenAI, Anthropic, Google,
open-weight models on Bedrock). The gateway resolves router tags such as
`mlpal` to a concrete model, meters every request in compute units (CU), and
returns the response in the shape of the wire you called.

Base URL: `https://models.mlpal.ai`

## 1. Get an API key

Dashboard: **Assistants → Keys → Create key**. Or with the CLI (0.8.4), which
writes the key straight into an env file and never shows it:

```bash
uv tool install mlpal-cli --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/
mlpal assistants keys create --name my-agent --env-file .env --env-var MLPAL_API_KEY
```

Load the key from the environment in every
snippet below: `export MLPAL_API_KEY=…` (or source `.env`). Never paste it
into code, prompts, logs or commits.

## 2. Authentication

Every request carries the key as a bearer token. The Anthropic-wire endpoint
also accepts it in `x-api-key`.

```
Authorization: Bearer $MLPAL_API_KEY
```

## 3. Chat with the OpenAI-style wire (curl and Python)

`POST /v1/chat/completions` takes the OpenAI request shape (`model`,
`messages`, `max_tokens`, `tools`, `response_format`). The response is the
gateway's own flat shape: `content`, `role`, `finish_reason`, `tool_calls`,
`cost`, `routing`. It is **not** `choices[]`, so read it with plain HTTP rather
than the OpenAI SDK's chat client.

```bash
curl -s https://models.mlpal.ai/v1/chat/completions \
  -H "Authorization: Bearer $MLPAL_API_KEY" \
  -H "content-type: application/json" \
  -d '{"model": "mlpal", "max_tokens": 256,
       "messages": [{"role": "user", "content": "Say ok."}]}'
```

```python
import os, httpx

r = httpx.post(
    "https://models.mlpal.ai/v1/chat/completions",
    headers={"Authorization": f"Bearer {os.environ['MLPAL_API_KEY']}"},
    json={"model": "mlpal", "max_tokens": 256,
          "messages": [{"role": "user", "content": "Say ok."}]},
    timeout=120,
)
r.raise_for_status()
body = r.json()
print(body["content"])                      # "Ok."
print(body["routing"]["resolved_model"])    # the concrete model that ran
print(body["cost"]["compute_units"])        # CU charged for this call
```

Note: `max_tokens` must be at least 16 for OpenAI models.

## 4. Anthropic Messages wire

`POST /v1/messages` is a drop-in for the Anthropic SDK with Claude models:
set `base_url` to the gateway and use the MLPal key.

```python
import os
from anthropic import Anthropic

client = Anthropic(api_key=os.environ["MLPAL_API_KEY"],
                   base_url="https://models.mlpal.ai")
msg = client.messages.create(model="claude-haiku-5-5", max_tokens=256,
                             messages=[{"role": "user", "content": "Say ok."}])
print(msg.content[0].text)

with client.messages.stream(model="claude-haiku-5-5", max_tokens=256,
                            messages=[{"role": "user", "content": "Say ok."}]) as s:
    for text in s.text_stream:
        print(text, end="", flush=True)
```

`POST /v2/messages` serves the same Anthropic request/response shape for
**any** model, including router tags (`"model": "mlpal-lite"`).

## 5. The official Python SDK (`mlpal-assistants`)

```bash
pip install --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/ mlpal-assistants
```

```python
from mlpal_assistants import MLPal

client = MLPal()  # reads MLPAL_API_KEY; base_url defaults to models.mlpal.ai
msg = client.messages.create(
    model="mlpal",
    max_tokens=256,
    messages=[{"role": "user", "content": "Say ok."}],
)
print(msg.text)            # the reply
print(msg.model)           # the concrete model that ran
print(msg.compute_units)   # CU charged (pass-through cost, no markup)
```

The SDK speaks the Anthropic Messages shape to every model, so one code path
covers OpenAI, Google and open-weight models as well as Claude.

## 6. Models and routing

```bash
curl -s https://models.mlpal.ai/v1/models -H "Authorization: Bearer $MLPAL_API_KEY"
mlpal assistants models list
```

The list is filtered to what your key may use. Each entry carries `model_tag`
(the id to send), `provider`, `pricing_tier`, `context_length`,
`max_output_tokens`, `capabilities` and `is_deprecated`.

Router tags pick a model for you and record the choice in `routing`:

| tag | strategy | resolves to today |
|---|---|---|
| `mlpal` | best quality | `claude-opus-5-5` |
| `mlpal-lite` | lowest cost | `gpt-5-nano` |

Any concrete `model_tag` works too, for example `claude-haiku-5-5`,
`text-embedding-3-small`.

## 7. Streaming

The OpenAI-style wire streams on a separate path. The `stream: true` flag on
`/v1/chat/completions` is ignored and returns one JSON body.

```bash
curl -N https://models.mlpal.ai/v1/chat/completions/stream \
  -H "Authorization: Bearer $MLPAL_API_KEY" -H "content-type: application/json" \
  -d '{"model": "claude-haiku-5-5", "max_tokens": 256,
       "messages": [{"role": "user", "content": "Say ok."}]}'
```

Server-sent events, one JSON object per `data:` line:

```
data: {"content": "ok", "done": false}
data: {"content": "", "done": true, "finish_reason": "stop", "cost": {...}}
data: [DONE]
```

Concatenate `content` while `done` is false; the final event carries the
`cost`. For Claude models the Anthropic SDK's `messages.stream` (section 4)
also works unchanged.

## 8. Embeddings

`POST /v1/embeddings` takes and returns the OpenAI shape, so the OpenAI SDK's
embeddings client works as-is.

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["MLPAL_API_KEY"], base_url="https://models.mlpal.ai/v1")
vec = client.embeddings.create(model="text-embedding-3-small", input="hello").data[0].embedding
print(len(vec))  # 1536
```

Embedding models served today: `text-embedding-3-small`,
`text-embedding-3-large`, `gemini-embedding-001`, `gemini-embedding-2`,
`amazon.titan-embed-text-v2:0`.

## 9. Usage and cost

Every non-streaming response and every final stream event reports
`cost.compute_units`: the model's pass-through cost, metered by the gateway
with no markup. Account totals for the current month:

```bash
curl -s https://models.mlpal.ai/v1/usage/summary -H "Authorization: Bearer $MLPAL_API_KEY"
mlpal assistants usage            # this month
mlpal assistants usage --days 7   # per-day breakdown
```

The summary returns `total_requests`, `total_input_tokens`,
`total_output_tokens`, `total_compute_units`, `quota_limit`,
`quota_remaining` and a `by_model` breakdown. The dashboard shows the same
under **Assistants → Usage**.

## 10. Errors

Errors are JSON with a stable `code` and a human `message`:

| HTTP | `code` | meaning | what to do |
|---|---|---|---|
| 401 | `invalid_api_key` | key missing, malformed, revoked or expired | create a new key (section 1); check the env var is set |
| 402 | `wallet_empty` | the account's balance is used up | top up the wallet in the dashboard, then retry |
| 403 | `risk_hold` | the account is on hold (for example until a card is verified) | the `message` says what is required; do not retry blindly |
| 403 | `account_suspended` | the account is suspended | contact support |
| 403 | `model_access_denied` | this key's model policy excludes the model | pick an allowed model from `/v1/models` or adjust the key's policy |
| 429 | rate limit | too many requests | back off exponentially and retry |
| 400 | `url_rejected` | a URL in the request (file, image) was refused | use an allowed, public URL |

Deprecated models stay listed with `is_deprecated: true` and a
`deprecation_message`; move to the replacement before the date it names.

## 11. Security

- Read the key from `MLPAL_API_KEY`; never print it, never commit it, never
  put it in a prompt.
- Use `--env-file` when minting with the CLI so the secret never crosses the
  terminal.
- One key per application or agent, named after it, so it can be revoked
  alone: `mlpal assistants keys revoke`.
- Revoked and expired keys stop working immediately (401).
