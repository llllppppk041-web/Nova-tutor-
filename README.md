# OmniSearch Interstellar — AI Gateway

**Open-source integration layer for OmniSearch Interstellar v3.9.0 / v4.0.0.**

This repository publishes the connected-AI gateway code developed for OmniSearch Interstellar. It connects the existing teaching engine to:

- A local Microsoft BitNet `bitnet.cpp` `llama-server` endpoint.
- Other operator-configured OpenAI-compatible APIs over HTTPS.
- Optional Firecrawl web retrieval, with source links restricted to URLs actually returned by retrieval.
- The existing local command runner, retained as the privacy-first default.

## Status

This repository is the **AI gateway integration**, not a complete mirror of the entire OmniSearch app. The full application includes Android, PWA, animation renderers, study modules and a much larger test suite. The separately supplied v4.0.0 source bundle contains those original modules plus this integration.

## Architecture

```text
OmniSearch PWA / Android client
            |
            | same-origin /api/ask
            v
     FastAPI application
            |
            v
   Provider gateway / router
       |              |
       |              +---- Firecrawl search (optional)
       |
       +---- Local custom runner (default)
       |
       +---- OpenAI-compatible endpoint
                    |
                    +---- Microsoft BitNet llama-server
                    +---- HTTPS hosted provider
            |
            v
   validated AskResponse JSON
            |
            v
  safe scene / animation renderer
```

The model generates declarative scene data, not executable JavaScript, HTML, SVG or CSS. The existing frontend validates scene geometry and animation actions before rendering them.

## Quick setup

1. Copy `.env.example` to a private `.env` file in the backend.
2. Keep `OMNI_AI_PROVIDER=local_command` for local-first operation.
3. To use a local BitNet server, set `OMNI_AI_PROVIDER=openai_compatible` and `OMNI_OPENAI_BASE_URL=http://127.0.0.1:8080/v1`.
4. For a hosted provider, use an official HTTPS endpoint and place its key only in the backend environment.
5. Start the FastAPI app and inspect `GET /api/providers/status`.
6. Run `POST /api/diagnostics/provider` to verify actual model inference.

Detailed instructions: [AI_PROVIDER_GATEWAY.md](AI_PROVIDER_GATEWAY.md).

## Security defaults

- Local command runner remains the default.
- No API keys are committed.
- Remote providers require HTTPS and a backend-only key.
- Remote fallback requires explicit operator consent.
- Browser clients cannot choose arbitrary provider URLs.
- Model-invented source URLs are discarded.
- Firecrawl search excerpts are treated as untrusted material, not instructions.
- Bind production instances to loopback until authentication and a reviewed remote-access layer are added.

## BitNet and BitCOS

Microsoft BitNet b1.58 is a documented model family with the `bitnet.cpp` inference runtime. BitCOS is retained only as a custom local runner/checkpoint label from the original project; it is not represented here as a standard model format or interchangeable with BitNet.

## Licence

MIT. See [LICENSE](LICENSE). The licence applies to the files in this repository. Check licences of external models, libraries and any separately supplied full application components before redistribution.
