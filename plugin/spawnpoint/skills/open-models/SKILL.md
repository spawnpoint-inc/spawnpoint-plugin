---
name: open-models
description: Run an app's OpenAI or Anthropic calls on open models through spawnpoint, with no provider key. Use when an app about to be deployed to spawnpoint calls a model provider (the openai or anthropic SDK, the Vercel AI SDK, LangChain, or api.openai.com / api.anthropic.com over HTTP), or when the user asks for open models, a free model, or an app with no API key.
---

# Open models on spawnpoint

spawnpoint can run an app's chat calls on open models (gpt-oss, Gemma, and Mistral) and
its embeddings on `cohere-embed-v3`, through its own endpoint. The user needs no provider
account and no key: the deploy mints the project's key and ships it as env. The user
chooses, once, per project.

## 1. Detect

Before `deploy_project`, look at the bundle for a provider call:

| Signal | Means |
|---|---|
| `openai` in `package.json` or `requirements.txt`; `new OpenAI(`, `from openai import` | OpenAI SDK: chat, embeddings, or both |
| `@anthropic-ai/sdk` or `anthropic`; `new Anthropic(`, `from anthropic import` | Anthropic SDK, Messages format |
| `ai` with `@ai-sdk/openai` or `@ai-sdk/anthropic` | Vercel AI SDK over one of the two |
| `langchain_openai`, `langchain_anthropic`, `@langchain/openai`, `@langchain/anthropic` | LangChain over one of the two |
| `api.openai.com` or `api.anthropic.com` in source | Raw HTTP to a provider |
| `OPENAI_API_KEY` or `ANTHROPIC_API_KEY` read from env or in `.env` | The key the switch removes |

Note what the calls do: chat, tool calling, embeddings, images in the messages, image
generation, audio, structured JSON. The choice depends on it.

## 2. Ask, once

One question, before the deploy, naming what was found and the model you would pick,
with keeping the provider as a plain option:

> This app calls OpenAI for chat and embeddings. spawnpoint can run both on open models
> with no OpenAI key: chat on gpt-oss-120b, embeddings on cohere-embed-v3. The requests
> would go through spawnpoint to Amazon Bedrock in AWS's Oregon region, which is set to
> keep none of them; the companies that make the models cannot see them, and nothing is
> used for training. Switch, or keep OpenAI?

Never switch without the answer. Never ask twice for one project: a project that
already exists without an `ai` block on `get_project` was deployed on the provider on
purpose, so redeploy it as it is unless the user raises it. When the user already said
which they want (in the request, or earlier in the conversation), skip the question.
An app you are writing from scratch that needs a model: use the open model with no
question, since there is no provider to keep.

Every ask names where requests go, as above. When the user asks for more, or when the
app would send other people's personal data or confidential material (customer records,
call transcripts, mailbox contents) to the model, give the rest: each request, chat and
embeddings alike, goes through spawnpoint to Amazon Bedrock in AWS's us-west-2 region
(Oregon), and no other region. spawnpoint's AWS account is set to zero data retention for
Bedrock, so AWS keeps no requests or answers, does not use them to train models, and does
not share them with the companies that make the models. spawnpoint stores no prompts,
answers, texts, or vectors either, only a usage record per call (project, model, tokens,
cost, time). The chat models are made by OpenAI, Google, and Mistral AI; the embeddings
model by Cohere. Say this plainly before switching such an app, and offer keeping the
provider, with its own key and retention terms, as the other choice.
https://getspawnpoint.com/how-apps-run.html has the details.

## 3. Choose

