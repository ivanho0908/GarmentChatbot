# GarmentChatbot

**COMP4431 Artificial Intelligence 2026/27 Group Project — The Hong Kong Polytechnic University**

## 1. Project Overview

GarmentChatbot helps fashion design students and designers discover existing T-shirt references for design exploration and mood boards. Users can describe a garment, upload a reference photograph, or combine an image with a supported change such as “make it black.” The system retrieves real catalog photographs; changing a requirement changes the search, rather than generating or editing an image.

This repository contains the browser frontend, FastAPI backend, saved evaluation and original image catalog. Documentation follows the submitted final project report, with implementation details checked against this repository. Deployment observations in the report are historical evidence, not a current service-availability guarantee.

## 2. Group Members

Names and student IDs are transcribed exactly from the report's team-member table.

| English name | Student ID |
|---|---|
| Ho Siu Wang | 24036797d |
| Kwok Ho Yam | 24058195d |
| Feng,Qihao | 24063665d |

## 3. Basic Features

The chatbot accepts English garment descriptions and retrieves catalog products using semantic similarity and supported attributes. Its language workflow distinguishes search, fashion information, unrelated requests and clarification. Structured extraction covers category, colours, patterns, neckline, sleeve, style and visible buttons; unspecified properties remain absent. Subjective concepts can remain in semantic search text without becoming invented metadata labels.

Responses display retrieved identities and photographs. Prices, inventory, brands and material properties are not verified product capabilities. Exact queries may legitimately produce few or no results.

## 4. Additional Features

### Natural-language understanding and controlled execution

A Responses-compatible LLM proposes a bounded plan containing intent, search strategy, resolved query, relation to context and result count. Backend validation and known retrieval functions control execution. The LLM does not execute arbitrary code or directly administer Qdrant. Separate strict schemas extract text attributes and image modifications; unsupported or ambiguous changes can produce clarification rather than fabricated requirements.

### Image and image-plus-text retrieval

FashionCLIP encodes a reference into the same normalized vector space as the catalog. A zero-shot classifier compares that vector with garment-label prompts to infer optional attributes. Image-plus-text search retains the original image vector for ranking and applies supported text overrides as mandatory filters. A black override replaces an inferred white colour for that field. This provides visual discovery with explicit user control, without image recolouring or a newly trained fusion model.

### Search modes

| Mode | Text request | Image request, including optional text |
|---|---|---|
| Auto | Planner chooses exact, similar or both; not a fixed fallback sequence. | Similarity with reliable optional image attributes; inferred constraints may relax, but explicit text requirements remain. |
| Exact | Requires supported attributes and ranks matching products using the text vector. | Retains mandatory overrides and up to two reliable inherited image attributes; strict filters are not silently dropped. |
| Similar | Semantic ranking without exact text-attribute requirements. | Original image-vector ranking without inferred hard filters; explicit supported modifications still apply. |

Image filtering uses score separation heuristics: a minimum margin of 0.015 for most attributes and 0.04 for style. Auto admits at most two optional attributes and checks candidate availability, normally requiring four times the requested result count. These heuristics are not calibrated confidence probabilities or proven optimal thresholds.

### Conversational refinement and continuation

Resolved queries, effective filters, mode and displayed identifiers support turns such as “make it long sleeve” and “show more.” Image-reference state includes a model identifier, 512-dimensional normalized embedding and attribute analysis, allowing compatible follow-ups without retransmitting the upload. Validation rejects incompatible or malformed state. Previously displayed product/point IDs are excluded during continuation; this prevents repeated identities within retained state, not visually similar products under different IDs.

Requests default to eight products and support counts from one to eight. The carousel shows at most four cards per page; moving within the carousel reveals already retrieved products, whereas “show more” performs continuation.

### Browser interface and speech

The HTML/CSS/JavaScript interface offers image attachment, Auto/Exact/Similar selection, product inspection, automatic conversation titles and local history. Browser `localStorage` preserves compatible text and reference state on the same origin; original upload files and previews are not durable assets. There is no server-side user account or cross-device history synchronization.

Browser speech recognition converts voice input to ordinary text requests. Speech synthesis provides manual response playback and optional automatic replies for voice submissions. These use browser services rather than a separate project speech model. Actual microphone acceptance remains unverified and depends on browser support, permissions, language, hardware and a suitable secure context.

## 5. System Architecture

