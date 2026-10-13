---
name: open-models
description: Run an app's OpenAI or Anthropic calls on open models through spawnpoint, with no provider key. Use when an app about to be deployed to spawnpoint calls a model provider (the openai or anthropic SDK, the Vercel AI SDK, LangChain, or api.openai.com / api.anthropic.com over HTTP), or when the user asks for open models, a free model, or an app with no API key.
---

# Open models on spawnpoint

spawnpoint can run an app's model calls on open models (gpt-oss, GLM, Gemma, DeepSeek,
bge) through its own endpoint. The user needs no provider account and no key: the deploy
mints the project's key and ships it as env. The user chooses, once, per project.

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

> This app calls OpenAI for chat and embeddings. spawnpoint can run it on open models
> (gpt-oss-120b for chat, bge-m3 for embeddings) with no OpenAI key and no provider
> account. Switch, or keep OpenAI?

Never switch without the answer. Never ask twice for one project: a project that
already exists without an `ai` block on `get_project` was deployed on the provider on
purpose, so redeploy it as it is unless the user raises it. When the user already said
which they want (in the request, or earlier in the conversation), skip the question.
An app you are writing from scratch that needs a model: use the open model with no
question, since there is no provider to keep.

## 3. Choose

| The app does | Model | Pass on |
|---|---|---|
| Chat, assistants, summarizing, tool calling | `gpt-oss-120b` | `max_tokens` 300 or more: it reasons before it answers, and a small budget returns empty content |
| High volume or latency-sensitive chat | `gpt-oss-20b` | Same budget rule |
| Anthropic Messages format (the Anthropic SDK, kept as is) | `glm-5.2` | A `thinking` block precedes the text: read the block whose `type` is `text`, not `content[0]` |
| Images in the conversation (describe, read, compare) | `gemma-4-31b` | Data URIs work; `glm-5.3-flash` is the faster second choice |
| Embeddings, search, RAG | `bge-m3` | 1024 dimensions; an index built on another model must be rebuilt |
| Structured JSON output | any chat model | No `response_format`: ask for JSON in the prompt, parse, retry once on failure |
| Sorting text into your own categories, scoring it on your own scale, or a yes/no with a probability (routing, moderation, triage, tagging) | `laya` | `POST $SPAWNPOINT_AI_URL/decisions` with `{model, state, questions: {id: {type: choice|score|noul, instructions, criteria}}}`, bearer `SPAWNPOINT_AI_KEY`; one pass, about 0.2 s, no text written. Over-confident as shipped: check it on a few labeled examples before trusting it with something that matters |
| Image generation, video, audio, speech | keep the provider | Not on the open catalog: say so, keep that key in `env`, and move on |

An app that does both (chat plus image generation) moves the chat and keeps the
provider for the rest; that still needs the provider key in `env`. Current catalog:
`get_project`'s `ai.models` once a project has a key, or `GET <base_url>/models`.

## 4. Rewrite

Deploy with `inference: true`. The app then wakes up with `SPAWNPOINT_AI_URL`,
`SPAWNPOINT_AI_KEY`, `SPAWNPOINT_AI_MODEL` (`gpt-oss-120b`), and, unless the user set
their own, `OPENAI_BASE_URL` + `OPENAI_API_KEY` and `ANTHROPIC_BASE_URL` +
`ANTHROPIC_API_KEY`. The edit is the smallest one per SDK:

- **OpenAI SDK** (Node, Python): the client already reads the base URL and key from
  env. Change the model to `process.env.SPAWNPOINT_AI_MODEL || "gpt-oss-120b"` (or the
  catalog name for embeddings), and drop `OPENAI_API_KEY` from the deploy's `env`.
- **Anthropic SDK**: the same through env; model `glm-5.2`; drop `ANTHROPIC_API_KEY`.
- **Vercel AI SDK**: `createOpenAI({ baseURL: process.env.OPENAI_BASE_URL, apiKey:
  process.env.OPENAI_API_KEY })` instead of the default instance; model from env.
- **LangChain**: `ChatOpenAI(base_url=..., api_key=..., model=...)` from env.
- **Raw HTTP**: the host becomes `SPAWNPOINT_AI_URL`, the bearer `SPAWNPOINT_AI_KEY`.
  Decisions have no SDK: a plain POST, answer in `answers.<id>.choice`, `.score`
  (with `.legend`), or `.noul` (the probability of yes), plus `.confidence`.

Never write a key into a file, never print one, never hard-code the endpoint, and do
not change a prompt. Keep a local fallback (`|| "gpt-oss-120b"`) so the app also runs
on a developer's machine with their own `OPENAI_BASE_URL`.

## 5. Tell the user

Say the app runs on `<model>` through spawnpoint, that no OpenAI or Anthropic key was
needed, and which rule applies (the reasoning budget, the thinking block, JSON by
prompt, the embedding dimension). `get_project` shows the `ai` block: `base_url`,
`key_env`, `model`, `models`. The key dies with the project. A deploy refused with
"Open models are not enabled on this spawnpoint": deploy without `inference` and with
the provider key in `env`, and say so.
