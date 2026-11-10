---
name: mlpal-gateway
description: Use when calling LLMs through the MLPal AI Gateway: API keys, one base URL for the OpenAI SDK (chat, responses, models, embeddings) and the Anthropic SDK (Messages), pointing Codex CLI at MLPal, the Python SDK, model routing with model="mlpal", streaming, usage and billing in CU, errors.
---

# MLPal AI Gateway

One API key, one base URL, every model MLPal serves (OpenAI, Anthropic, Google,
open-weight models on Bedrock). The gateway resolves router tags such as
`mlpal` to a concrete model, meters every request in compute units (CU), and
answers in the shape of the SDK you called it with:

| SDK | `base_url` | paths |
|---|---|---|
| OpenAI (`openai`, any language) | `https://models.mlpal.ai/v1` | `chat.completions`, `responses`, `models`, `embeddings` |
| Anthropic (`anthropic`) | `https://models.mlpal.ai` | `messages` (Claude; `/v2/messages` for any model) |
| MLPal (`mlpal-assistants`) | default | both |

Plain HTTP works too; see section 3 for how the gateway tells the shapes apart.

## 1. Get an API key

Dashboard: **Assistants → Keys → Create key**. Or with the CLI (0.8.4), which
writes the key straight into an env file and never shows it:

```bash
uv tool install --index https://pypi.mlpal.ai/simple/ --default-index https://pypi.org/simple/ mlpal-cli
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

## 3. Chat with the OpenAI SDK, curl or httpx

`POST /v1/chat/completions` takes the OpenAI request shape (`model`,
`messages`, `max_tokens`, `tools`, `response_format`, `stream`) and answers
with OpenAI's `chat.completion` (`choices[]`, `usage`) plus one extra field,
`mlpal`, carrying what OpenAI's shape has no place for: `cost` (CU), `routing`
(the concrete model a router tag resolved to), `data` (parsed structured
output), `trace_id`. The OpenAI SDKs keep unknown fields, so nothing is lost.

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["MLPAL_API_KEY"], base_url="https://models.mlpal.ai/v1")
r = client.chat.completions.create(
    model="mlpal", max_tokens=256,
    messages=[{"role": "user", "content": "Say ok."}],
)
print(r.choices[0].message.content)                       # "Ok."
extra = r.model_extra["mlpal"]
print(extra["routing"]["resolved_model"], extra["cost"]["compute_units"])
```

The OpenAI SDKs identify themselves, so the gateway picks this shape for
them automatically. Plain HTTP clients say which shape they want with one
header: `X-MLPal-Wire: openai` for the shape above, `X-MLPal-Wire: mlpal`
for the gateway's older flat shape (`content`, `cost`, `routing` at the top
level). A request with neither header gets the flat shape today; that default
moves to the OpenAI shape, so send the header.

```bash
curl -s https://models.mlpal.ai/v1/chat/completions \
  -H "Authorization: Bearer $MLPAL_API_KEY" -H "X-MLPal-Wire: openai" \
  -H "content-type: application/json" \
  -d '{"model": "mlpal", "max_tokens": 256,
       "messages": [{"role": "user", "content": "Say ok."}]}'
```

```python
import os, httpx

r = httpx.post(
    "https://models.mlpal.ai/v1/chat/completions",
    headers={"Authorization": f"Bearer {os.environ['MLPAL_API_KEY']}", "X-MLPal-Wire": "openai"},
    json={"model": "mlpal", "max_tokens": 256,
          "messages": [{"role": "user", "content": "Say ok."}]},
    timeout=120,
)
r.raise_for_status()
body = r.json()
print(body["choices"][0]["message"]["content"])   # "Ok."
print(body["mlpal"]["routing"]["resolved_model"])  # the concrete model that ran
print(body["mlpal"]["cost"]["compute_units"])      # CU charged for this call
```

Note: `max_tokens` must be at least 16 for OpenAI models. `mlpal-lite` routes to a reasoning
model that may spend tokens thinking before it answers: give it 256 or more. Non-streaming calls
are cut off with a `504 gateway_timeout` after 110 s; stream long generations.

## 3a. The Responses API (`POST /v1/responses`) and Codex CLI

The OpenAI SDKs' `responses.create`, `responses.stream` and `responses.parse`
work against the same `base_url`, for every model the gateway serves. The
response is OpenAI's `response` object plus the `mlpal` extra.

