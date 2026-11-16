# OmniSearch Interstellar v5 — Research, Multi-Student and Kaggle CPU Evolution Plan

**Planning snapshot:** 16 November 2026  
**Base:** existing OmniSearch Interstellar v4.0.0 full-source bundle + the public AI gateway integration in this repository.  
**Execution target:** one bounded 8-hour Kaggle CPU notebook run. This is a plan, not a claim that a Kaggle job has been run.

## 1. What already exists in the app

The current full app already has:

- A typed, declarative lesson/animation schema with roughly 60 scene element types, 14 action types, connectors and cinematic effects.
- Canvas 2D animation as the safe baseline, plus a separate optional Three.js 0.186 lab.
- Five procedural 3D scenes: black hole, atom, DNA, solar system and wave interference.
- Function, parametric and vector-field plot modes with a bounded expression parser.
- An existing browser GGUF/Wllama CPU route and a native Android LiteRT-LM CPU route with optional exact-device NPU dispatch.
- A FastAPI local custom-runner interface and the new optional OpenAI-compatible provider gateway.
- Nine pedagogical response modes, local recall cards, quiz generation, review scheduling and topic progress signals.
- Source-level safeguards: the model returns declarative scene data, not executable JavaScript/HTML/SVG/CSS.

Do not rebuild these. Evolve them behind adapters and versioned schemas.

## 2. The target architecture

```text
                       STUDENT APP
             PWA / Android / local offline mode
                          |
             Authenticated student session
                          |
              Learning Orchestrator API
                          |
      +-------------------+-------------------+
      |                   |                   |
  Local AI route       Teacher router      Research broker
  BitNet / small       frontier models     search indexes
  GGUF / LiteRT        (opt-in cloud)      cache + citations
      |                   |                   |
      +-------------------+-------------------+
                          |
                 Lesson Compiler
        intent → causal model → storyboard → scene graph
                          |
              Validation + deterministic solvers
                          |
          +---------------+---------------+
          |               |               |
       Canvas 2D       Three.js        audio/TTS
          |               |               |
          +---------------+---------------+
                          |
                Timeline / interaction
                          |
           Quiz, explanation and mastery evidence
```

**Non-negotiable rule:** the LLM proposes a scene; deterministic domain modules calculate the physics, maths, geometry and time-series values. A renderer must not trust model-generated code. Unknown concepts should degrade to a safe diagram/timeline/explanation, not a fabricated simulation.

## 3. Provider strategy: 50 connectors, only a few active per task

Do not send every prompt to 50 providers. Build a provider catalogue/adapter registry that can hold up to 50 endpoints, but route each task to one primary and at most one or two reviewers. Use:
- A primary local model for ordinary lessons and offline use.
- One low-cost/fast cloud teacher only when the user enables online help.
- One independent verifier for consequential scientific claims or a new animation template.
- One research broker for source retrieval, not 50 duplicated model calls.
- A circuit breaker, timeouts, request budgets, retries with backoff, per-provider metrics and a kill switch.

### Candidate provider registry

| Provider | Candidate model/API | Intended job | Important limitation |
|---|---|---|---|
| Microsoft BitNet | bitnet.cpp + supported b1.58 GGUF | Local CPU teacher / fallback | Build and model support depend on CPU architecture; this is not automatically LiteRT-LM or browser-WASM compatible |
| Z.ai | GLM-5.3 | Long-horizon code/reasoning teacher | Paid inference; budget it |
| Z.ai | GLM-5.3-Flash | Fast multimodal teacher | Check endpoint lifecycle before depending on it; Cloudflare currently marks its route for deprecation on 20 Nov 2026 |
| Google Cloud | Gemini 4 Hydrogen | Fast frontier agentic teacher | Verify project access, model identifier and billing in the actual Google Cloud project; do not assume it is in the Gemini API free tier |
| OpenRouter | openrouter/free and selected :free model IDs | Optional prototype/fallback | Free variants currently have 50 requests/day and 20 requests/minute; free catalog and availability change |
| Cloudflare Workers AI | @cf/zai-org/glm-5.3 | Cloud-hosted coding/reasoning | This specific model requires Workers Paid or prepaid AI Gateway credits |
| Cloudflare Workers AI | @cf/zai-org/glm-5.3-flash | Multimodal routing test | This specific route is paid and marked planned deprecation; don't hard-wire it as the long-term default |
| Moonshot / Alibaba / NVIDIA | kimi-k3.1-pro | High-end reviewer / teacher | Cloud model, not phone local inference; provider, region and trial conditions matter |

**Snapshot facts:** OpenRouter describes its free pool as 25+ models / 4 providers and caps free variants at 50 requests per day and 20 per minute; its FAQ states the daily ceiling rises to 1,000 after at least $10 in credits. Cloudflare's Workers AI free allocation is 10,000 neurons/day overall, but its GLM-5.3 and GLM-5.3-Flash endpoints require Workers Paid or prepaid credits. Gemini rate limits are project-level RPM, TPM and RPD and differ by model/tier. Kimi K3.1 Pro is described by external provider docs as a 2.8T-total/104B-active MoE with a 1M-token context: it is a cloud teacher, not a realistic phone model.

