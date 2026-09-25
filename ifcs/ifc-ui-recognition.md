# Interface Contract: UI Server ↔ Recognition Server

**Jira:** TM-26
**Status:** v1 draft (initial implementation)
**Last updated:** 2026-09-25

---

## 1. Purpose

This document is the contract between the **UI side** (eScriptorium, acting as the caller) and the **recognition server** (a FastAPI service, running in its own Docker container, which talks to the model on the university server over SSH — see `recognition-communication-approach.md` for that internal leg).

It defines: which HTTP endpoints exist, what each one expects and returns, what state the recognition server keeps, and how errors are reported. It does **not** define how eScriptorium is wired to call these endpoints internally (plugin, custom view, etc.) — that is still to be worked out on the UI side.

## 2. Actors

- **Caller (UI):** eScriptorium. Exact integration point (plugin vs. custom code) is still open.
- **Recognition server:** a FastAPI application, containerized, reachable by the UI over HTTP. Internally it forwards work to a model loaded in an SSH session on the university server (UBB MLHub).

## 3. Scope and assumptions for v1

These are deliberate simplifications for the first version, not final decisions:

- **No authentication.** Endpoints are open. Auth will be added later once the service moves beyond an internal/dev setting.
- **Single user, single global state.** The server tracks exactly one loaded model and one "current image" at a time — there is no per-session or per-user isolation yet.
- **All calls are synchronous.** A request (e.g. `load model`) blocks until the operation finishes or fails. No polling/streaming/webhooks in v1, even though model loading over SSH can take a while.
- **No confidence scores.** `recognize` returns text only.
- **No image constraints enforced.** No format/size validation at the boundary yet.
- **Model list is not cached.** `GET /models` re-scans the models directory on every call.

## 4. Server-side state

The recognition server holds three pieces of state, all global (see assumptions above):

| State | Values | Set by | Cleared by |
|---|---|---|---|
| `loaded_model` | model name, or none | `POST /models/load` | `POST /models/unload`, or replaced by a new `load` |
| `current_image` | an image URL, or none | `POST /image` | never explicitly (only overwritten by a newer `POST /image`) |
| `busy` (recognizing) | true/false | set while a `recognize` call is running | cleared when it finishes |

These three pieces of state govern the error conditions listed per-endpoint below.

## 5. Conventions

- **Base path:** `/api/v1`
- **Content type:** `application/json` for all requests and responses, except where noted.
- **Error format:** FastAPI's default `HTTPException` shape:
  ```json
  { "detail": "Human-readable explanation of what went wrong." }
  ```
  Error messages are meant to be shown to the user as-is for now — there's no structured error code yet.
- **HTTP status codes used:** `200` success, `400` bad request (malformed input), `404` not found (unknown model), `409` conflict (operation not valid given current server state), `500` server-side failure (e.g. GPU OOM, SSH failure), `502` bad gateway (image fetch failed).

---

## 6. Endpoints

### 6.1 `GET /api/v1/models`

List the models currently available to load.

**Behavior:** scans the models directory on the recognition server on every call (no caching in v1).

**Request:** none.

**Response `200`:**
```json
{
  "models": ["model_base_hu_v1", "model_large_hu_v1"]
}
```
Model names carry meaningful info (e.g. base/large, language) by convention for now — there's no separate metadata field in v1.

---

### 6.2 `POST /api/v1/models/load`

Load a model onto the GPU on the university server.

**Request:**
```json
{ "model_name": "model_base_hu_v1" }
```

**Response `200`:**
```json
{ "status": "loaded", "model_name": "model_base_hu_v1" }
```

**Behavior:**
- If a different model is already loaded, it is unloaded first, then the requested one is loaded (replace, not stack — only one model in memory at a time in v1).
- If a `recognize` call is currently in progress (`busy == true`), the load request is **rejected** rather than queued.

