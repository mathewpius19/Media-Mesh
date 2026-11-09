# Teammate 3 — REEL-FINDER Reuse, CLIP Embeddings, Video Indexing and Search

**Mission:** Adapt the existing REEL-FINDER semantic-retrieval service for NatGeo wildlife photos and short footage. Own Python ML, model inference, pgvector search, vector schema and the indexing worker. Do **not** implement SSO, Spring public APIs, S3 upload URLs or React UI.

## You own
- Entire `ml-service/` directory, tests and Python dependencies.
- SQL migrations `V3xx__...sql`: enable pgvector; `asset_vectors(asset_id,frame_timestamp_seconds,model_version,embedding)` and indexes.
- Internal search/embedding APIs and worker code; matching/ranking logic.
- Lightweight evaluation dataset and expected search examples.

## REEL-FINDER reuse plan
- **Reuse:** Python Flask service structure, Sentence Transformers model-loading pattern, text query embedding, nearest-neighbor retrieval, ranking and service integration patterns.
- **Change:** movie title/genre/rating data -> NatGeo `assetId`, photo/video frame embeddings; use compatible **CLIP text + image encoder**, not the old MiniLM movie vectors.
- **Simplify:** pgvector as durable index; retain FAISS only for an isolated proof if useful. Similar assets = embedding similarity.
- **Defer:** Ollama agent, MCP, personalized watch history, native audio embeddings, temporal video models.

## Tasks in implementation order

- [ ] **T3.1 — Prove CLIP retrieval locally.** Select one text/image CLIP model. Embed 20–30 sample wildlife images and retrieve e.g. “elephants at a watering hole” from text. Record model name, version and dimensionality.
- [ ] **T3.2 — Vector schema.** `asset_vectors` keyed by `asset_id` + nullable `frame_timestamp_seconds`, `model_version`, vector field with matching dimension; FK references teammate 2's `assets`. Avoid migrations touching `assets` itself.
- [ ] **T3.3 — Image indexing.** Given a local file or S3-readable asset reference, create one CLIP embedding and upsert idempotently into pgvector. Verify deleted/reindexed media does not generate duplicates.
- [ ] **T3.4 — Video indexing.** Use FFmpeg to sample a *small configurable number* of frames from short videos, embed frames separately and store timestamps. No full video understanding or transcription.
- [ ] **T3.5 — Python worker.** Poll/claim teammate 2's `index_jobs` table (or use a stable SQL adapter), fetch private S3 image/video bytes using server-side credentials, index, and write `READY`; on error mark retryable failure with attempts/lease behavior. **Worker processing belongs to you, job schema/lifecycle API to teammate 2.**
- [ ] **T3.6 — Internal similarity API.** `POST /internal/search` takes text and limit, returns ranked `{assetId, score, frameTimestampSeconds}`. Filter READY assets or have backend enforce READY; never serve direct public URLs.
- [ ] **T3.7 — Similar-assets query.** Optionally expose `POST /internal/similar` for the same embedding index if it fits schedule; this reuses similarity ranking, not personalization. Treat as the lowest-priority required search extension.
- [ ] **T3.8 — Tests and quality.** Fixed test images, deterministic embedding dimensionality, index upsert, empty-index search, video timestamps, failure/retry handling, 5 expected semantic query matches. Record inference startup latency and model download instructions.

## Contract you publish at Checkpoint A

```http
POST /internal/search
Content-Type: application/json
```

```json
{"query":"elephants drinking at a watering hole at sunset","limit":20}
```

```json
{"matches":[{"assetId":"uuid","score":0.83,"frameTimestampSeconds":72},{"assetId":"uuid2","score":0.79,"frameTimestampSeconds":null}]}
```

`frameTimestampSeconds = null` for photos. Return no S3 URLs or personal user data. Spring (teammate 2) applies role checks and validated metadata filters. Internal service should be reachable only over private Docker network or authenticated service-to-service connection, never public internet.

**Indexing integration:** Teammate 2 creates an `index_jobs` row for an `assets.id`; your worker claims and processes it. Agree exact DB column names and transaction/lease semantics at Checkpoint A. If embedding fails, the asset remains `FAILED`/retryable; don't silently mark READY.

## Dependencies / non-overlap
- Obtain sample S3 media and asset IDs from **2**, but start with local fixtures; you do not edit 2's storage code.
- Agree vector dimension before creating migration; **2** owns asset metadata, you own embeddings only.
- **4** uses Spring API mock results; never call internal Python endpoint from React.
- **1** owns user roles and tokens. ML trusts only internal backend/worker traffic, not user-supplied role strings.

## Independent demo
1. Index small wildlife photo folder into pgvector.
2. Search “elephants drinking at sunset”; show relevant ranked photos.
3. Index a short clip, return matched frame timestamp.
4. Re-run same job: no duplicate vectors; intentionally fail one job and retry.

## Done when
- [ ] Image/text embeddings use one compatible model and are persisted in pgvector.
- [ ] Text searches return valid asset UUIDs and timestamps.
- [ ] Python worker can index uploaded S3 assets using teammate 2's job contract.
- [ ] ML service and worker are containerizable and documented with repeatable test fixtures.