```python
import os
from openai import OpenAI

client = OpenAI(api_key=os.environ["MLPAL_API_KEY"], base_url="https://models.mlpal.ai/v1")
r = client.responses.create(model="mlpal-lite", input="Say ok.", max_output_tokens=256)
print(r.output_text, r.model_extra["mlpal"]["cost"]["compute_units"])

with client.responses.stream(model="claude-haiku-5-5", input="Say ok.", max_output_tokens=256) as s:
    for event in s:
        if event.type == "response.output_text.delta":
            print(event.delta, end="", flush=True)
    final = s.get_final_response()   # usage + mlpal extra on response.completed
```

Supported: `input` as text or items (`message`, `function_call`,
`function_call_output`, `input_image` by URL or data URL, `input_file` by
inline `file_data`), `instructions`, function `tools` and `tool_choice`,
`text.format` (`json_schema` / `json_object`; `responses.parse` with a
Pydantic model works), `reasoning.effort`, `max_output_tokens`,
`temperature`, `top_p`, `metadata`, `stream`.

Stateless: the gateway stores nothing (and sends `store: false` upstream).
`previous_response_id`, `store: true`, `background`, `conversation` and
`GET /v1/responses/{id}` are refused with a typed 400 — send the full
`input` each turn. Hosted tools (`web_search`, `file_search`,
`code_interpreter`, `image_generation`, `mcp`, …) are not executed: they are
dropped and listed in `mlpal.metadata.dropped`; function tools run as usual.
A tool output that is only an image becomes the text
"[image omitted: the gateway does not yet pass images in tool results]" so
the conversation continues. Every client input error is a 400 with
`error.param` naming the field, never a 5xx.

**Codex CLI.** Point Codex at the gateway with a provider block in its
`config.toml` (`~/.codex/config.toml`, or a separate `CODEX_HOME`):

```toml
model = "claude-haiku-5-5"
model_provider = "mlpal"

[model_providers.mlpal]
name = "MLPal gateway"
base_url = "https://models.mlpal.ai/v1"
env_key = "MLPAL_API_KEY"
wire_api = "responses"
```

```bash
export MLPAL_API_KEY=mlpal_sk_...
codex exec "Read names.txt with a shell command and count its lines."
```

