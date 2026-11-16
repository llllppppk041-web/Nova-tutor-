# Connected AI Gateway

## Supported pathways

### 1. Local-first (default)

Set `OMNI_AI_PROVIDER=local_command`. The original local model/BitCOS custom runner remains the primary brain. No hosted provider is contacted unless the operator explicitly enables remote fallback.

### 2. Microsoft BitNet

Microsoft's official `bitnet.cpp` project implements CPU/GPU inference for supported BitNet b1.58 checkpoints. It includes an upstream server wrapper that launches `llama-server`, which exposes an OpenAI-compatible chat-completions endpoint.

- Runtime: https://github.com/microsoft/BitNet
- Model collection: https://huggingface.co/collections/microsoft/bitnet
- Official 2.4B parameter checkpoint: https://huggingface.co/microsoft/BitNet-b1.58-2B-4T
- Technical report: https://arxiv.org/abs/2502.11880

Build the upstream runtime using the current instructions for your CPU/OS. A typical BitNet checkout can launch a loopback-only server with:

```bash
python run_inference_server.py \
  --model /absolute/path/to/ggml-model-i2_s.gguf \
  --host 127.0.0.1 --port 8080 --threads 4 --ctx-size 4096
```

The model path and context size must match the downloaded model and available memory. Do not bind an unauthenticated inference server to a public interface.

Configure OmniSearch's backend:

```dotenv
OMNI_AI_PROVIDER=openai_compatible
OMNI_OPENAI_BASE_URL=http://127.0.0.1:8080/v1
OMNI_OPENAI_MODEL=bitnet
OMNI_OPENAI_API_KEY=
```

A successful configuration check is not enough; call `POST /api/diagnostics/provider` for a real inference probe.

### 3. Hosted OpenAI-compatible endpoint

Configure the official HTTPS base URL, model ID and key in the backend's private environment. The adapter supports the standard non-streaming `/chat/completions` interface. It is not a promise that every provider implements the same request schema, JSON mode, model names, context limits or rate limits.

Never insert provider keys in frontend JavaScript, Android resources, browser storage, GitHub or student-visible content.

### 4. Optional Firecrawl retrieval

Set `OMNI_FIRECRAWL_API_KEY` in the backend environment to enable the optional Firecrawl search API. The gateway passes a small number of retrieved excerpts to the model as untrusted reference material and only returns source cards backed by actual URLs returned by Firecrawl.

Search API documentation: https://docs.firecrawl.dev/features/search

If the key is absent, web search is unavailable and the app must not pretend to have retrieved live sources.

## API route

The FastAPI integration provides:

- `GET /api/providers/status`: safe configuration status; never returns credentials or full endpoint URLs.
- `POST /api/diagnostics/provider`: sends a small diagnostic prompt and checks the model's response.
- `POST /api/ask`: routes the existing AskRequest through the configured brain and returns the existing AskResponse schema.

## Visual generation contract

The model is instructed to provide a structured teaching response with a causal visual explanation, storyboard beats, scene elements and bounded animation actions when appropriate. It must never return executable code for the renderer. The existing frontend performs a second validation pass and only uses supported primitives.

A robust pipeline is:

`Question → lesson plan → factual/mechanism structure → storyboard → validated scene graph → animation timeline → render → knowledge check`

For mathematical animation, use deterministic math/physics calculations for numerical values and let the model explain and direct the scene. Do not ask an LLM to invent exact simulation trajectories.

## BitNet versus BitCOS

BitNet b1.58 has a documented model family and supported inference runtime. The original OmniSearch source mentions BitCOS as a custom runner/checkpoint label but does not define a standard architecture, tokenizer, model format or converter. The gateway therefore preserves BitCOS as a custom command integration. A BitCOS checkpoint must not be renamed or treated as a BitNet model.

## Privacy boundary

When remote fallback is enabled, a learner's question and bounded learning context can leave the device. Both flags are required:

```dotenv
OMNI_AI_FALLBACK=openai_compatible
OMNI_REMOTE_FALLBACK_CONSENT=true
```

Keep the flags off for a fully local learning mode. Production remote deployment additionally needs authentication, per-student authorization, rate limiting and privacy disclosure; this gateway alone is not a multi-tenant security boundary.
