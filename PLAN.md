# Ollama → Unsloth API Proxy — Implementation Plan

## Goal

A lightweight Python proxy that listens on Ollama's port (11434), accepts standard Ollama API requests, translates them to Unsloth's OpenAI-compatible API, and converts responses back to Ollama format — including full streaming support. This lets any Ollama-compatible client (Open WebUI, Continue, etc.) work with a Unsloth backend transparently.

---

## File Structure

```
unsloth-to-ollama/
├── main.py           # FastAPI app + all route handlers
├── translate.py      # Pure translation functions (request/response conversion)
├── requirements.txt  # fastapi, httpx, uvicorn[standard], python-dotenv
└── .env.example      # Config template
```

---

## Configuration

Via `.env` file:

```
UNSLOTH_BASE_URL=http://localhost:8000
UNSLOTH_API_KEY=sk-unsloth-your-key-here
PROXY_HOST=0.0.0.0
PROXY_PORT=11434
```

---

## Endpoint Coverage

| Ollama Endpoint | Method | Mapped To | Notes |
|---|---|---|---|
| `/api/chat` | POST | `/v1/chat/completions` | Full translation + streaming |
| `/api/generate` | POST | `/v1/chat/completions` | Convert prompt → messages array |
| `/api/tags` | GET | `/v1/models` | Convert model list format |
| `/api/ps` | GET | `/v1/models` | Shape model list as "running models" |
| `/api/show` | POST | `/v1/models` | Return matching model info |
| `/v1/chat/completions` | POST | `/v1/chat/completions` | Pass-through |
| `/v1/models` | GET | `/v1/models` | Pass-through |
| `/v1/completions` | POST | `/v1/chat/completions` | Convert prompt → messages |
| `/api/version` | GET | — | Return static fake version string |
| `/api/pull` | POST | — | Streaming success stub |
| `/api/delete` | DELETE | — | 200 stub |
| `/api/copy`, `/api/push`, `/api/create` | POST | — | 501 stub |
| `/api/embed`, `/v1/embeddings` | POST | — | 501 (not supported by Unsloth) |

---

## Key Translation Logic (`translate.py`)

### `/api/chat` → OpenAI `/v1/chat/completions` (request)

| Ollama field | OpenAI field | Notes |
|---|---|---|
| `model` | `model` | direct |
| `messages` | `messages` | direct; image content parts stay compatible |
| `stream` | `stream` | direct |
| `tools` | `tools` | Ollama args are dicts → serialize to JSON string |
| `format: "json"` | `response_format` | `{"type": "json_object"}` |
| `format: <schema>` | `response_format` | `{"type": "json_schema", "json_schema": ...}` |
| `options.temperature` | `temperature` | |
| `options.top_p` | `top_p` | |
| `options.seed` | `seed` | |
| `options.stop` | `stop` | |
| `options.num_predict` | `max_tokens` | |
| `options.top_k` | — | dropped (not in OpenAI spec) |
| `think` | — | dropped (Unsloth handles internally) |

### `/api/generate` → OpenAI `/v1/chat/completions` (request)

Build messages array:
1. If `system` present: `[{"role": "system", "content": system}]`
2. User message: `{"role": "user", "content": prompt}` with any `images` embedded as base64 `image_url` content parts

Same `options` mapping as above.

### OpenAI response → Ollama `/api/chat` response (non-streaming)

```
choices[0].message.content → message.content
choices[0].finish_reason    → done_reason
usage.prompt_tokens         → prompt_eval_count
usage.completion_tokens     → eval_count
created (unix timestamp)    → created_at (ISO 8601)
done: true
total_duration / load_duration / eval_duration: 0
```

### OpenAI response → Ollama `/api/generate` response (non-streaming)

Same as above but `message.content` → `response` (flat string field).

### Streaming: SSE → NDJSON

Unsloth streams SSE (`data: {...}\n\n`). Ollama clients expect NDJSON (`{...}\n`).

Per SSE chunk:
1. Strip `data: ` prefix; skip `[DONE]`
2. Parse JSON, extract `choices[0].delta.content`
3. Emit Ollama NDJSON with `done: false`
4. When `finish_reason` is set: emit final line with `done: true`, `done_reason`, token counts

### Tool calls (streaming)

Accumulate `delta.tool_calls` across chunks, then on `finish_reason`:
- Emit assembled `message.tool_calls` in the final Ollama chunk
- Convert OpenAI format (`arguments` as JSON string) → Ollama format (`arguments` as dict)

---

## `main.py` Route Structure

```python
app = FastAPI()
# httpx.AsyncClient pointed at UNSLOTH_BASE_URL with Authorization header

@app.post("/api/chat")           # translate → /v1/chat/completions, SSE→NDJSON
@app.post("/api/generate")       # translate → /v1/chat/completions, SSE→NDJSON
@app.get("/api/tags")            # → /v1/models, convert response shape
@app.get("/api/ps")              # → /v1/models, fake running-model shape
@app.post("/api/show")           # → /v1/models, return matching model
@app.get("/api/version")         # static stub
@app.post("/api/pull")           # streaming success stub
@app.delete("/api/delete")       # 200 stub
# stubs for /api/copy, /api/push, /api/create

# OpenAI-compat pass-throughs
@app.post("/v1/chat/completions") # forward directly (stream aware)
@app.get("/v1/models")            # forward directly
@app.post("/v1/completions")      # convert prompt→messages, forward
@app.post("/v1/embeddings")       # 501 Not Implemented
@app.post("/api/embed")           # 501 Not Implemented
```

Streaming routes return `StreamingResponse` that iterates the httpx async stream, converting each SSE chunk to NDJSON on the fly.

---

## Running

```bash
pip install -r requirements.txt
cp .env.example .env
# Edit .env with your Unsloth URL and API key
python main.py
```

Then point any Ollama-compatible client at `http://localhost:11434`.

---

## Verification Steps

1. `curl http://localhost:11434/api/version` → `{"version":"0.0.1-unsloth-proxy"}`
2. `curl http://localhost:11434/api/tags` → model list from Unsloth
3. Non-streaming chat:
   ```bash
   curl http://localhost:11434/api/chat -d '{"model":"<model>","messages":[{"role":"user","content":"hi"}],"stream":false}'
   ```
4. Streaming chat (default):
   ```bash
   curl http://localhost:11434/api/chat -d '{"model":"<model>","messages":[{"role":"user","content":"hi"}]}'
   ```
5. Generate:
   ```bash
   curl http://localhost:11434/api/generate -d '{"model":"<model>","prompt":"Why is the sky blue?","stream":false}'
   ```
6. Point Open WebUI (configured for `http://localhost:11434`) at the proxy and verify full UI flow
