# 06 — AI providers

Your install calls AI providers directly with **your** keys. No vendor proxy sits in between, and the vendor gets no usage data.

| Provider | `.env` | Used by |
|---|---|---|
| Anthropic | `ANTHROPIC_API_KEY=sk-ant-…` (+ `AI_PROVIDER=anthropic`) | chat, `agent` steps, AI builders, `ai_agent` steps (Claude models) |
| OpenAI | `OPENAI_API_KEY=sk-…` (+ `AI_PROVIDER=openai` to make it the default) | same, with GPT models |
| OpenRouter | `OPENROUTER_API_KEY=sk-or-…` | extra `ai_agent` models (Gemini, Llama, DeepSeek, Grok, …) |

After editing, run `./webarmctl restart`. In the pipeline editor, the `ai_agent` model list shows only the models whose provider key is set (`GET /api/pipelines/ai-models`).

## Choosing models

- `CLAUDE_MODEL` / `OPENAI_MODEL` set the model for chat and `agent` steps. A small, fast model is a good default here.
- Each `ai_agent` step picks its own model. You can also set a `fallback_model`, which is used if the first one fails.
- To add models, or rename them in the catalog:
  ```bash
  AI_AGENT_MODELS=[{"provider":"openrouter","id":"google/gemini-2.5-pro","label":"Gemini 2.5 Pro","tier":"premium","vision":true}]
  ```

## Cost control

- Prompt caching is on for Anthropic (`ANTHROPIC_PROMPT_CACHE=true`). It cuts the cost of agent loops a lot.
- `AGENT_MAX_OUTPUT_TOKENS` caps each model turn.
- `LLM_REQUESTS_PER_MINUTE` caps total traffic to the provider.
- Set per-key spend limits in the provider console. You can also use separate keys per environment.

## Gateways, proxies, private endpoints

- **Outbound HTTP proxy:** add `HTTPS_PROXY=http://proxy:3128` and `NO_PROXY=db,localhost,127.0.0.1` to `.env`.
- **OpenAI-compatible gateway** (LiteLLM, vLLM, an Azure OpenAI proxy, a self-hosted model server):
  1. Set `OPENROUTER_BASE_URL=https://llm-gateway.internal/v1` and `OPENROUTER_API_KEY=<gateway key>`.
  2. List the gateway's models with `AI_AGENT_MODELS` (`"provider":"openrouter","id":"<model name on the gateway>"`).
  3. If the gateway is on a private network, also set `AI_AGENT_ALLOW_PRIVATE_NETWORK=true`. This is only needed when `ai_agent` web tools must reach it; model calls are not restricted.
- **Data residency:** choose provider regions or enterprise endpoints according to your policies. Prompts include the page content your agents read.