```text
Browser: text / image / mode / bounded conversation state
    |
    | JSON or multipart HTTP
    v
FastAPI: validation, orchestration and response assembly
    +--> Responses-compatible LLM: plan, resolve context, extract attributes
    +--> FashionCLIP: normalized text or image query vector
    |       +--> Zero-shot label prompts: optional image attributes
    +--> Qdrant: metadata eligibility + cosine-vector ranking
    |       --> ranked product payloads and identifiers
    +--> /images: original JPEGs from data/images/
    |
    v
Browser: product cards, details, refinement and continuation

Offline preparation (separate, not run on API startup):
Catalog images --> FashionCLIP + attribute classifier --> Qdrant vectors/payloads
```

Different attribute fields combine using Qdrant `must`; alternatives within one field use `MatchAny`, and Boolean button requirements use `MatchValue`. Qdrant stores vectors and metadata, while JPEG files remain in the repository catalog. Image paths resolve within the permitted asset root.

Endpoints include `POST /api/chat`, `POST /api/image-search`, `POST /api/image-search/continue`, `GET /api/health` and static `/images/...`. See [API documentation](backend/API.md) and [architecture notes](docs/ARCHITECTURE.md). Uploaded images are decoded in memory with format, byte and pixel validation.

## 6. Technologies and Dependencies

| Component | Role |
|---|---|
| HTML, CSS, JavaScript ES modules | Build-free browser application; Node runs frontend tests. |
| FastAPI, Pydantic, Uvicorn, python-multipart | HTTP API, typed validation, ASGI serving and multipart uploads. |
| PyTorch, Transformers, safetensors | Load pretrained FashionCLIP and compute embeddings. |
| `patrickjohncyh/fashion-clip` | Saved-study image/text representation; compatible 512-dimensional collection required. |
| NumPy, Pillow | Vector operations and image decoding. |
| Qdrant client and external Qdrant service | Filtered cosine search over existing catalog vectors/payloads. |
| OpenAI Python SDK and Responses-compatible endpoint | Structured planning, extraction and conversational wording; actual model selected privately by configuration. |
| sentence-transformers / `all-MiniLM-L6-v2` | Legacy optional text-to-label extraction when explicit filters are not supplied; not the product-vector encoder. |
| python-dotenv, pandas, tqdm | Configuration and supporting data/ingestion utilities. |

See [backend requirements](backend/requirements.txt), [test requirements](backend/requirements-api-test.txt) and [recorded development versions](docs/TESTED_VERSIONS.md). Requirements are not a cross-platform lock. The root Firebase package/configuration supports hosting; it is unnecessary for the local static frontend.

The report records Firebase Hosting with a CPU Cloud Run backend. This checkout's public [frontend configuration](frontend/config.js) instead targets `https://garmentchatbot.onrender.com`; neither the report nor that constant proves the current deployed backend. Cloud Run settings and historical timing figures in the report must not be assumed to describe this Render target. Current hosted searches were not invoked during submission preparation.

## 7. Project Structure

```text
GarmentChatbot/
├── README.md
├── frontend/                    Website, config.js and Node unit tests
├── backend/
│   ├── api.py                   FastAPI HTTP entry point
│   ├── chatbot.py               Language agent and command-line interface
│   ├── SKILL.md                 Application runtime prompt, required by agent
│   ├── .env.example             Nonsecret configuration template
│   ├── requirements*.txt        Runtime and API-test dependencies
│   ├── src/                     Embeddings, search, schemas, state and ingestion
│   ├── data/                    Product metadata and required attribute schema
│   ├── tests/                   Offline regression tests
│   └── other_tools/             Data preparation utilities
├── data/
│   ├── images/tshirt/           Original 1,993 JPEGs, paths preserved
│   ├── products.json            Root catalog metadata
│   ├── products.csv
│   └── samples/
├── evaluation/                  Saved rankings, labels, metrics and scripts
├── docs/                        Architecture, asset manifests and prior validation
├── models/                      Additional model helper
├── firebase.json               Static hosting configuration
└── package.json                Root Firebase dependency
```

There are 2,000 metadata entries and 1,993 available JPEGs (378,919,404 bytes, approximately 361.4 MiB); image IDs 357–363 are absent. The asset manifest documents the existing catalog. No replacement images, model caches or Qdrant snapshot are included. Older handover documents describe earlier integration stages; use the current source and these run instructions for this submission.

## 8. Installation and Setup

Commands below use a POSIX shell from the repository root. The recorded development baseline is Python 3.12 and Node 24; a fresh dependency install on every platform has not been validated.

```sh
git clone https://github.com/ivanho0908/GarmentChatbot.git
cd GarmentChatbot
python3.12 -m venv .venv
. .venv/bin/activate
python -m pip install -r backend/requirements.txt -r backend/requirements-api-test.txt
cp backend/.env.example backend/.env
```

