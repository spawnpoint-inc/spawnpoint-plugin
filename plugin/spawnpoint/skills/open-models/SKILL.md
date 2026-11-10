---
name: open-models
description: Run an app's OpenAI or Anthropic calls on open models through spawnpoint, with no provider key. Use when an app about to be deployed to spawnpoint calls a model provider (the openai or anthropic SDK, the Vercel AI SDK, LangChain, or api.openai.com / api.anthropic.com over HTTP), or when the user asks for open models, a free model, or an app with no API key.
---

# Open models on spawnpoint

spawnpoint can run an app's chat calls on open models (gpt-oss, Gemma, and Mistral)
through its own endpoint. It has no embeddings model: those stay on the provider. The user needs no provider account and no
key: the deploy mints the project's key and ships it as env. The user chooses, once, per
project.

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

> This app calls OpenAI for chat and embeddings. spawnpoint can run the chat on an open
> model (gpt-oss-120b) with no OpenAI cost for it. The embeddings stay on OpenAI with
> your key, since spawnpoint's open models do not include embeddings. Chat requests
> would go through spawnpoint to Amazon Bedrock in AWS's Oregon region, which is set to
> keep none of them (https://getspawnpoint.com/privacy.html#who-else). Switch the chat,
> or keep OpenAI for everything?

An app whose only model calls are embeddings has nothing to switch: keep the provider
and do not ask.

Never switch without the answer. Never ask twice for one project: a project that
already exists without an `ai` block on `get_project` was deployed on the provider on
purpose, so redeploy it as it is unless the user raises it. When the user already said
which they want (in the request, or earlier in the conversation), skip the question.
An app you are writing from scratch that needs a model: use the open model with no
question, since there is no provider to keep.

Every ask names where requests go, as above. When the user asks for more, or when the
app would send other people's personal data or confidential material (customer records,
call transcripts, mailbox contents) to the model, give the rest: each request goes
through spawnpoint to Amazon Bedrock in AWS's us-west-2 region (Oregon), and no other
region. spawnpoint's AWS account is set to zero data retention for Bedrock, so AWS keeps
no requests or answers, does not use them to train models, and does not share them with
the companies that make the models. spawnpoint stores no prompts or answers either, only
a usage record per call (project, model, tokens, cost, time). The models are made by
OpenAI, Google, and Mistral AI. Say this plainly before switching such an app,
and offer keeping the provider, with its own key and retention terms, as the other
choice. https://getspawnpoint.com/how-apps-run.html has the details.

## 3. Choose

| The app does | Model | Pass on |
|---|---|---|
| Chat, assistants, summarizing, tool calling | `gpt-oss-120b` (OpenAI) | `max_tokens` 300 or more: it reasons before it answers, and a small budget returns empty content. The reasoning is in `message.reasoning`; `message.content` holds only the answer |
| High volume or latency-sensitive chat | `gpt-oss-20b` (OpenAI) | Same budget rule |
| Chat in many languages, with no images | `mistral-large-3` (Mistral AI) | Does not read images: an image sent to it is refused |
| Anthropic Messages format (the Anthropic SDK, kept as is) | `gpt-oss-120b` | Read the block whose `type` is `text`, not `content[0]`: a `thinking` block comes first when the request asks for thinking |
| Images in the conversation (describe, read, compare, receipts, scans) | `gemma-4-31b` (Google) | Data URIs work; `gemma-4-26b` is the faster second choice |
| Embeddings, search, RAG | keep the provider | Not on the open catalog: keep the provider's embeddings model and its key in `env`. Chat in the same app can still move |
| Structured JSON output | any chat model | No `response_format`: ask for JSON in the prompt, parse, retry once on failure |
| Sorting text into your own categories, scoring it, or a yes/no (routing, moderation, triage, tagging) | `gpt-oss-20b` | JSON by prompt, as above: name the allowed values in the prompt and check the answer is one of them |
| Image generation, video, audio, speech | keep the provider | Not on the open catalog: say so, keep that key in `env`, and move on |

When the user's policy rules out some models (by maker or by country), choose among the
rest and say which one you used; when none fits, keep the provider.