Verified with Codex 0.144.5: shell commands through Codex's function tools,
multi-turn with `codex exec resume --last`, on `claude-haiku-5-5`. For now
use a non-GPT model name (`claude-*`, `gemini-*`, `mlpal-lite`); Codex
prints "Model metadata for `<model>` not found. Defaulting to fallback
metadata" for them, which is harmless. Codex's `view_image` tool returns an
image the gateway cannot yet pass on: the model is told the image was
omitted and the session continues. Codex's `web_search` is dropped the same
way. Sub-agent tools (Codex's `multi_agent_v1` namespace) work.

**Reasoning state across turns (OpenAI reasoning models).** Ask for it with
`include=["reasoning.encrypted_content"]` together with a `reasoning.effort`
(verified on `gpt-5.4-mini` with `effort: "low"`): the response's `output`
then starts with a `reasoning` item whose `encrypted_content` is the
gateway's envelope `mlpal1.<backend>.<provider item id>.<blob>`. Replay the
item exactly as the SDK returns it before the assistant turn it belongs to
on the next call; the gateway unwraps it, forwards it to the backend that
produced it, and the model continues its reasoning. Rules worth knowing:

- Only OpenAI models. On any other model a replayed item with
  `encrypted_content` is a 400 `reasoning_state_unsupported`; the same for
  `include` on a non-OpenAI model, and on tenant connections (BYOK/BYOM).
- Replay only what this gateway produced. An encrypted blob minted by OpenAI
  directly (no envelope) is a 400 `invalid_request`; an envelope naming a
  backend the model is not served on, or two backends in one input, is a
  409 `reasoning_state_conflict`. None of these is retried by the SDK; every
  message says: resend the input without the reasoning items to continue.
- With an active `reasoning.effort`, OpenAI rejects `temperature` and
  `top_p` on gpt-5.x, so the gateway omits them; if you set them explicitly,
  `mlpal.metadata.sampling_ignored` says so.
- Items with only a `summary` (what Codex CLI replays, since it never asks
  for encrypted content) are accepted and dropped; Codex is unaffected.

```python
r1 = client.responses.create(model="gpt-5.4-mini", input="What is 17*23?",
                             include=["reasoning.encrypted_content"],
                             reasoning={"effort": "low"}, max_output_tokens=256)
items = [o.model_dump(exclude_none=True) for o in r1.output]   # reasoning item first
r2 = client.responses.create(model="gpt-5.4-mini",
                             include=["reasoning.encrypted_content"],
                             reasoning={"effort": "low"}, max_output_tokens=256,
                             input=[{"role": "user", "content": "What is 17*23?"}, *items,
                                    {"role": "user", "content": "Now add 1 to that."}])
```

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
pip install --index-url https://pypi.mlpal.ai/simple/ --extra-index-url https://pypi.org/simple/ "mlpal-assistants>=0.4.0"
# with uv (its extra index would outrank the MLPal index and pick an older PyPI release):
uv pip install --index https://pypi.mlpal.ai/simple/ "mlpal-assistants>=0.4.0"
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

`client.messages` speaks the Anthropic Messages shape to every model. The
`Assistant` facade speaks the OpenAI shape and returns typed `cost` and
`routing`; use 0.4.0 or later for it (0.3.x `stream_chat` returned nothing
against the gateway):

```python
import asyncio
from mlpal_assistants import Assistant

async def main():
    a = Assistant()  # reads MLPAL_API_KEY
    r = await a.chat("Say ok.", model="mlpal-lite", max_tokens=256)
    print(r.content, r.routing.resolved_model, r.cost.compute_units)
    async with a.stream_chat([{"role": "user", "content": "Say ok."}], model="mlpal-lite", max_tokens=256) as s:
        async for d in s:
            print(d.content or "", end="", flush=True)
            if d.cost:              # the final chunk carries the CU
                print("\n", d.cost.compute_units)
    await a.close()

asyncio.run(main())
```

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
| `mlpal-flash` | fastest | `gemini-3.8-flash` |
| `mlpal-lite` | lowest cost | `gpt-6-luna` |

Any concrete `model_tag` works too, for example `claude-haiku-5-5`,
`text-embedding-3-small`.

## 7. Streaming

With the OpenAI SDK, `stream=True` on `chat.completions.create` works as it
does against OpenAI: `chat.completion.chunk` events, then `[DONE]`. Pass
`stream_options={"include_usage": True}` to get the trailing usage chunk;
without it, `usage` and the `mlpal` extra ride on the final chunk.

```python
with client.chat.completions.create(model="claude-haiku-5-5", max_tokens=256, stream=True,
                                    messages=[{"role": "user", "content": "Say ok."}]) as s:
    for chunk in s:
        if chunk.choices and chunk.choices[0].delta.content:
            print(chunk.choices[0].delta.content, end="", flush=True)
```

Plain HTTP clients send `stream: true` with `X-MLPal-Wire: openai` for the same
events, or use the gateway's own stream path, unchanged:

```bash
curl -N https://models.mlpal.ai/v1/chat/completions/stream \
  -H "Authorization: Bearer $MLPAL_API_KEY" -H "content-type: application/json" \
  -d '{"model": "claude-haiku-5-5", "max_tokens": 256,
       "messages": [{"role": "user", "content": "Say ok."}]}'
```

```
data: {"content": "ok", "done": false}
data: {"content": "", "done": true, "finish_reason": "stop", "cost": {...}}
data: [DONE]
```

Streams send a `: ping` comment every 15 s while a model thinks; ignore it.
For Claude models the Anthropic SDK's `messages.stream` (section 4) also
works unchanged.

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

Usage is deducted from your prepaid wallet a few minutes after each request,
but it counts against your available balance straight away. When the available
balance runs out, requests return `402 wallet_empty` until you top up; nothing
is queued or retried for you.

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
| 413 | request too large | the body is over 128 MiB | send files by URL or trim inline media |
| 504 | `gateway_timeout` | a non-streaming call ran past 110 s and was cancelled, nothing billed | stream the call, or retry with a smaller request |

On the OpenAI shape the same errors arrive in OpenAI's envelope
(`{"error": {"message", "type", "code"}}`) with the same `code`, so the SDK
raises its usual `AuthenticationError`, `RateLimitError` and so on.

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