Edit `backend/.env` privately:

- Set `POE_API`, `POE_URL` and `OPENAI_MODEL` for an approved Responses-compatible provider. Alternatively, remove/unset `POE_API` and `POE_URL` and configure `OPENAI_API_KEY` for SDK defaults. Empty strings are not equivalent to unset values.
- Supply `QDRANT_URL`, `QDRANT_COLLECTION` and, when required, `QDRANT_API` for an existing populated compatible collection. The saved study used 512-dimensional cosine vectors; confirm the embedding model, payload schema and image paths with the collection owner. A blank database alone cannot reproduce the search results.
- Set `CLIP_MODEL` to the exact model used for ingestion (saved-study identifier: `patrickjohncyh/fashion-clip`). Model weights must be available through approved downloads or a local cache. Optional `HF_TOKEN` is private and should be omitted when unnecessary.
- Set `CORS_ORIGINS=http://127.0.0.1:8000,http://localhost:8000` for the local commands below. A blank value overrides the default origin list and prevents browser access.

For local use, change only the public `API_BASE_URL` value in `frontend/config.js` to `http://127.0.0.1:8001`. Its committed value is a hosted URL. No frontend build or npm installation is required to serve the site. Never place credentials in frontend files.

Keep `backend/SKILL.md`, `backend/data/samples/sample_tshirt_schema.json` and the root catalog paths intact. Starting the API does not ingest or rebuild the collection. Offline ingestion writes Qdrant and is intentionally outside these run instructions. Private `.env` files, service keys and model caches are excluded from the submission.

## 9. How to Run the Chatbot

After setup, use two terminals:

```sh
# Terminal 1: repository root, virtual environment activated
cd backend
uvicorn api:app --reload --host 127.0.0.1 --port 8001
```

```sh
# Terminal 2: repository root
python3 -m http.server 8000 --bind 127.0.0.1 --directory frontend
```