| The app does | Model | Pass on |
|---|---|---|
| Chat, assistants, summarizing, tool calling | `gpt-oss-120b` (OpenAI) | `max_tokens` 300 or more: it reasons before it answers, and a small budget returns empty content. The reasoning is in `message.reasoning`; `message.content` holds only the answer |
| High volume or latency-sensitive chat | `gpt-oss-20b` (OpenAI) | Same budget rule |
| Chat in many languages, with no images | `mistral-large-3` (Mistral AI) | Does not read images: an image sent to it is refused |
| Anthropic Messages format (the Anthropic SDK, kept as is) | `gpt-oss-120b` | Read the block whose `type` is `text`, not `content[0]`: a `thinking` block comes first when the request asks for thinking |
| Images in the conversation (describe, read, compare, receipts, scans) | `gemma-4-31b` (Google) | Data URIs work; `gemma-4-26b` is the faster second choice |
| Embeddings, search, RAG | `cohere-embed-v3` (Cohere Embed Multilingual v3) | 1024 dimensions. Up to 96 texts a call, about 512 tokens each; longer text is cut at the end, so split long documents into chunks. spawnpoint serves about 20 embedding calls a minute in all: send texts in arrays, never one call per text, and retry a 429 after its `Retry-After`. A vector column sized for another model (1536 for `text-embedding-3-small`) becomes 1024, and vectors already stored from another model must be embedded again: vectors from two models do not compare |
| Structured JSON output | any chat model | No `response_format`: ask for JSON in the prompt, parse, retry once on failure |
| Sorting text into your own categories, scoring it, or a yes/no (routing, moderation, triage, tagging) | `gpt-oss-20b` | JSON by prompt, as above: name the allowed values in the prompt and check the answer is one of them |
| Image generation, video, audio, speech | keep the provider | Not on the open catalog: say so, keep that key in `env`, and move on |

When the user's policy rules out some models (by maker or by country), choose among the
rest and say which one you used; when none fits, keep the provider.