A team can approve which models its apps may call, and spawnpoint's model endpoint
refuses every other model. Before you choose for a team app, call `whoami`: when
`team.ai_models` is there, or the project's `ai` block has `approved_only: true` (its
`models` is then the list), choose only from that list. If the table's choice is not on
it, take the closest approved model that serves the same call: a chat model, and for
an app kept on the Anthropic SDK one that takes that SDK (the server's
instructions and the `ai` block's note name them; with none, keep the provider). Tell the user which model and
why. If no approved model serves the call, keep the provider and tell the user the team
has not approved an open model for this. No tool changes the list: a team admin changes
it in the console.

An app that does both (chat plus embeddings, or chat plus image generation) moves the
chat and keeps the provider for the rest; that still needs the provider key in `env`
(section 4 says how the two clients split). Current catalog:
`get_project`'s `ai.models` once a project has a key, or `GET <base_url>/models`.

## 4. Rewrite

Deploy with `inference: true`. The app then wakes up with `SPAWNPOINT_AI_URL`,
`SPAWNPOINT_AI_KEY`, `SPAWNPOINT_AI_MODEL` (`gpt-oss-120b`), and, unless the user set
their own, `OPENAI_BASE_URL` + `OPENAI_API_KEY` and `ANTHROPIC_BASE_URL` +
`ANTHROPIC_API_KEY`. The edit is the smallest one per SDK:

- **OpenAI SDK** (Node, Python): the client already reads the base URL and key from
  env. Change the model to `process.env.SPAWNPOINT_AI_MODEL || "gpt-oss-120b"`, and
  drop `OPENAI_API_KEY` from the deploy's `env`.
- **Chat moves, embeddings (or images, audio) stay**: keep `OPENAI_API_KEY` in the
  deploy's `env`. spawnpoint then leaves `OPENAI_BASE_URL` and `OPENAI_API_KEY` alone,
  so the default client still reaches OpenAI: leave the embeddings calls on it. Give
  the chat calls a second client, `new OpenAI({ baseURL: process.env.SPAWNPOINT_AI_URL,
  apiKey: process.env.SPAWNPOINT_AI_KEY })` (Python: `OpenAI(base_url=...,
  api_key=...)` from the same two vars), with the model from `SPAWNPOINT_AI_MODEL`.
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
- **LangChain**: `ChatOpenAI(base_url=..., api_key=..., model=...)` from env.
- **Raw HTTP**: the host becomes `SPAWNPOINT_AI_URL`, the bearer `SPAWNPOINT_AI_KEY`.

Never write a key into a file, never print one, never hard-code the endpoint, and do
not change a prompt (except to ask for JSON, as above). Keep a local fallback
(`|| "gpt-oss-120b"`) so the app also runs on a developer's machine with their own
`OPENAI_BASE_URL`.

An app already on open models that calls embeddings (`bge-m3`, `embed`, or any name):
every embeddings call is refused with a sentence, so move the embeddings back to a
provider with its own key in `env` (the two-client split above), then embed the stored
texts again with the provider's model and replace the old vectors (vectors from two
models do not compare). Its chat stays on spawnpoint. `laya` and
`POST /decisions` are retired: move those calls to a chat model with JSON by prompt.
`glm-5.2`, `glm-5.3-flash`, `qwen3.8-27b`, and `deepseek-ocr-2` still answer, on
`gpt-oss-120b`, `gemma-4-26b`, `mistral-large-3`, and `gemma-4-31b`; write the new name
when you edit the code.

## 5. Tell the user

Say the app runs on `<model>` through spawnpoint, that its requests go to Amazon
Bedrock in AWS's Oregon region, which keeps none of them (the line from the ask; for an
app you wrote from scratch, this is the first time the user hears it), that no OpenAI or
Anthropic key was needed, and which rule applies (the reasoning budget, the text block,
JSON by prompt). An app that kept part of its calls on the provider (embeddings,
images, audio): say those assurances cover only the calls that moved, that the rest
still go to the provider under its own terms, and that its key stays in `env`. `get_project` shows the `ai` block: `base_url`,
`key_env`, `model`, `models`. The key dies with the project. A deploy refused with
"Open models are not enabled on this spawnpoint": deploy without `inference` and with
the provider key in `env`, and say so.