Do not evade per-account limits by rotating accounts or keys. Use provider-approved budgets and terms.

## 4. What “260 free web searches” should mean

First define the unit. A “search” might mean one user question, one query sent to an index, one provider request, or one returned result. Those are not interchangeable.

Recommended target:
- Up to 260 **source-result fetches** in a broad research job, not 260 model-provider calls for every student question.
- Normal lesson: zero web searches unless asked or a fact is time-sensitive.
- Research lesson: 3–8 focused queries across independent source families; retrieve the strongest results; deduplicate by canonical URL and content hash.
- Deep research: bounded batches, e.g. 20–40 queries across a job, then continue only when evidence gaps remain.
- Save query and result cache by normalized topic, source, language, age band and date.
- Keep an audit trail: query, source, timestamp, result URL, title, retrieved excerpt, licence/terms, relevance score and freshness.
- Return citations to learners. No invented links and no claims that all source indexes were checked.

A 260-request target cannot be advertised as “instant, free, unlimited.” Provider rate limits, robots/terms, request costs, CORS, result quality and freshness make that false. Treat 260 as an optional research-job budget with per-source limits and a visible quota meter.

## 5. Multi-student architecture

Do not add shared memory without isolation. Every request and record must be scoped to:
- tenant / school or family,
- authenticated user,
- learner profile,
- course / subject,
- lesson and source-set identifiers.

Minimum controls before multiple real learners:
1. Authentication and role-based access for learner, guardian/teacher and admin.
2. Tenant-scoped database queries with server-side authorization checks.
3. No client-provided student ID trusted as authorization.
4. Isolated conversation history, model memory, retrieval indexes and generated artefacts.
5. Encrypted transport and protected secret storage.
6. Minimal learner data, retention/deletion/export controls and age-appropriate consent.
7. Rate limits per tenant, user, provider and IP; bounded concurrent generations.
8. Queue-based generation with back-pressure and cancellation.
9. Separate local/offline data store per device profile.
10. No sending student work to frontier providers or training pipelines without explicit disclosed consent and a provider-terms review.

Use Postgres (or another relational store) for identities, courses and audit metadata; Redis or an equivalent queue/cache for jobs and rate limits; object storage for immutable lesson artefacts; IndexedDB/Android app storage for private offline learner data. Avoid putting raw chat transcripts in logs.

## 6. Frontier distillation: teacher → verifier → student

A safe distillation pipeline:

1. Create task prompts across subjects, difficulty bands, language styles, misconception cases and visual lesson types.
2. Ask one teacher model for a structured response: explanation, evidence, assumptions, equations, storyboard and a declarative scene graph.
3. Ask an independent verifier to check reasoning, formulas, source coverage and unsupported claims.
4. Run deterministic checks: units, equations, geometry bounds, physics invariants, numerical tolerances, JSON schema and renderer compatibility.
5. Reject or manually review failed samples. Preserve teacher source, model ID, prompt version, date, license/terms and verification outcome.
6. Build a de-duplicated training set with held-out test splits and no contamination between train/eval.
7. Fine-tune/ distill a small student using a suitable GPU/TPU run. CPU-only Kaggle is for data creation, tests and inference benchmarks—not a realistic route to full foundation-model training.
8. Measure factual accuracy, pedagogy, schema validity, animation render success, latency, token rate, RAM, energy and language coverage before replacing the old model.
9. Keep RAG for current facts; do not attempt to distil news or rapidly changing reference data into model weights.
10. Never assume that a provider's API output may be used for training. Check each provider's current terms, data controls and licence first.

## 7. Interstellar Visual Compiler — make “almost any topic” drawable

Current primitives already cover a broad set of basic scenes. Add the following *modular* capabilities instead of making an unbounded AI-to-code renderer:

- **Concept graph:** entities, states, properties, relations, causal edges and uncertainty.
- **Storyboard director:** hook → intuition → causal mechanism → experiment / transfer task.
- **Scene graph compiler:** map graph nodes into existing typed 2D/3D primitives and validated connectors.
- **Domain simulators:** deterministic physics, mechanics, electricity, optics, waves, chemistry, cell biology, anatomy, astronomy, geometry, calculus and statistics.
- **Camera/lighting director:** safe finite presets (dolly, orbit, zoom, pan, depth of field only where renderer supports it).
- **Asset catalogue:** licensed, cached glTF models, vector icons, audio and textures with provenance; no hotlinked unknown assets.
- **Timeline engine:** keyframes, easing, event markers, audio cues, captions, pause/seek and reduced-motion behavior.
- **Fallback renderer:** when 3D or a particular asset is unavailable, render a 2D schematic and explain the simplification.
- **Visual QA:** screenshot tests, deterministic replay, geometry validation, accessibility checks and low-memory performance tiers.

The first five procedural 3D scenes should remain regression fixtures. Do not replace them with generated video. Use text-to-video/image models only as an optional cloud-produced background/media asset, not as the core explanation engine.

## 8. Kaggle CPU 8-hour run plan