A team can approve which models its apps may call, and spawnpoint's model endpoint
refuses every other model. Before you choose for a team app, call `whoami`: when
`team.ai_models` is there, or the project's `ai` block has `approved_only: true` (its
`models` is then the list), choose only from that list. If the table's choice is not on
it, take the closest approved model that serves the same call: a chat model for chat, and for
an app kept on the Anthropic SDK one that takes that SDK (the server's
instructions and the `ai` block's note name them; with none, keep the provider). Embeddings
need `cohere-embed-v3` on the list; there is no other embeddings model. Tell the user which
model and why. If no approved model serves the call, keep the provider for that call and
tell the user the team has not approved an open model for it (a team admin can approve
one, `cohere-embed-v3` for embeddings, on the team's page in the console). No tool changes
the list.

An app that does chat plus something the catalog lacks (image generation, audio, video)
moves the chat and keeps the provider for the rest; that still needs the provider key in `env` (section 4 says how the two clients
split). Current catalog: `get_project`'s `ai.models` once a project has a key, or
`GET <base_url>/models`.

## 4. Rewrite

Deploy with `inference: true`. The app then wakes up with `SPAWNPOINT_AI_URL`,
`SPAWNPOINT_AI_KEY`, `SPAWNPOINT_AI_MODEL` (`gpt-oss-120b`), and, unless the user set
their own, `OPENAI_BASE_URL` + `OPENAI_API_KEY` and `ANTHROPIC_BASE_URL` +
`ANTHROPIC_API_KEY`. The edit is the smallest one per SDK:

- **OpenAI SDK** (Node, Python): the client already reads the base URL and key from
  env. Change the chat model to `process.env.SPAWNPOINT_AI_MODEL || "gpt-oss-120b"` and
  the embeddings model to `"cohere-embed-v3"` on the same client, and drop
  `OPENAI_API_KEY` from the deploy's `env`. Pass no `dimensions` to the embeddings call.
  When the app embeds a search query (not a document it stores), add
  `input_type: "search_query"` to that call's parameters (Python:
  `extra_body={"input_type": "search_query"}`); documents need nothing.
- **Chat moves, the rest stays** (images or audio on the provider, or the team has not
  approved `cohere-embed-v3`): keep `OPENAI_API_KEY` in the
  deploy's `env`. spawnpoint then leaves `OPENAI_BASE_URL` and `OPENAI_API_KEY` alone,
  so the default client still reaches OpenAI: leave those calls on it. Give
  the chat calls (and embeddings, when they move) a second client, `new OpenAI({ baseURL: process.env.SPAWNPOINT_AI_URL,
  apiKey: process.env.SPAWNPOINT_AI_KEY })` (Python: `OpenAI(base_url=...,
  api_key=...)` from the same two vars), with the model from `SPAWNPOINT_AI_MODEL`
  (`"cohere-embed-v3"` for embeddings).
  `SPAWNPOINT_AI_URL` is the OpenAI-style base and ends in `/v1`: never hand it to the
  Anthropic SDK. An Anthropic SDK app that keeps `OPENAI_API_KEY` for OpenAI
  embeddings changes nothing on its chat client: drop `ANTHROPIC_API_KEY` and it still
  reads `ANTHROPIC_BASE_URL` and `ANTHROPIC_API_KEY` from env.
- **Anthropic SDK**: the same through env; model `gpt-oss-120b`; drop
  `ANTHROPIC_API_KEY`. In a stream, a tool call arrives whole at the end, not piece by
  piece. A stop at one of `stop_sequences` usually reports `end_turn`, not
  `stop_sequence`. `document` blocks, structured output (`output_config.format`), and
  server tools are refused with a sentence: send the document's text in the message,
  and ask for JSON in the prompt.
- **Vercel AI SDK**: `createOpenAI({ baseURL: process.env.OPENAI_BASE_URL, apiKey:
  process.env.OPENAI_API_KEY })` instead of the default instance; model from env.
- **LangChain**: `ChatOpenAI(base_url=..., api_key=..., model=...)` from env, and
  `OpenAIEmbeddings(base_url=..., api_key=..., model="cohere-embed-v3",
  check_embedding_ctx_length=False)` (it would otherwise send token ids, not text).
- **Raw HTTP**: the host becomes `SPAWNPOINT_AI_URL`, the bearer `SPAWNPOINT_AI_KEY`.

Never write a key into a file, never print one, never hard-code the endpoint, and do
not change a prompt (except to ask for JSON, as above). Keep a local fallback
(`|| "gpt-oss-120b"`) so the app also runs on a developer's machine with their own
`OPENAI_BASE_URL`.

`bge-m3` is retired, and the endpoint refuses it: an app that called it must name
`cohere-embed-v3` (not `embed`) and embed every stored text again, since its stored
vectors do not compare with the new model's. `laya` and `POST /decisions` are retired:
move those calls to a chat model with JSON by prompt.
`glm-5.2`, `glm-5.3-flash`, `qwen3.8-27b`, and `deepseek-ocr-2` still answer, on
`gpt-oss-120b`, `gemma-4-26b`, `mistral-large-3`, and `gemma-4-31b`; write the new name
when you edit the code.

## 5. Tell the user

Say the app runs on `<model>` through spawnpoint, that its requests go to Amazon Bedrock
in AWS's Oregon region, which keeps none of them (the line from the ask; for an app you
wrote from scratch, this is the first time the user hears it), that no OpenAI or
Anthropic key was needed, and which rule applies (the reasoning budget, the text block,
JSON by prompt). An app whose embeddings moved: say they run on `cohere-embed-v3`, about
20 calls a minute across spawnpoint, so the app batches texts and retries when busy, and
that stored vectors from another model were (or must be) embedded again. An app that kept part of its calls on the provider: say the
assurances cover only the calls that moved, that the rest still go to the provider under
its own terms, and that its key stays in `env`. `get_project` shows the `ai` block: `base_url`,
`key_env`, `model`, `models`. The key dies with the project. A deploy refused with
"Open models are not enabled on this spawnpoint": deploy without `inference` and with
the provider key in `env`, and say so.
