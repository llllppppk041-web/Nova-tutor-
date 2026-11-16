# Integrate the gateway into an OmniSearch Interstellar checkout

This repository holds the gateway patch. It expects the original OmniSearch app modules to be present in the same backend package.

## 1. Copy the service

Place `backend/app/services/provider_gateway.py` in:

`backend/app/services/provider_gateway.py`

It imports the existing request/response schemas, local runner and teaching-character catalog; those modules remain part of the full app source.

## 2. Wire FastAPI routes

In `backend/app/main.py`, import:

```python
from .services.provider_gateway import (
    ask_configured_brain,
    diagnose_openai_compatible,
    provider_runtime_status,
)
```

Expose the safe status route:

```python
@app.get("/api/providers/status")
def ai_provider_status():
    return provider_runtime_status()
```

Expose a diagnostic route protected by the application's local-only policy:

```python
@app.post("/api/diagnostics/provider")
async def diagnose_ai_provider():
    return await diagnose_openai_compatible()
```

In the existing `POST /api/ask` handler, route the request using:

```python
result = await ask_configured_brain(request, local_runner=ask_local_brain)
```

Keep the app's existing exception mapping and `AskResponse` validation.

## 3. Frontend routing

The full app's PWA checks `GET /api/providers/status`. If `server_provider_selected` is true, it sends the existing AskRequest to the same-origin `/api/ask` route. If status is unavailable, the client continues using the existing local browser/native path.

Never put provider credentials in frontend code or Android resources.

## 4. Configure a local BitNet server

Start the upstream Microsoft BitNet `llama-server` on loopback, then set:

```dotenv
OMNI_AI_PROVIDER=openai_compatible
OMNI_OPENAI_BASE_URL=http://127.0.0.1:8080/v1
OMNI_OPENAI_MODEL=bitnet
OMNI_OPENAI_API_KEY=
```

Use `POST /api/diagnostics/provider` to verify actual model inference.

## 5. Regression tests

Run the original full application test suite after integration:

```bash
cd backend && PYTHONPATH=. pytest -q
cd ../frontend && npm test && npm run check
cd .. && python scripts/check_android_npu.py
python scripts/security_invariants.py
python scripts/verify_release.py --verify
```

The full-source archive supplied with this work already contains the gateway wired into the FastAPI application, tests and frontend routing.