Open [local frontend](http://127.0.0.1:8000/). The backend is at [local API](http://127.0.0.1:8001/), with interactive API documentation at `/docs`.

```sh
curl http://127.0.0.1:8001/api/health
```

Health reports liveness, not model, credential or Qdrant readiness. Real retrieval requires all external configuration above. Catalog photographs are served at `/images/tshirt/<id>.jpg`; missing images return 404. For the command-line agent, run `python chatbot.py` from `backend/` in the activated environment. Press **Ctrl+C** in each server terminal to stop it; run `deactivate` to leave the virtual environment.

## 10. Example Usage and Demonstration

1. Select Auto and submit “Show me 5 black T-shirts.” The backend plans the request, extracts supported attributes, embeds the query and retrieves eligible photographs. Saved case T1 returned IDs 1335, 679, 1205, 1265 and 1354; current rankings depend on external services and collection state.
2. Attach `data/images/tshirt/941.jpg` and enter “make it black.” Saved case M1 retained the reference vector with a mandatory black override and accepted plain/no-button attributes, returning IDs 1265, 890, 1618, 1114 and 1288. These are existing products, not recoloured versions of reference 941.
3. Refine the active image search with “make it long sleeve,” then request “show more.” Inspect returned cards and use carousel arrows to view the rest of the current response. Continuation preserves compatible effective requirements and excludes retained shown IDs.
4. Compare Exact and Similar to explain strictness; narrow constraints can underfill. “Japanese streetwear graphic T-shirts” illustrates semantic concepts that may exceed the supported attribute vocabulary.
5. On a supported browser, try voice input and response playback after checking permissions and hardware. This is a proposed demonstration step, not a claim of completed microphone acceptance.

## 11. Evaluation and Results

The saved study uses four text, three image and three image-plus-text primary queries, plus three pure-image baseline runs. One human reviewer judged products against query-specific visual criteria. Precision@5 uses a fixed denominator of five, including missing positions; T2's acknowledged zero-result ranking scores zero.

| Evidence | Saved result |
|---|---|
| Ten primary queries | Mean P@5 **0.6600**; 42 returned products |
| Five-result fill rate | **80%** (8 of 10 queries), availability rather than relevance |
| Text / image / image-plus-text means | 0.500 / 0.667 / 0.867; different, unmatched query groups |
| Paired pure-image v1, three references | Mean P@5 **0.7333** |
| Paired attribute-guided v2, same references | Mean P@5 **0.6667** |
| Saved rankings | 57 rows across 13 runs; 45 unique products |

Attribute guidance performed worse on average in this sample: two reference pairs tied, while the multi-garment reference fell from 0.200 to 0.000. This negative result is retained. The small purposive sample, single reviewer and unmatched modality groups do not establish statistical significance, general modality superiority or validation of every later mode/continuation path.

T2 returned no products for a compound white/graphic/short-sleeve request; its trace does not isolate the cause. T4 returned two products with one relevant. Reference 442 illustrates whole-image ambiguity: multicolor/color-block predicates excluded a relevant green/plain candidate. M3's plain modification admitted branded items whose payloads contain both plain and logo labels. Metadata compliance and visual relevance are different.

See [evaluation report](evaluation/EVALUATION_REPORT.md), [saved summary](evaluation/summary.csv) and [reproduction notes](evaluation/README.md). To recompute arithmetic without retrieval or provider calls:

```sh
python evaluation/calculate_metrics.py
```

This script writes the three metric CSVs from existing labels. Preserve the study inputs; recomputation is not a new retrieval experiment or human review.

For offline functional checks from the repository root:

```sh
node --test frontend/*.test.js
cd backend
CLIP_MODEL=patrickjohncyh/fashion-clip PYTHONPATH=.:tests python -B -m unittest test_chatbot_results test_api test_image_api test_image_text test_hybrid_image_search test_image_search test_extraction_robustness test_pagination test_image_context test_result_count test_unified_modes test_repository_layout
```

Submission verification on 17 November 2026 reran **103 frontend and 160 backend tests successfully**, using bundled Node and the existing local Python 3.12 development environment. Catalog hashes and real HTTP image serving passed; metric recomputation matched the saved CSV values. No fresh dependency installation, live retrieval, paid API call or microphone test was performed.

The original integration evidence records 103 frontend and 160 backend tests. The final report separately records 164 backend, 103 frontend and 8 script checks, plus two historical hosted acceptance rounds of 12 checks each; those later deployment artifacts are not all present in this checkout. These are distinct historical records, not fresh test results or 46 independent image tests. Tests mock provider/model calls and use fixtures or in-memory Qdrant. Real speech, production capacity and live relevance require separate acceptance.

## 12. Technical Challenges and Limitations

- **Semantic intent versus finite labels:** strict schemas preserve unsupported language for similarity, but text/image vocabularies differ (for example, graphic versus graphic print). An audited mapping and metadata review remain future work.
- **Visual ambiguity and noisy metadata:** zero-shot labels can reflect background garments or admit conflicting plain/logo descriptions. Foreground cropping, human label audits and controlled comparisons are proposed, not implemented improvements.
- **Strict retrieval:** exact constraints can exclude useful products or return shortages. Similarity scores are representational distances, not relevance probabilities; users must inspect results.
- **State across modalities:** validated reference embeddings and bounded shown-ID histories support continuity. Browser-local state can be cleared or modified and does not provide authenticated persistence or visual deduplication.
- **External reproducibility:** model weights, private LLM access and the populated Qdrant collection are external. No project-specific model training is claimed. Unpinned dependencies and missing collection/model revisions limit exact reproduction.
- **Deployment evidence:** the report records a 31.375-second first model-backed request and warm intervals of roughly 0.65–5.8 seconds on its historical deployment. These are observations, not a repeated cold-start distribution or load benchmark. Its one-instance/concurrency-one configuration limits throughput; current checkout deployment targets differ.
- **Privacy and security:** uploads are processed in memory, while conversations/reference state can remain in localStorage. CORS is not authentication, and SDK `store=False` does not prove provider-wide non-retention. Historical credential exposure identified by existing documentation still requires owner rotation/remediation; this submission does not rewrite history. Only a nonsecret configuration template is tracked in this revision.
- **Assets and speech:** seven catalog images are absent; dataset provenance and redistribution licensing remain unconfirmed in project evidence. Existing catalog paths are preserved. Voice availability and real microphone acceptance on the presentation device remain outstanding.

See [dataset notes](docs/DATASET_ASSETS.md), [integration evidence](docs/INTEGRATION_REPORT.md) and [historical hosting handover](docs/HANDOVER.md) for the original records and their boundaries.

## 13. Conclusion

GarmentChatbot combines structured language planning, pretrained fashion representations and filtered vector search into a conversational T-shirt discovery workflow. Its contribution is the integration of text, visual references, explicit modifications and continuing search with transparent retrieval controls. Saved evaluation demonstrates useful selected cases alongside failures; better metadata, broader human evaluation and confirmed deployment and voice acceptance remain the next steps.
