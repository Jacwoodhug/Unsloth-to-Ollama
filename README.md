# Unsloth-to-Ollama Proxy

A FastAPI proxy server that translates Ollama API requests to Unsloth's OpenAI-compatible API.

## Features

- **Ollama-compatible API** on port `11434`
- **Unsloth backend** integration via OpenAI-compatible endpoints
- **Streaming support** for real-time responses
- **Environment-based configuration**

## Requirements

- Python 3.10+
- `uv` (Python package manager)

## Installation

1. Clone this repository
2. Run `run.bat` (Windows) or manually:
   ```bash
   uv venv .venv
   uv pip install -r requirements.txt
   ```

## Configuration

Create a `.env` file in the project root:

```env
UNSLOTH_BASE_URL=http://localhost:8888
UNSLOTH_API_KEY=your_api_key_here
PROXY_HOST=0.0.0.0
PROXY_PORT=11434
MODEL_CONTEXT_LENGTH=65536```

## Usage

Run `run.bat` to start the proxy server. The server will be available at `http://localhost:11434`.

## API Coverage

The proxy implements Ollama-compatible endpoints including:
- `/v1/chat/completions`
- `/v1/models`
- `/health`

See `PLAN.md` for full endpoint mapping.

## Hard-coded Values

The following values are hard-coded and not configurable via environment variables:

- **Ollama version** (`/api/version`): `0.24.0`
- **Model capabilities**: `["completion", "tools"]`

If your Ollama client checks for a specific version or capability set, you may need to update these values in `main.py` and `translate.py` respectively.

## Important: Context Length Configuration

### Why Context Length Matters

When using this proxy with **VS Code Copilot**, the context length reported by the proxy determines how much context Copilot will use before triggering "Conversation Compact". If the context length is too low, Copilot will compact conversations prematurely, reducing effective context.

### Setting Context Length in `.env`

Set `MODEL_CONTEXT_LENGTH` in your `.env` file to match your model's actual context window:

```env
MODEL_CONTEXT_LENGTH=65536
```

### Loading the Model in Unsloth

**Crucially**, you must load your model in Unsloth with the same context length you set in `.env`. Unsloth's OpenAI-compatible API returns the context length from the loaded model, and the proxy uses that value. If you don't load the model with the correct context length in Unsloth, the proxy will report the wrong context length to Copilot.

For example, if you want 65536 context but load the model in Unsloth with a smaller context, the proxy will report that smaller value to Copilot, ignoring your `.env` setting.

### Architecture-Specific Context Length

The proxy automatically adds architecture-specific context length keys (e.g., `qwen35.context_length`) to the `/api/show` response. This ensures VS Code Copilot correctly reads the context length instead of falling back to the default 32768.

If you're using a custom model architecture, ensure `_guess_details()` in `translate.py` correctly identifies it so the appropriate context length key is generated.

## License

MIT