**Errors:**
- `404` — `model_name` not found among available models.
- `409` — a recognition is currently in progress.
- `500` — load failed on the server side (e.g. GPU OOM, SSH connection dropped mid-load). `detail` carries the underlying message.

---

### 6.3 `POST /api/v1/image`

Tell the recognition server which image is currently active, so later `recognize` calls know what to operate on. Sent whenever the UI loads/changes the image.

**Request:**
```json
{ "image_url": "https://.../iiif/.../full/full/0/default.jpg" }
```
The recognition server fetches the image itself from this URL (IIIF-based, per eScriptorium) rather than receiving raw bytes — avoids shipping image data twice (UI → recognition server → university server).

**Response `200`:**
```json
{ "status": "ok" }
```

**Behavior:** overwrites `current_image`. No image identifier/versioning yet — "last sent image" is simply the current one, referenced implicitly (not by ID) in `recognize` responses too.

**Errors:**
- `400` — `image_url` missing or malformed.
- `502` — the recognition server could not fetch the image from the given URL.

---

### 6.4 `POST /api/v1/recognize`

Run recognition on one or more regions of the current image.

**Request:**
```json
{
  "regions": [
    { "id": "r1", "points": [[120, 340], [180, 342], [181, 365], [119, 363]] },
    { "id": "r2", "points": [[190, 340], [260, 344], [259, 366], [188, 362]] }
  ]
}
```
- `points` is a simple ordered list of `[x, y]` pixel coordinates on the image as sent via `POST /image` (not some normalized/IIIF coordinate space). A single line/polygon per region — matches what will come from the PageXML source; exact point count/closure convention (e.g. whether first/last point repeat) still to be confirmed once you see real PageXML data.
- `id` is caller-supplied, used only to match a result back to its request — the recognition server does not interpret it.
- Works for one region or many in a single call (a list of length 1 or more).

**Response `200`:**
```json
{
  "results": [
    { "id": "r1", "text": "recognized" },
    { "id": "r2", "text": "word" }
  ]
}
```
Response order is not guaranteed to match request order — always match on `id`.

**Errors:**
- `409` — no model is loaded (`loaded_model` is none).
- `409` — no image has been sent yet (`current_image` is none).
- `400` — `regions` missing/empty, or a region has fewer than 2 points.

---

### 6.5 `POST /api/v1/models/unload`

Remove the currently loaded model from GPU memory.

**Request:** none.

**Response `200`:**
```json
{ "status": "unloaded" }
```

**Behavior:** idempotent / no-op safe — returns `200` with the same response even if no model was loaded. Intended to be called from a "remove model" UI button, and later also right before the connection/session ends.

**Errors:** none expected in v1 (always succeeds).

---

## 7. Open questions / deliberately deferred

These are known gaps, tracked for follow-up rather than blocking v1:

- **Auth** — none yet; to be added before this leaves an internal/dev setting.
- **Concurrency / multi-user** — v1 assumes exactly one caller. Needs real design once more than one user/session exists.
- **Async model loading** — currently blocks the HTTP call for the full SSH load time. May need a "started + poll status" pattern if this becomes a UX problem.
- **Image identity** — no explicit image ID; "last sent" is assumed current. Will need an ID once multiple images can be in flight.
- **PageXML coordinate format** — the exact shape/convention of `points` coming out of PageXML is not yet confirmed; `[[x,y], ...]` here is a best guess pending real data.
- **Confidence scores / alternates** — not returned yet; likely a future addition to the `recognize` response.
- **Model metadata** — `GET /models` returns names only; structured metadata (base/large, language, "currently loaded") may be added later.
- **Exact eScriptorium integration point** — how eScriptorium actually calls these endpoints (plugin, custom panel, etc.) is not yet decided.

## 8. Related Jira

- TM-24 — connection possibilities for the university server
- **TM-26 — interface contract between the UI server and recognition server (this document)**
- TM-29 — initial version of the recognition server