Kaggle's published notebook specification currently describes CPU sessions with 4 CPU cores and 30 GB RAM, 20 GB auto-saved working storage, up to 12 hours for CPU/GPU sessions, and a 20-minute idle timeout for interactive editing. “Save & Run All” must finish inside the 12-hour session ceiling. An 8-hour plan fits on paper, but it is not a promise of guaranteed uninterrupted compute or unchanged quotas.

| Elapsed time | CPU notebook stage | Exit artefact / stop condition |
|---:|---|---|
| 00:00–00:25 | Hardware, disk, network, dependency and secret preflight; capture environment versions | Preflight JSON; abort early on missing network/space |
| 00:25–01:10 | Clone the public gateway repo; load the full app archive as a Kaggle Dataset/input; verify the 228-file SHA-256 manifest | Source tree + manifest pass |
| 01:10–02:00 | Run backend/frontend/static tests; establish baseline and save results | Baseline test report |
| 02:00–03:00 | Provider router: adapters, quotas, timeouts, circuit breakers and mock tests; test only providers whose keys are present | Provider compatibility matrix (no keys in notebook output) |
| 03:00–04:00 | Research broker: query planner, independent source adapters, deduplication, citation cache and per-source quota enforcement | 20–40-query sample and provenance report |
| 04:00–05:00 | Multi-student simulation with isolated mock tenants, queue limits and concurrent requests | Isolation/load-test report |
| 05:00–06:00 | Teacher-output dataset generation in bounded batches; verifier + deterministic JSON/math/physics/scene checks | Versioned accepted/rejected sample dataset |
| 06:00–07:00 | Visual compiler regression: create 2D/3D scene fixtures, render test frames, check schema and renderer fallbacks | Visual regression bundle |
| 07:00–07:35 | CPU-only local model benchmark: load a supported BitNet checkpoint if available; measure tokens/s, peak RSS and answer/scene validity | Device-independent inference benchmark |
| 07:35–08:00 | Final regression, archive, SHA-256, notebook output and release report | Reproducible artefacts and stop |

For a true 8-hour unattended run, use a saved/committed notebook execution rather than relying on an idle interactive editor. Persist checkpoints, dataset files and logs to `/kaggle/working`; never leave API keys in notebook output. An interruption should resume from the last completed stage, not restart paid API calls.

## 9. What Kaggle CPU should and should not do

**Good CPU tasks:** source validation, tests, provider adapter mocks, web-result ranking, dataset cleaning, schema checking, CPU inference measurements, quantized model sanity tests, deterministic math/physics and visual regression.

**Not a sensible CPU-only target:** training a new frontier model, fine-tuning a 2.4B+ model for useful quality in eight hours, or running dozens of frontier providers in parallel without quotas. For actual model training use an appropriate GPU/TPU job after the dataset passes QA. Kaggle CPU's 4 cores and 30 GB RAM are useful for pipeline validation, not frontier-scale training.

## 10. Release gates

Do not mark v5 ready until all of these are true:
- Provider keys never appear in client bundles, notebook outputs or logs.
- All remote calls are explicit, opt-in and budgeted.
- Per-student access tests prove cross-tenant isolation.
- All animation JSON passes schema validation before render.
- Math/physics output is checked by deterministic solvers and unit tests.
- Research citations are actual retrieved URLs, not model-created.
- The app remains usable offline with web research and cloud teachers disabled.
- Phone benchmark reports measured RAM/FPS/token rate on the target handset.
- The full suite and release SHA manifest pass.
- Every optional cloud provider can be disabled without breaking the core local lesson loop.

## Official references checked for this plan

- Kaggle notebook limits: https://www.kaggle.com/docs/notebooks
- Microsoft BitNet: https://github.com/microsoft/BitNet
- OpenRouter free models and rate limits: https://openrouter.ai/collections/free-models
- OpenRouter pricing: https://openrouter.ai/pricing
- Cloudflare Workers AI pricing: https://developers.cloudflare.com/workers-ai/platform/pricing/
- Cloudflare GLM-5.3 model: https://developers.cloudflare.com/workers-ai/models/glm-5.3/
- Cloudflare GLM-5.3-Flash model: https://developers.cloudflare.com/workers-ai/models/glm-5.3-flash/
- Google Gemini API rate limits: https://ai.google.dev/gemini-api/docs/rate-limits
- Google Gemini API pricing: https://ai.google.dev/gemini-api/docs/pricing
- Gemini 4 Hydrogen model page: https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/gemini/4-hydrogen
- Kimi K3.1 Pro model card: https://huggingface.co/moonshotai/Kimi-K3.1-Pro
- NVIDIA NIM Kimi K3.1 Pro: https://docs.api.nvidia.com/nim/reference/moonshotai-kimi-k3_1-pro
- Alibaba Cloud Kimi K3.1 Pro: https://www.alibabacloud.com/help/en/model-studio/kimi-k3-1-pro

## Important distinction

This roadmap is a design and execution plan. The public repository is currently the AI gateway integration layer, while the full OmniSearch v4.0.0 application is supplied as a separate source bundle. No Kaggle execution, provider-account connection, model training, Android APK install or live phone benchmark is claimed by this document.
